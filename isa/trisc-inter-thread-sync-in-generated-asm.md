# How the three compute TRISCs synchronize, read from generated assembly

**Finding (source-read, not device-measured).** In
`example/standalone/matmul_single_core/asm_kernel/` (tt-metal `4f9fa9e0`,
Blackhole p150a), UNPACK / MATH / PACK are three independent RV32 programs with
no shared control flow. The whole async contract is carried by five mechanisms,
and all five are visible in the `.S` once you decode them:

| Mechanism | Where it appears | What it couples |
| --- | --- | --- |
| Hardware semaphores at `pc_buf_base + 4*(8+i)` | plain `lw`/`sw` polling loops | RISC-visible protocol state (MATH_PACK, UNPACK_SYNC) |
| `SEMWAIT` / `SEMPOST` / `SEMGET` / `SEMINIT` in the Tensix stream | `.ttinsn`, decoded below | Tensix-side blocking on a semaphore, per thread class |
| `STALLWAIT` mask | `.ttinsn` | which hardware class of a thread is held at the Wait Gate |
| SrcA/SrcB bank ownership | `UNPACR … SetDatValid`, `SETRWC CLR_A/CLR_B` | UNPACK → MATH |
| Dst half ownership | `MATH_PACK` count + `ZEROACC` + `dest_offset_id` | MATH → PACK |

Cross-thread ordering on the shared Tensix CFG registers is a sixth, weaker
mechanism: the `STALLWAIT(p_stall::STALL_CFG, p_stall::THCON)` /
`STALLWAIT(p_stall::STALL_THCON, …)` pairs and the `ATGETM/ATRELM(mutex=0)`
bursts around the burst of `SETADC*`/`WRCFG` pushes.

## Decoding the semaphore ops

`ckernel_structs.h` defines `t6_sem(i) = 1 << i`, and the semaphore *index* also
names the protocol:

| Index | Bit | Name | Meaning |
| --- | ---: | --- | --- |
| 1 | `0x2` | `MATH_PACK` | math ↔ pack, dest register ownership |
| 5 | `0x20` | `UNPACK_SYNC` | scalar ↔ unpacker, busy unpack contexts |

`SEMWAIT(stall_res, sem_sel, cond)` is `0xa6` with `sem_sel` = the **bit**, not
the index, optionally OR-ed with `0x800` (`p_stall::SEMAPHORE_BIAS`), and
`cond` 1 = `STALL_ON_ZERO`, 2 = `STALL_ON_MAX`. So `0xa6a1000a` is
`SEMWAIT(stall = STALL_MATH|STALL_SFPU|STALL_SYNC, sem = 1<<1, ON_MAX)` —
`cmath_common.h:300` (`math_dest_wait`). `SEMPOST`/`SEMGET`/`SEMINIT` use the
same bit encoding.

`semaphore_read(i)` is `pc_buf_base[PC_BUF_SEMAPHORE_BASE + i]` with
`PC_BUF_SEMAPHORE_BASE = 8`, i.e. byte address `pc_buf_base + 4*(8+i)`.
That is how the scalar loops in the `.S` can be attributed:

| `.S` site | Compiled loop | Source |
| --- | --- | --- |
| `mm_trisc0_unpack.S:26` (`lw a5,52(a2)`, `andi 0xff`, `bne`) | `while (sem(5) != 0)` | `cunpack_common.h:194` `wait_for_idle()` |
| `mm_trisc0_unpack.S:270` (`lw a5,52(a3)`, `andi 0xfe`, `bne`) | `while (sem(5) >= 2)` | `cunpack_common.h:165` `wait_for_next_context(2)` |
| `mm_trisc0_unpack.S:277`, `:360` (`sw zero,52(a3)`) | `sem(5) = 0` → SEMPOST | `llk_unpack_AB_matmul.h:367` |
| `mm_trisc1_math.S:27` (`lw a5,36(a4)`, `andi 0xff`, `bne`) | `while (sem(1) > 0)` | `llk_math_common.h:130` |

`andi 0xff` + `bne` is the compiler's form of `> 0` / `!= 0`; `andi 0xfe` + `bne`
is its form of `>= 2` for the `uint8_t` return. Note `pc_buf_base + 52` is
`4*(8+5)`, not a "mailbox 13".

The `sw; lw; and x0,x0,` sequences at `pc_buf_base + 4` and `+ 8` are a
different thing (`ManualTTSync`, `CoprocessorDoneCheck` / `MOPExpanderDoneCheck`);
see `tti-mop-timing-boundaries.md`.

`UNPACK_SYNC` is **only ever read by the unpack thread itself** — `wait_for_idle()`
and `wait_for_next_context()` (`cunpack_common.h:194` and `:165`) are its only two
readers in the whole tree. It is not a cross-thread semaphore: it is a
scalar↔own-backend distance counter, posted by a RISC store and decremented by
the `SEMGET` sitting behind the unpacker in the Tensix stream. `MATH_PACK` is
the genuinely cross-thread one.

### Semaphore lifetime: nothing resets them between kernel launches

The only initializer is BRISC firmware `device_setup()`
(`hw/firmware/src/tt-1xx/brisc.cc:262` → `c_tensix_core::initialize_tensix_semaphores`,
`hw/inc/internal/tt-1xx/blackhole/c_tensix_core.h:415`). It runs **once in
`main()` at firmware boot** — not per launch — and sets `MATH_PACK` to
max=1/value=0 plus `UNPACK_TO_DEST`, `MATH_DONE`, `PACK_DONE`. `UNPACK_SYNC` is
*not* initialized there (its `ex_sem_init` is commented out at `brisc.cc:268`),
and no host-side or dispatch-side code re-inits them either.

Both start-of-kernel waits follow from that:

* `mm_trisc1_math.S:27` `while (sem(MATH_PACK) > 0) {}`
  (`_llk_math_pack_sync_init_`, `llk_math_common.h:130`, comment "Wait for
  previous packs to finish before claiming all dest") is a **cross-launch**
  drain. Count > 0 means a previous program on this core posted a tile that its
  packer never released — the packer only does `SEMGET` after its `ZEROACC`.
  Only once it reads 0 does the kernel `SEMINIT` it to max=2 for `SyncHalf`.
  The `tensix_sync()` just above only drains *this* thread's in-flight Tensix
  work, so it cannot replace the check.
* `mm_trisc0_unpack.S:26` `wait_for_idle()` is the same kind of drain for the
  unpacker, reading the count the *previous* program's `SEMGET`s left behind.
  Every kernel that uses `UNPACK_SYNC` must therefore leave it balanced at 0.

## The MATH_PACK (dest ownership) protocol

Init: `mm_trisc1_math.S:32` `SEMINIT(max=2, init=0, MATH_PACK)` — max 2 because
startup uses `DstSync::SyncHalf` (`llk_math_common.h:142`).

Per output tile (`mm_trisc1_math.S`, main loop):

```asm
.Lmm_trisc1_math_15:
        .ttinsn 0xa6a1000a   ; SEMWAIT(MATH_PACK, ON_MAX)  <- wait for a free dest half
        ...
.Lmm_trisc1_math_14:
        sw a4,0(s0)          ; push SETC16(DEST_TARGET_REG_CFG_MATH_Offset, half)
        .ttinsn 0x01800000   ; MOP(1,0,0): 4 replays x 16 MVMULs = 64 MVMULs
        .ttinsn 0x3780000f   ; SETRWC(CLR_B, SET_ABD_F) <- returns SrcB to UNPACK
        ...
        .ttinsn 0xa2010810   ; STALLWAIT(STALL_SYNC, MATH|SFPU1)   ckernel.h:296
        .ttinsn 0xa4000008   ; SEMPOST(MATH_PACK)                  ckernel.h:299
```

Packer side, per tile (`mm_trisc2_pack.S`): `SEMWAIT(TDMA|SYNC, MATH_PACK,
ON_ZERO)` at `:262` (`llk_pack_common.h:32`), then after the pack MOP
`STALLWAIT(STALL_MATH, PACK0)` `:324`, `ZEROACC(CLR_HALF, …, where =
`dest_offset_id)` pushed at `:328`, `SEMGET(MATH_PACK)` `:331`, then
`dest_offset_id ^= 1` and a `WRCFG` re-select of the packer dest offset.
Count 2 = two halves in flight; count 0 = both halves consumed.

## The UNPACK_SYNC (context) protocol

`mm_trisc0_unpack.S` main loop: poll CB credit, `wait_for_next_context(2)`,
program the cfg context base addresses, `sw zero,52(a3)` (SEMPOST), then
`STALLWAIT(STALL_UNPACK, TRISC_CFG)` `:280`, `UNPACR SrcB …` `:283`, and
`SEMGET(UNPACK_SYNC)` `:289` (`llk_unpack_AB_matmul.h:410`). The post is scalar
and immediate; the get sits behind the unpacker in the Tensix stream, so the
counter measures in-flight unpack contexts.

### What "context" means

An unpacker **context** is a complete configuration set — per-`THCON_SEC` L1
base address, tile X dim, unpack in/out data format, zero-compress flag —
selected by a context number. `Context_count` (a `THCON_SEC` field) says how
many exist; unpacker 0 supports up to 4, unpacker 1 only 2 (`UNPACR_Regular.md`:
`if (WhichUnpacker == 1 && WhichContext >= 2) UndefinedBehavior()`). An UNPACR
picks one with `ContextNumber`, or with a per-thread `ContextCounter` that
`UNPACR_IncrementContextCounter` advances, and the hardware then adds a
software-writable bias, `WhichContext += UNPACK_MISC_CFG_CfgContextOffset[WhichUnpacker]`.

The matmul LLK ping-pongs two contexts and switches them three ways at once,
all of them visible in the `.S`:

* *Which register the base address goes to*: `THCON_SEC0_REG3_Base_address_ADDR32
  = 76` (ctx0) vs `..._Base_cntx1_address_ADDR32 = 77` (ctx1) — byte offsets
  `304`/`308` in the `0xFFEF0000` CFG window, i.e. `sw t0,304(t3)` at
  `mm_trisc0_unpack.S:275` vs `sw t0,308(t3)` at `:358`. `THCON_SEC1`'s pair is
  124/125 → offsets `496`/`500` (`:276`, `:359`); Sec0 gets operand B and Sec1
  gets operand A because startup uses `SrcOrder::Reverse`.
* *The bias*: a `SETC16(UNPACK_MISC_CFG_CfgContextOffset_0, 0x0101 | 0x0000)`
  pushed by `switch_config_context()` (`cunpack_common.h:170`), which sits in
  the Tensix stream and is therefore ordered *after* the UNPACR that used the
  old context.
* *Which recorded sequence replays*: the 12-word replay buffer holds two 6-word
  halves (`UNPACR + RDCFG + ADDDMAREG + STALLWAIT + WRCFG + NOP`, one per
  context), and the runtime `TT_MOP(0, count-1, unp_cfg_context ? 0xff : 0x00)`
  (`llk_unpack_AB_matmul.h:407`) selects the half.

`wait_for_next_context(2)` is the guard that makes reprogramming safe: with a
2-context ping-pong the context being written now is the one used two
iterations ago, and "fewer than 2 in flight" is exactly the condition that the
intervening one has been consumed.

## SrcA/SrcB ownership is not a pushed instruction

`TTREPLAY 16,16,0,1` (`mm_trisc1_math.S:121`) records the 16 `MVMUL` words that
follow it; the release of SrcA is **inside the MOP template**, not a separate
push: `MopCfg[3] = end_op0 = 0x3740000f` (`SETRWC CLR_A, SET_ABD_F`, written at
`mm_trisc1_math.S:191`), with `MopCfg[5] = loop_op0 = 0x04040100`
(`REPLAY(16,16,0,0)`). The explicit `SETRWC(CLR_B)` at `:239` comes from
`llk_math_matmul.h:773`. So a `.ttinsn` census of the math stream sees one MOP
per K step, not the 64 MVMULs or the two bank releases it produces.

## The packer never pushes a PACR

`mm_trisc2_pack.S:144` is `li a5,-4718592` = `0xFFB80000` =
`TENSIX_MOP_CFG_BASE`, and `:153` stores `0x41000000` (a `PACR`) into
`MopCfg[5]`; the template is outer=4, inner=4, so one pushed `MOP(1,0,0)`
(`:297`) emits 16 `PACR`s, the last one `Last=1` to drain the packer. There is
therefore no `0x41` word anywhere in the `.ttinsn` text of `mm_trisc2_pack.S`.
Same reason `ZEROACC` appears as a plain `sw` (`TT_ZEROACC`, `:327`): its
`where` operand is the runtime `dest_offset_id`.

## Auto TTSync is not what makes these waits unnecessary

`ckernel::set_ttsync_enables<TRACK_ALL>()` exists in the Blackhole LLK
(`common/inc/ckernel.h:628`, a `SETC16` to `ThreadConfig[56]`) but has **no
callers** on Blackhole — only the Quasar LLK calls it — and the generated `.S`
files contain no `SETC16` to register 56. tt-llk's own audit skill
(`.claude/skills/mmio-race-audit/SKILL.md`) states the policy: "Treat WH/BH with
the manual-ordering rules."

Even where it is enabled, `AutoTTSync.md` covers "store to push Tensix
instruction followed by store to Tensix Backend Configuration" by stalling the
RISC store until the pushed instruction *passed through the Wait Gate* — a
narrower promise than `wait_for_idle()`, which needs the unpacker to have
*consumed* every context. The same doc lists what is never tracked: MOP
Expander configuration, Tensix Dst, and **Tensix semaphores** — and
`UNPACK_SYNC`/`MATH_PACK` are semaphores. So Auto TTSync could subsume the
cfg-store-ordering half of `wait_for_idle()`, but not its accounting half, and
it says nothing about work left in flight by a *previous* program.

## Consequence for timing models

A `.ttinsn` count is not a work count, and a RISC push is not an execution.
The blocking points that actually pace these three threads are the Wait Gate
(`SEMWAIT`/`STALLWAIT`), the Src bank handshake, and the MATH_PACK count — none
of which a static instruction census or a stock `llvm-mca` run can see.
