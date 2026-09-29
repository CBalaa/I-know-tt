# Timing boundaries in `matmul_multi_core` assembly

**Finding (source-read, not device-measured).** For the Blackhole p150a assembly
generated with tt-metal `4f9fa9e0`, instruction syntax alone does not identify
the timing event. Classify each executed instruction by its address or decoded
Tensix effect, then track issue, acceptance, completion, and observation
separately. The five `.S` files under
`example/standalone/matmul_multi_core/asm_kernel/` contain only `kernel_main`;
linked firmware and called helpers are outside them.

## Cases present in this kernel

| Assembly site | Timing effect |
| --- | --- |
| `reader_mm_output_tiles_partitioned_ncrisc.S:67-72` | `divu`/`remu` use runtime operands, so the executed path and divider cost matter. |
| `reader_mm_output_tiles_partitioned_ncrisc.S:112-159`, `writer_unary_interleaved_start_id_brisc.S:81-137` | RISC loads/stores access NoC command/status MMIO. Command-register store, NIU acceptance, response/ACK, and the next successful RISC poll are different events. |
| `mm_trisc0_unpack.S:257-263`, `mm_trisc2_pack.S:261-265` | RISC `lw` polls CB credit in NoC-overlay stream registers. Related updates may be RISC stores or Tensix `STOREREG`; see [`../tt-metal/cb-credit-counters-in-noc-stream-regs.md`](../tt-metal/cb-credit-counters-in-noc-stream-regs.md). |
| `mm_trisc1_math.S:19-24,178-185`, `mm_trisc0_unpack.S:190-198`, `mm_trisc2_pack.S:126-135,177-185` | `sw; lw; and` at `PC_BUF_BASE+4/+8` uses Manual TTSync. The load response can wait inside the memory system; it is not a RISC polling loop. `+4` waits for this Tensix thread's in-flight work; `+8` waits only for MOP expansion. |
| `mm_trisc1_math.S:238`, `mm_trisc0_unpack.S:283`, `mm_trisc2_pack.S:272` | `sw` to `INSTRN_BUF_BASE` pushes a Tensix instruction, just like `.ttinsn`. The RISC store may retire before the FIFO accepts the push. |
| `mm_trisc1_math.S:127-175,190-204,241` | `TTREPLAY 16,16,0,1` records the following 16 `MVMUL`s without executing them. The later `0x01800000` `.ttinsn` is template-1 `MOP`, not `REPLAY`: its nine MMIO configuration stores set four inner iterations with `REPLAY(16,16,0,0)` as the loop op. One source `.ttinsn` is not necessarily one executed backend operation. |
| `mm_trisc1_math.S:74,219` | Calls target a helper outside the extracted `kernel_main` assembly; use linked ELF disassembly when that path executes. |

The instruction FIFO can stall its issuing RISC when full. Blackhole hardware
can fuse up to four adjacent TRISC `.ttinsn` pushes when gathering is enabled,
but this tt-metal build disables gathering by default (details below). The
Tensix frontend dequeues at most one instruction per thread per cycle. `STALLWAIT` and
`SEMWAIT` hold a Tensix thread at its Wait Gate; Auto TTSync can also stall RISC
MMIO accesses. Backend work has resource and data dependencies, including the
documented four-cycle no-read window after a `Dst` block write. A constant
per-push latency therefore cannot establish a completion timestamp.

## Frontend and synchronization rules

These are documented rules, not measured cycle costs for this kernel:

- `sw` to `INSTRN_BUF_BASE` and `.ttinsn` both enter the same per-thread
  frontend. The push is a RISC store: its EX1 entry, store-queue emission,
  FIFO acceptance, and Tensix completion are distinct events. Only adjacent
  encoded `.ttinsn` instructions on TRISC are eligible for up to four-way
  I-cache fusion when `cfg0.DisTriscCache` is clear; the fused group acts as
  one store for RISC memory ordering.
  Use the linked ELF instruction addresses, not source comments or `.S` line
  numbers, to form candidate fusion groups.
- **This kernel's default tt-metal run has fusion disabled.** The firmware
  sets `cfg0.DisTriscCache`; a p150a probe read bit 18 set and found no fusion
  in default runs. Model one push per executed RISC instruction. The source,
  opt-in behavior, device results, and the checked-in assembly's build-flag
  caveat are in
  [`../tt-metal/blackhole-instruction-gathering.md`](../tt-metal/blackhole-instruction-gathering.md).
- The three checked-in TRISC `kernel_main` bodies contain 8 math, 19 unpack,
  and 26 pack static `sw` sites targeting `__instrn_buffer`. Decode the
  *executed store address* first and then its 32-bit value; the dynamic words
  require functional execution of RISC branches and register updates. The
  groups below cover these `kernel_main` bodies only; math calls
  `_apply_src_zero_flag_` at `:74/:219`, whose linked body may push more.

  | Thread | Push-site groups in its `.S` | Decoded effect |
  | --- | --- | --- |
  | Math | `:51-64` (6); `:238/:268` (2) | Six `RMWCIB` config words; dynamic `SETC16` selects Dst offset 0 or 512 (`0xb2010000` / `0xb2010200`). |
  | Unpack | `:44-66` (7), `:132-150` (6); `:283/:366` (2); `:302/:320`, `:308/:326` (4) | `RMWCIB`, runtime CB-base `SETDMAREG`, `SETADCXX`; one of two context-selecting MOPs; dynamic CB ack-count `SETDMAREG` followed by `STOREREG` to the CB0/CB1 overlay ack register. |
  | Pack | `:26-34` (4), `:57-94` (9), `:153-156` (4); `:205` (1); `:272-324` (7); `:336` (1) | `SETDMAREG` and `RMWCIB` setup, `WRCFG`; runtime `SETADC`, output-address and CB received-count `SETDMAREG`, `STOREREG` to CB16 overlay, Dst-half `ZEROACC`; final Dst-offset `WRCFG`. |

  Stores to `0xFFB80000` (math `:190-204`, unpack `:201-212`, pack
  `:139-152`) instead configure the MOP Expander and are not instruction
  pushes. The same static `sw` site can execute many times in the loops.
- The first FIFO holds at most 32 instructions and normally stops accepting
  pushes above occupancy 28; a fused four-instruction push from 28 can reach
  32. The MOP and Replay Expanders each emit at most one instruction per
  thread per cycle. Downstream FIFO capacities cannot simply be added to 32
  unless expansion occurs. Exact pipeline delays and same-cycle enqueue /
  dequeue behavior still need device calibration. Gathering disabled does
  not remove backpressure: MOP/Replay expansion and Wait Gate stalls can
  prevent dequeue, while RISC pushes and its four-entry store queue continue
  to fill the input path. A controlled NOP/Wait-Gate probe found that an MMIO
  wall-clock read after 30 default-config pushes waits for gate release,
  while a CSR bracket shows the pushes themselves still advance; see
  [`blackhole-tensix-push-backpressure.md`](blackhole-tensix-push-backpressure.md).
- The math kernel's 16 recorded `MVMUL`s generate no backend work during
  `Load=1, Exec=0`. Each replay emits one recorded instruction per cycle and
  cannot ingest later inputs until it finishes. The configured template-1 MOP
  has outer count 1 and inner count 4; its loop op is `REPLAY(16,16,0,0)`,
  so one MOP requests four 16-instruction replays plus one `SETRWC` end op.
  The unpack kernel records two six-instruction halves at
  `mm_trisc0_unpack.S:151-187`; its template-0 MOP at `:283` or `:366`
  selects one half. Each half contains `UNPACR`, `RDCFG`, `ADDDMAREG`,
  `STALLWAIT`, `WRCFG`, and `NOP`. The pack template-1 MOP configured at
  `mm_trisc2_pack.S:139-152` expands to 16 `PACR` instructions per output
  tile (`:293`). For the checked-in host dimensions and p150a's 100 active
  cores, each core handles four output tiles with `Kt=20`: 80 math MOPs
  expand to 5120 `MVMUL`s and 80 end `SETRWC`s; 80 unpack MOPs expand to
  480 instructions, and four pack MOPs expand to 64 `PACR`s. These are
  backend counts, not counts of RISC pushes or backend completion cycles.
  Expansion must precede backend scheduling and `Dst` analysis. The detailed
  expander timing comes from the upstream Wormhole page, which explicitly
  describes Blackhole's wider MOP counts; its cycle-level details still need
  Blackhole device validation.
- A `Dst` write prevents a Matrix Unit or `PACR` read of the same aligned
  8x16 block during the next four cycles; hardware stalls that Tensix thread.
  This math kernel's 16 recorded `MVMUL`s use address modifiers
  `0,1,0,2,0,1,0,4,0,1,0,2,0,1,0,5`. Under its default full-tile
  configuration, they visit logical Dst block starts `0,8,...,56` twice;
  four HiFi4 replays repeat the sequence. `DstSync::SyncHalf` adds either 0
  or 512, and the Blackhole `Adj16` remap, if enabled, permutes the eight
  starts to `0,32,8,40,16,48,24,56`. Thus the same physical block is
  revisited only after at least eight emitted `MVMUL`s. With at most one
  replay emission per cycle, this specific replay does not force a
  four-cycle same-block stall. Cross-thread `PACR` reads still require
  scheduling against math completion and semaphore release. Pack also issues
  `ZEROACC` before releasing `MATH_PACK`; `ZEROACC` clears Dst row-valid bits,
  so the next math tile must observe that update before accumulation. Do not
  classify it as an ordinary Dst data write solely from its name. This is a
  source-level deduction, not a device timing measurement.
- Manual TTSync `+4` waits for all in-flight work of this Tensix thread;
  `+8` only waits until the MOP Expander has no queued MOP and is idle. The
  preceding `sw` orders earlier instruction pushes before the check; the
  dependent `and` waits for the load result. Independent RISC instructions
  can overlap the load. Neither address is a generic fixed-latency fence.
- Unpack-to-math operand readiness uses the `SrcA`/`SrcB` bank ownership
  handshake, not `UNPACK_SYNC`. The matmul unpack replay's `UNPACR` instructions
  set `FlipSrc`, handing each completed bank to the Matrix Unit. `MVMUL` waits
  at the Wait Gate until both selected banks are available to it; after
  consumption, the math MOP's end-op `SETRWC(CLR_A)` and the per-iteration
  `SETRWC(CLR_B)` return the banks to the unpackers. `UNPACR` itself waits
  before writing a bank that the Matrix Unit still owns. See
  [`UNPACR_Regular.md`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/UNPACR_Regular.md),
  [`MVMUL.md`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/MVMUL.md),
  [`SETRWC.md`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/SETRWC.md),
  and the Blackhole LLK `llk_unpack_AB_matmul.h` / `llk_math_matmul.h`.
  The unpack thread's `UNPACK_SYNC` RISC poll/post and Tensix get limit
  in-flight unpack configuration contexts; its `STALLWAIT` instructions order
  configuration work. Neither is the operand-ready signal to math.
- Math-to-pack and pack-to-math Dst ownership uses `MATH_PACK` with max count 2
  in this kernel's `DstSync::SyncHalf` mode. Math's `tile_regs_acquire()` emits
  `SEMWAIT` on semaphore max to avoid reusing an occupied half. Its
  `tile_regs_commit()` emits `STALLWAIT` for math/SFPU completion followed by
  `SEMPOST(MATH_PACK)`. Pack's `tile_regs_wait()` emits `SEMWAIT` on zero before
  packing; its `tile_regs_release()` emits `STALLWAIT(PACK)`, `ZEROACC`, then
  `SEMGET(MATH_PACK)` to return a Dst half. See `llk_math_common.h`,
  `cmath_common.h`, and `llk_pack_common.h`. Auto TTSync does not track Dst,
  semaphores, or cross-thread dependencies. `SEMWAIT` and `STALLWAIT` each
  have a documented one-cycle lag when a satisfied condition removes their
  blocking mask. These are Tensix Wait Gate effects; pushing their encodings
  does not itself wait for the dependency to resolve at the RISC boundary.

`llvm-mca -mcpu=tt-bh` can provide a static scalar scheduling baseline. It has
no runtime addresses, branch outcomes, MMIO effects, or Tensix queue state.
TT-NPE's public `npeAPI` runs a batch workload with predetermined NoC start
offsets; it does not model CB barriers or provide a live completion callback
for the existing TT-loop scheduler. Its internal `ResponseScheduler` can step
some read responses, but is not an incremental interface to the full NoC
model. The measured TT-NPE accuracy on this kernel used *device-recorded* NoC
issue timestamps, not issue times predicted from these `.S` files.

References: Blackhole Baby-RISC
[`PushTensixInstruction.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/PushTensixInstruction.md),
[`ManualTTSync.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/ManualTTSync.md),
[`AutoTTSync.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/AutoTTSync.md),
[`Dst.md`](../tt-isa-documentation/BlackholeA0/TensixTile/TensixCoprocessor/Dst.md),
[`REPLAY.md`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/REPLAY.md),
[`MOPExpander.md`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/MOPExpander.md),
[`SEMWAIT.md`](../tt-isa-documentation/BlackholeA0/TensixTile/TensixCoprocessor/SEMWAIT.md),
[`STALLWAIT.md`](../tt-isa-documentation/BlackholeA0/TensixTile/TensixCoprocessor/STALLWAIT.md),
and [`../tt-npe/prediction-model.md`](../tt-npe/prediction-model.md).
