# `-mcpu=tt-bh`: a Baby-RISC-V scheduling model and its limits

**Finding.** The `tt-bh` scheduling model adds a Blackhole Baby-RISC-V processor to
LLVM's RISC-V target. It analyses the available `brisc`/`ncrisc`/`trisc` assembly
without silently dropping the scalar instructions covered by this model (the Rocket
workaround dropped 3 and 8 in the tested BRISC/NCRISC inputs).

Model: `tools/llvm-mca-tensix/` in the consuming repo (`RISCVSchedTensixBaby.td` +
a two-hunk registration patch + `apply.sh`). Probe details:
`test/mca_vs_measured/README.md`. Observed on Blackhole p150a, tt-metal `4f9fa9e0`,
`llvm-project` `34152bb7d2d`. **Measured** unless marked otherwise.

## What the model says

The core is 32-bit, in-order, single-issue: `MicroOpBufferSize = 0`, `IssueWidth = 1`.
Three resources: `EX1` (a `BufferSize = 0` *dispatch hazard*, so a long-occupancy
instruction blocks everything behind it), `EX2` (pipelined, multiply/FP only) and the
LSU (pipelined).

| class | modelled | source |
| --- | --- | --- |
| int ALU / shift / Zba / Zbb | 1 | ISA README: "latency 1 and throughput 1" |
| `mul` | 2, throughput 1 | one cycle EX1 + one cycle EX2 |
| `div`/`rem` | **20, blocking** | ISA gives a *range*, 6..33 — this is a `[CHOICE]` |
| load / store | 2 | EX1 + one cycle of LSU |
| `fence` | 9, blocking | measured on p150a |
| amo | 12 | ISA: L1 atomic >= 12 |
| `fadd.s`/`fmul.s`/`.h` | 2 | one cycle EX2 |
| `fdiv.s`/`fsqrt.s` | unsupported | ISA: "not implemented" |

`fence` needed `InstRW`, not a `WriteRes` in a shared schedule file: `FENCE` is
declared `Sched<[]>` in `RISCVInstrInfo.td`, so it carries no scheduling information
in *any* RISC-V model. That is why every stock model reports it as `lack-sched`.

Validated against measurements already in this repository: `fence` 9 vs 9 measured;
dependent `add` 1/op and dependent `mul` 2/op against the documented latencies. It is
**wrong about back-to-back fences** — it charges 9 each, where the measurement says 4
each after the first, so `n` adjacent fences are over-predicted by `5*(n-1)`.

## Gaps that matter

1. **Load latency is keyed on the opcode, not the address.** The ISA documents 2
   cycles for local data RAM or an L0 hit up to >= 12 for an L1 atomic; the model
   cannot tell `lw a5,0(sp)` from an `lw` of a NoC-overlay register. This is the
   single largest source of error and it is not fixable in the model format. Use
   `# LLVM-MCA-LATENCY <n>`, which the model's comments point at.
2. **Divide latency is operand-dependent** (documented 6..33) and the format cannot
   say so; the model picks 20. Budget +/- 13 cycles per divide.
3. **`fence` is flat 9.** See above.
4. **CSR access is documented as frontend-serializing but unquantified**; modelled as
   a plain one-cycle EX1 op, i.e. an occupancy without the serialization.

## Scope

Blackhole (`tt-1xx`), scalar pipeline of RISCV B / T0 / T1 / NC. It does **not** model
the Tensix coprocessor (the `.ttinsn` stream) and does not model RISCV T2's vector
unit. A Wormhole variant is a small delta and the differences are in
`WormholeB0/TensixTile/BabyRISCV/README.md`: `mul` there blocks its successor, there
is no L0 data cache so every L1 load is >= 8, and `fence` is a no-op.

## The rule to carry

> `llvm-mca -mcpu=tt-bh` schedules a static assembly instruction sequence using the
> supplied model. Its output is not a measurement or an end-to-end kernel runtime:
> it does not execute branch paths or polling loops dynamically and cannot account
> for address-dependent memory behavior or NoC completion waits.

### Why the FIFO / Tensix backend cannot be added to this model

There are three separate reasons, and only the first is a scope choice:

1. **Declared scope.** `RISCVSchedTensixBaby.td` states it models the scalar
   pipeline of RISCV B / T0 / T1 / NC only and does not model the Tensix
   coprocessor (the `.ttinsn` stream). `TensixBabyModel` declares exactly three
   resources: `EX1` (`BufferSize = 0`, a dispatch hazard), `EX2` (pipelined,
   mul/FP) and the LSU (pipelined).
2. **No behaviour hook was installed.** The registration patch
   (`tools/llvm-mca-tensix/0001-register-tt-bh.patch`) is two hunks: include the
   `.td`, register the `tt-bh` processor. It adds no `CustomBehaviour`. Stock
   LLVM's only RISCV custom behaviour is RVV-only (`RISCVCustomBehaviour.cpp` is
   entirely `LMUL`/`SEW`/`VXMemOpInfo`), so there is no place a CB or FIFO hazard
   is being handled today.
3. **The format cannot express it.** TableGen `SchedWrite`/`WriteRes` keys on the
   *opcode*, not the address -- the `.td` says so itself in two `[GAP]` comments.
   But the push mechanism in `PushTensixInstruction.md` is address-driven
   (`INSTRN_BUF_BASE` `0xFFE4_0000` -> T0, `0xFFE5_0000` -> T1, `0xFFE6_0000` ->
   T2) *and* state-driven: pushing into a full FIFO stalls the RISCV until space
   frees, and the 32/28 acceptance rule plus the non-additive downstream FIFO
   capacity depend on MOP/Replay expansion happening at runtime. A static
   per-instruction resource declaration cannot represent a stall caused by
   downstream occupancy.

So `.ttinsn` never even reaches the model layer -- the tools README classifies it
as *a parse gap in stock LLVM, not a model gap* -- and a `sw` to
`INSTRN_BUF_BASE` is lowered to `WriteRes<WriteSTW>` with `Latency = 2`,
`ReleaseAtCycles = [1, 1]`, i.e. it retires two cycles after issue. The ISA says
the write-request is only *processed* once it lands in the frontend FIFO, and
everything downstream of that (MOP expansion, Replay, Wait Gate, backend
completion) is outside the model. Do not try to close this in the `.td`; it
requires the streaming MCA driver described in `trisc-matmul-single-core-model.md`.
