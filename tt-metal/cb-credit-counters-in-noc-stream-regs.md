# CB credit counters do NOT live in L1 — they are NOC-overlay stream registers

**Finding.** A circular buffer's two flow-control counters, `tiles_received` (`pages_received`)
and `tiles_acked` (`pages_acked`), are **not** fields of the CB's L1 control block and **not**
in `LocalCBInterface`. They are stored in the **NOC overlay ("stream") functional register
file** — "don't-care functional registers" that the overlay hardware would own if the NOC
overlay were enabled. Dataflow-RISC `cb_reserve_back` / `cb_wait_front` read
them with `reg_read()`, and its `cb_push_back` / `cb_pop_front` update them
with RISC-V stores. Compute-RISC updates can instead use Tensix `STOREREG`.

Observed on tt-metal `4f9fa9e0` (`v0.80.0-dev20260922~53`), arch **blackhole** (p150a).
**Read from source, not measured on device** — see "What is assumed" at the bottom.

## Where the address comes from

`tt_metal/hw/inc/internal/tt-1xx/blackhole/stream_io_map.h` (identical shape in
`tt-1xx/wormhole/` and `tt-2xx/quasar/`):

```c
const uint32_t OPERAND_START_STREAM = 0;                 // was 8 on old tt-metal
inline uint32_t get_operand_stream_id(int operand) { return OPERAND_START_STREAM + operand; }

inline volatile uint32_t* get_cb_tiles_received_ptr(int operand) {
    return (volatile uint32_t*)(uintptr_t)(STREAM_REG_ADDR(
        get_operand_stream_id(operand), STREAM_REMOTE_DEST_BUF_SIZE_REG_INDEX));   // reg 10
}
inline volatile uint32_t* get_cb_tiles_acked_ptr(int operand) {
    return (volatile uint32_t*)(uintptr_t)(STREAM_REG_ADDR(
        get_operand_stream_id(operand), STREAM_REMOTE_DEST_BUF_START_REG_INDEX));  // reg 8
}
```

and `tt_metal/hw/inc/internal/tt-1xx/blackhole/noc/noc_overlay_parameters.h:43-46`:

```c
#define NOC_OVERLAY_START_ADDR 0xFFB40000
#define NOC_STREAM_REG_SPACE_SIZE 0x1000
#define STREAM_REG_ADDR(stream_id, reg_id) \
    (NOC_OVERLAY_START_ADDR + (stream_id * NOC_STREAM_REG_SPACE_SIZE) + (reg_id << 2))
```

So CB *n*'s counters are at `0xFFB40000 + n*0x1000 + {0x20 (received), 0x28 (acked)}` on
Blackhole. `get_operand_stream_id` is the identity, so **operand id == stream id == CB id**.

Corroboration that these are real hardware registers and not L1:

- The HAL knows the same constants (`bh_hal.cpp:478-479` →
  `noc_stream_remote_dest_buf_size_reg_index_` / `..._start_reg_index_`), and its
  `valid_reg_addr_func_` (`bh_hal.cpp:415-416`) explicitly whitelists the
  `[0xFFB40000, +0x1000*NOC_NUM_STREAMS)` range as **register** addresses.
- The watcher reads the counters back over the register path, not the L1 path
  (`watcher_device_reader.cpp:1161-1185`, `DumpSyncRegs`), and prints
  `cb[%u](rcv %u!=ack %u)` when they disagree.

## The five functions

`tt_metal/hw/inc/api/dataflow/dataflow_api.h` (dataflow RISCVs — BRISC / NCRISC / ERISC):

```c
FORCE_INLINE void cb_reserve_back(int32_t operand, int32_t num_pages) {
    uintptr_t pages_acked_ptr = (uintptr_t)get_cb_tiles_acked_ptr(operand);
    uint32_t pages_received = get_cb_tiles_received_ptr(operand)[0];   // snapshot, once
    int32_t free_space_pages;
    WAYPOINT("CRBW");
    do {
        invalidate_l1_cache();
        uint16_t pages_acked = (uint16_t)reg_read(pages_acked_ptr);
        uint16_t free_space_pages_wrap =
            get_local_cb_interface(operand).fifo_num_pages - (pages_received - pages_acked);
        free_space_pages = (int32_t)free_space_pages_wrap;
    } while (free_space_pages < num_pages);
    WAYPOINT("CRBD");
}

void cb_push_back(const int32_t operand, const int32_t num_pages) {
    get_cb_tiles_received_ptr(operand)[0] += num_pages;   // credit
    get_local_cb_interface(operand).fifo_wr_ptr += num_pages * fifo_page_size;  // + wrap
}

void cb_wait_front(int32_t operand, int32_t num_pages) {
    uint32_t pages_acked = get_cb_tiles_acked_ptr(operand)[0];          // snapshot, once
    uintptr_t pages_received_ptr = (uintptr_t)get_cb_tiles_received_ptr(operand);
    uint16_t pages_received;
    WAYPOINT("CWFW");
    do { pages_received = ((uint16_t)reg_read(pages_received_ptr)) - pages_acked; }
    while (pages_received < num_pages);
    WAYPOINT("CWFD");
}

void cb_pop_front(int32_t operand, int32_t num_pages) {
    get_cb_tiles_acked_ptr(operand)[0] += num_pages;      // credit
    get_local_cb_interface(operand).fifo_rd_ptr += num_pages * fifo_page_size;  // + wrap
}
```

`cb_reserve_back` is a **pure poll**: it advances no pointer and takes no lock. `fifo_wr_ptr`
is only advanced by `cb_push_back`. The producer reserves, writes the pages into L1 itself
(with plain stores or NOC reads), and only then bumps the credit.

Compute RISCVs (TRISC) go through `tt_metal/hw/inc/api/compute/cb_api.h`, which maps
`cb_reserve_back(cbid, ntiles)` → `PACK((llk_wait_for_free_tiles(cbid, ntiles)))`
(`llk_io_pack.h:20-42`) and `cb_wait_front` → `UNPACK((llk_wait_tiles(...)))`
(`llk_io_unpack.h:18-30`). The generated wait loops poll the overlay counters
with RISC-V `lw` (for example, `mm_trisc0_unpack.S:257-263` and
`mm_trisc2_pack.S:262-265`). Some counter updates use Tensix `STOREREG` after
`STALLWAIT` (`llk_io_unpack.h:44-47`, `llk_io_pack.h:60-67`), while other
updates use RISC-V stores.

## Four details that are easy to get wrong

1. **`pages_received` is snapshotted once, before the loop; `pages_acked` is re-read every
   iteration.** That is deliberate: while a producer is spinning in `cb_reserve_back` it is
   not pushing, so `tiles_received` cannot change *for this RISC*. Only the consumer moves.
   (The comment says exactly this.) Do not "fix" it by hoisting the `acked` read out.

2. **Everything is 16-bit wrapping arithmetic and that is load-bearing.** The comparison
   `fifo_num_pages - (received - acked)` is computed in `uint16_t`, so it is correct across
   counter wraparound. The host enforces the other half of this contract:
   `circular_buffer.cpp:19-20` caps `num_pages <= 2^16 - 1` (`max_num_cb_pages`) with a
   `TT_FATAL`, because a CB with more pages than the counter can express would wrap wrongly.
   Note the two sides differ in width — `pages_received` is read as a full `uint32_t` on the
   dataflow side while `pages_acked` is masked to `uint16_t` — which is harmless only because
   the result is truncated to `uint16_t` anyway.

3. **`invalidate_l1_cache()` is a Blackhole-only fence.** `internal/tt-1xx/cache.h` is the
   whole definition:

   ```c
   inline __attribute__((always_inline)) void invalidate_l1_cache() {
   #if defined(ARCH_BLACKHOLE)
       asm("fence");
   #endif
   }
   ```

   On Wormhole it compiles to nothing. The counter is written by *another* RISCV (or by a NOC
   atomic for remote CBs), so the stated rationale is that a stale cached read would mean
   either spinning forever or proceeding early.

   **And the rationale is empirically confirmed — do not remove this fence.** The L0 data
   cache *does* serve the NoC-overlay register region, even though the ISA pipeline diagram
   draws that region as a sibling of the cache rather than behind it. Proof is in the
   tt-metal history: `7e6553092db` (2025-05-06, *"Add missing l1 cache invalidation to
   resolve Whisper hang"*) re-added this exact one line after a "perf cleanup" had stripped
   it, because *"Whisper was hanging on P100a due to missing cache invalidation"*.

   The full commit chain, and why the earlier "the fence is defensive" reading in this note
   was wrong: see [`../isa/l0-data-cache-and-fence.md`](../isa/l0-data-cache-and-fence.md).

   What the `fence` actually flushes is the per-RISCV 64-byte **L0 data cache**, not an "L1
   cache" — L1 is a 1536 KiB scratchpad RAM, not a hardware cache, so the source comment's
   wording is a misnomer. Full semantics, and the open question of whether the
   `0xFFB4_0000` register region is even covered by that cache: see
   [`../isa/l0-data-cache-and-fence.md`](../isa/l0-data-cache-and-fence.md).

4. **`WAYPOINT("CRBW"/"CRBD")` names the spin.** With `TT_METAL_WATCHER=1`, a kernel wedged in
   `cb_reserve_back` reports `CRBW` as its last waypoint, and a kernel wedged in
   `cb_wait_front` reports `CWFW`. This is the fastest way to tell "producer waiting for
   space" from "consumer waiting for data" from the watcher dump alone. (4 chars max — the
   waypoint is packed into one `uint32_t`; `api/debug/waypoint.h:24-25`.)

## What the generated assembly shows

Verified by compiling the kernel with tt-metal's own JIT flag set. Harness and
annotated output: `test/cb_reserve_asm/` (probe) and
`example/standalone/matmul_multi_core/asm_kernel/` (real kernels).

**The counters are demonstrably not L1.** `cb_reserve_back` compiles to a plain
`lw` from a constant high address:

```asm
	li	a1,-4980736          # 0xFFB40000 = CB 0's stream-register base
	lw	a2,40(a1)            # pages_received   (reg 10)
	lw	a3,32(a1)            # pages_acked      (reg 8)
	lw	a5,12(a4)            # cb_interface[0].fifo_num_pages  <-- this one is L1
```

The contrast is the proof: `fifo_num_pages` is reached through the `cb_interface`
`%hi/%lo` pair (an L1 symbol), while both counters are absolute constants. For CB
16 the immediate is `-4915200` = `0xFFB50000`.

**The loop is 7-8 instructions, one of which is the fence:**

```asm
.L33:
	fence                        # invalidate_l1_cache()
	lw	a2,32(a1)            # pages_acked, re-read every iteration
	lw	a5,12(a4)            # fifo_num_pages
	add	a5,a5,a2
	sub	a5,a5,a0            # a0 = (uint16_t)pages_received, hoisted out of the loop
	zext.h	a5,a5               # this IS the (uint16_t) cast in the source
	bgt	a3,a5,.L33           # while (num_pages > free_space)
```

The `zext.h` after the add/sub is what makes the comparison wrap-safe; the `add`
(instead of the source's `fifo_num_pages - (received - acked)`) is just GCC
picking the equivalent form mod 2^16. And **`num_pages` changes the emitted
compare**: with the common `n == 1`, `free < 1` folds to `free == 0` and then to
`n + acked == received`, so the `sub` disappears entirely — the hot loop is one
instruction shorter for `n == 1` than for `n == 4`.

**⚠️ Only `cb_reserve_back` fences.** The fence is not a property of CB sync in
general — it is specific to the producer's free-space wait. Confirmed in the
source and then in the emitted code. The asymmetry is real and the reserve-side flush is
**load-bearing** — it was restored by tt-metal `7e6553092db` to fix a Whisper hang after a
cleanup had removed it, so `cb_reserve_back` is not the odd one out by accident. Why the other
three are safe without it is still unexplained: see
[`../isa/l0-data-cache-and-fence.md`](../isa/l0-data-cache-and-fence.md).

| function | `invalidate_l1_cache()` per iteration |
| --- | --- |
| `cb_reserve_back` (`dataflow_api.h`) | **yes** |
| `cb_wait_front` (`dataflow_api.h`) | no |
| `llk_wait_for_free_tiles` (`llk_io_pack.h:20-42`) | no |
| `llk_wait_tiles` (`llk_io_unpack.h:18-30`) | no |

The `fence` counts in the generated kernels balance exactly, which is a good
cross-check that the loops were identified correctly: reader = 4 (2 reserves + 2
`noc_async_read_barrier`), writer = 1 (1 `noc_async_write_barrier`, 0 from
`cb_wait_front`), all three TRISC kernels = 0.

**The shadow-vs-register split is visible too.** In `mm_trisc2_pack.S` the
compute-side reserve reads the two halves from two different memories:

```asm
	lhu	a4,538(a7)           # cb_interface[16].tiles_received  <-- L1 SHADOW (offset 26)
	lw	a5,32(a1)            # tiles_acked                      <-- STREAM REGISTER
	add	a5,a0,a5
	zext.h	a5,a5
	beq	a4,a5,.L71
```

`lhu` off `cb_interface` for `received`, `lw` off `0xFFB50000` for `acked`, and no
fence. The reader's equivalent instead reads *both* from the registers
(`40(t2)` and `32(t2)`), because the dataflow path has no packer to race with.

Finally, `cb_reserve_back` has **no symbol** in a linked kernel — it is
`FORCE_INLINE`, so it only ever appears as a bare loop inside `kernel_main`.


## Who zeroes the counters, and when

This matters because a stale counter from a previous program would corrupt the free-space
math. The reset is **device-side and per-kernel-run**, not host-side:

- `tt_metal/hw/firmware/src/tt-1xx/trisc.cc:97-105` — `init_sync_registers()` loops
  `operand = 0 .. NUM_CIRCULAR_BUFFERS` and stores 0 to **both** counters. It runs on
  **TRISC0** only.
- `tt-1xx/trisc.cc:137-139` — TRISC0 calls it when its mailbox reads
  `RUN_SYNC_MSG_INIT_SYNC_REGISTERS`.
- `tt-1xx/brisc.cc:339` — `trigger_sync_register_init()` writes that message to
  `subordinate_sync->trisc0`. BRISC calls it **twice**: once at firmware bring-up after
  `noc_init` (`:391`), and once at the **end of every kernel run**, after
  `wait_ncrisc_trisc()` (`:555`).

So the registers are clean *before* the next launch because they are cleared *after* the
previous one. There is no host-side write of the CB counters anywhere in `tt_metal/impl/`
(the only `noc_stream_remote_dest_buf*` users are the dispatch stream, the watcher, and
`jit_device_config.cpp`) — worth knowing if you are chasing a counter that looks
wrong at kernel entry.

Also note the `LocalCBInterface` union still *has* `uint16_t tiles_acked` /
`uint16_t tiles_received` fields (`circular_buffer_interface.h:103-106`), zeroed by
`setup_local_cb_read_write_interfaces` (`circular_buffer_init.h:61`). Those are a
**TRISC-private shadow**, not the shared counter. `llk_wait_for_free_tiles` reads
`tiles_received` from the shadow on purpose — reading the register back would race with the
packer's own `TT_STOREREG` — while `tiles_acked` it reads from the register. Two different
sources of truth for the two halves of the same subtraction, and the comments at
`llk_io_pack.h:32-33` and `:51-57` explain why.

## Why this is worth remembering

- The skill doc `tt_metal/tt-llk/.claude/skills/dataflow-cb-sync-audit/SKILL.md` describes the
  counters as "two L1 counters per CB". That framing is **wrong for this version** — they are
  registers. Any reasoning that treats `pages_acked` as L1 (e.g. "a NOC write to that address
  will land", or "the watcher can read it as memory") has to be re-derived.
- The registers are **scratch use of hardware that is otherwise idle**. If the NOC overlay is
  ever enabled on a core, CB sync and the overlay would fight over the same words.
- The credit counter is only 16 bits wide, so a CB cannot have more than 65535 pages — this is
  a hard host-side `TT_FATAL`, not a soft limit.

## What is assumed, not measured

Everything above is read from the `4f9fa9e0` source. **Not** verified on device in this note:
that the register file really behaves as plain read/write storage at those addresses, that
the BH `fence` is what makes the cross-RISC read coherent, and that no other agent writes
`0xFFB40000 + n*0x1000` while a kernel is running. The `DumpSyncRegs` watcher path
(`watcher_device_reader.cpp:1161-1185`) is the cheap way to check the first one — it reads
both registers back over the host register path and prints them when they disagree.
