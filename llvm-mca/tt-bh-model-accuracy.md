# `-mcpu=tt-bh`: a baby RISCV scheduling model, and how accurate it is

**Finding.** There is now a real scheduling model for the Tensix baby RISCV in this
repository, named `tt-bh` after the tt-metal toolchain's own CPU name for the core.
It analyses every `brisc`/`ncrisc`/`trisc` kernel we have with **zero instructions
silently dropped** (the Rocket workaround dropped 3 and 8).

On `add_2_integers_in_riscv` it predicts **109 cycles** for the whole kernel against a
measured **910** — an 8.3x gap that is *not* a model defect. On the one segment of
that kernel that contains no waiting, it predicts **35** against a measured **39**.

Model: `tools/llvm-mca-tensix/` in the consuming repo (`RISCVSchedTensixBaby.td` +
a two-hunk registration patch + `apply.sh`). Harness and full numbers:
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

## The accuracy result

`add_2_integers_in_riscv` on BRISC. The total comes for free: the firmware already
brackets every kernel run with a `BRISC-KERNEL` zone, so no kernel edit is needed —
run with `TT_METAL_DEVICE_PROFILER=1` and subtract the 14-cycle marker window.

| | cycles |
| --- | --- |
| measured, C++ JIT path | **910** |
| measured, hand-written `.S` path | 854 |
| `llvm-mca -mcpu=tt-bh`, whole kernel, model defaults | **109** |
| `llvm-mca -mcpu=tt-bh`, whole kernel, per-address load latencies by hand | 279 |

The gap is structural. Instrumenting the kernel with four zones puts the 910 here:

| segment | cycles | share |
| --- | --- | --- |
| `noc_async_read` x2 + `read_barrier` (issue **and** unbounded wait) | 520 | 47.6% |
| `noc_async_write` + `write_barrier` (issue **and** unbounded wait) | 350 | 32.0% |
| the actual integer add | 39 | 3.6% |
| prologue, address generation, marker overhead | 184 | 16.8% |

**~80% of the kernel is NoC command issue plus waiting for the NoC.** `llvm-mca`
analyses one basic block: it models each of the five poll loops as running exactly
once, it has no notion of a NoC round trip, and it does not model the store queue at
all — which is the entire point of `ckernel::load_blocking`. No better scheduling
model can recover any of that.

## Where the comparison is actually meaningful

The one wait-free segment is six instructions: two L1 loads, an `add`, a store, the
`load_blocking` load, and an `and`.

`llvm-mca` says **9**. The hardware says **39**. The model is not wrong about the
instructions; it is wrong about the *loads*. It charges the documented minimum, 2
cycles (L1 with an L0 data-cache hit) — but this segment sits immediately after a
`fence`, and a `fence` flushes the whole 64-byte L0 data cache
([`../isa/l0-data-cache-and-fence.md`](../isa/l0-data-cache-and-fence.md)). All three
loads are therefore L0 *misses* at >= 8.

Annotating just the first two loads with `# LLVM-MCA-LATENCY 8` gets **35** against
the measured 39.

⚠️ **Read that 11% as suggestive, not as proof.** `llvm-mca` reaches 35 by charging
load latency; the hardware is most likely charging a store-queue drain on the
`load_blocking` load. Two different mechanisms landing near the same number is a
coincidence, not a validation. The robust conclusion is the weaker one: *with
per-address load latencies supplied by hand, the model is in the right neighbourhood
for wait-free code.*

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

> `llvm-mca -mcpu=tt-bh` is a **structure** tool for Tensix kernels, not a cycle
> predictor. Trust it for: A/B comparison of two instruction sequences under the same
> model, port/issue pressure, and dependency stalls in straight-line code. Do not
> trust it for: any kernel whose time is dominated by NoC waits (which is most
> data-movement kernels), anything whose load latency you have not pinned down, or
> absolute cycle counts.
