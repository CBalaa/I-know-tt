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
