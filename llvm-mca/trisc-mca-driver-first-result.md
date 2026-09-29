# First generated-assembly TRISC driver result

Observed on tt-metal `4f9fa9e0`, Blackhole p150a, with the generated
`matmul_single_core` assembly in the consuming repository.

## What the driver does

`example/standalone/matmul_single_core/predict_trisc.py` reads the generated
`.S` files directly.  It follows dynamic scalar branches with the persistent
Baby-RISC timing core in `ttloop/baby_risc.py`, then writes a flattened scalar
trace to a temporary assembly file and invokes `llvm-mca -mtriple=riscv32
-mcpu=tt-bh`.  The C++ kernel is not passed to LLVM-MCA.  Reader and Writer
CB visibility is an explicit JSON input; the all-ready input is only an
isolated scalar experiment.

The dynamic run found these counts for `Mt=Kt=Nt=20`:

| thread | scalar instructions | dynamic `sw __instrn_buffer` | backend expansion |
| --- | ---: | ---: | --- |
| TRISC0 | 350,330 | 40,013 | 8,000 x 6 UNPACK words = 48,000 |
| TRISC1 | 29,747 | 8,406 | 8,000 x 4 x 16 MVMUL = 512,000 |
| TRISC2 | 17,293 | 3,218 | 400 x 16 PACR = 6,400 |

The dynamic store counts are not the static `.ttinsn` counts (29, 41, and 36).
Runtime stores to `__instrn_buffer` are part of the push stream and must be
counted after branch execution.

For the measured-CB run, the actual scalar instruction streams were:

| thread | issued RV trace | `TTREPLAY` | `sw __instrn_buffer` | instructions passed to MCA |
| --- | ---: | ---: | ---: | ---: |
| TRISC0 | 2,872,727 | 1 | 40,013 | 2,872,726 |
| TRISC1 | 29,747 | 1 | 8,406 | 29,746 |
| TRISC2 | 3,299,377 | 0 | 3,218 | 3,299,377 |

MCA sees the scalar RV32 operations: ALU, branches, loads, stores, and shifts.
An instruction-buffer `sw` is still modelled as a scalar `sw`; its Tensix push
semantics are not visible to the `tt-bh` scheduling model.  Literal `.ttinsn`
directives are skipped by the trace parser, and `TTREPLAY` is explicitly
removed before the temporary assembly is passed to MCA.  The corresponding
UNPACK/MVMUL/PACR work is accounted for by the separate backend expansion
layer, not by the MCA instruction count.

## First comparison

One assembly-only profiler run recorded `TRISC-KERNEL` windows of 9,051,869
(TRISC0), 9,042,842 (TRISC1), and 9,029,218 (TRISC2) cycles.  With all CB
events ready, dynamic `tt-bh` MCA reported 382,333, 29,751, and 17,697 scalar
cycles respectively.  These values intentionally exclude Tensix backend
completion and external CB waits; they are scalar baselines.

A separate CB probe supplied 16 observed CB0 and CB1 visibility events and 16
observed CB16 credit-release events (the predictor adds the two initial CB16
free slots).  After correcting the probe overhead with
the NCRISC/BRISC kernel endpoints, the extrapolated periods were 1,137,
1,137, and 22,736 cycles.  Feeding that fixed event trace to the persistent
Baby-RISC timing layer produced 9,097,795, 9,216,000, and 9,050,916 cycles.
Against the assembly-only windows, the errors were 0.51%, 1.91%, and 0.24%.
The 9,216,000 value uses a provisional 18-cycle MVMUL issue period; it is a
calibration input, not a documented MVMUL latency.

The result must be reported in layers.  For the same measured event trace,
plain dynamic-trace MCA produced 3,745,529 (TRISC0), 29,751 (TRISC1), and
4,120,302 (TRISC2) scalar cycles.  MCA cannot see a future CB visibility event
or Tensix backend completion.  The sub-2% figures therefore validate the
hybrid event-aware scalar layer plus the fixed backend envelope, not an
unmodified LLVM-MCA-only prediction of a complete TRISC kernel.

The next model layers must preserve the event boundaries documented in
`trisc-matmul-single-core-model.md`: scalar store issue, FIFO acceptance,
MOP/Replay expansion, Wait Gate release, backend dispatch/completion, Manual
TTSync release, and CB visibility.  A single fixed latency for a `sw` or a
`.ttinsn` cannot represent those boundaries.

## Why the event-aware layer lands: the CB-bound threads are producer-paced

Re-running the predictor reproduces the table above exactly.  Instrumenting the
TRISC0 dynamic trace explains the mechanism rather than just the agreement:

| poll site | dynamic loads | instructions charged |
| --- | ---: | ---: |
| `lw a5,40(a1)` CB0 wait (`.Lmm_trisc0_unpack_77`) | 367,206 | 1,101,618 |
| `lw a5,40(a2)` CB1 wait (`.Lmm_trisc0_unpack_78`) | 489,596 | 1,468,788 |
| `lw a5,52(a3)` local-state wait (`.Lmm_trisc0_unpack_79`) | 8,000 | 24,000 |
| poll bodies, total | 864,802 | 2,594,406 |

90.3% of TRISC0's 2,872,727 dynamic instructions are the three `lw`/`zext.h`/`branch`
poll bodies.  Each poll iteration costs about ten cycles, so 8,000 CB0 and 8,000
CB1 events produce roughly 46 and 61 poll iterations per event respectively.

TRISC0's 9,097,795 cycles divided by its 8,000 CB0 pushes is 1,137.2 cycles per
event, which is the extrapolated CB0/CB1 push period itself.  The unpack thread
is therefore not scheduled by its own code at all: it is paced one-to-one by the
reader's push rate, and the predictor's accuracy on this thread comes from
transcribing the measured producer period into the poll branch, not from the
scalar schedule.

The corollary is that `MCA + backend` is the wrong shape for TRISC0 and TRISC2.
MCA pipelines the poll iterations as independent loads and so reports about 1.3
cycles per instruction where the branch-dependent reality is about 3.15; that is
the whole 58.6% / 54.4% gap.  Only TRISC1 is backend-bound, where the fixed
envelope is the prediction and the 1.91% error is inherited from the 18-cycle
MVMUL constant (an exact fit would need 17.66 cycles per MVMUL word).

## Fragility in the backend layer

The backend envelope is `setup_cycles + sum(words_r * period_r)` over
`unpack`/`mvmul`/`pacr`, and it enters the result only as a floor:
`thread_model_cycles = max(mca_cycles, core_cycles, backend_cycles)`.  It is an
envelope, not a composition -- TRISC0's all-ready row is 561,821 (the scalar
side), not 561,821 + 48,000.

Two things to know before touching this layer:

1. **Only one term is ever non-zero per thread.**  `_expanded_words` returns
   `{unpack: 48000}` for TRISC0, `{mvmul: 512000}` for TRISC1 and `{pacr: 6400}`
   for TRISC2.  The three-term sum is therefore always `setup + one product`.
   The additive form reads as if three independent units contribute in
   parallel, which is wrong -- the backend issues at most one word per cycle and
   MOP/Replay expansion serializes.  Do not extend one thread to two non-zero
   terms without replacing the sum with a serialization model.
2. **`setup_cycles` has no CLI flag.**  `main()` builds
   `BackendConfig(unpack_period, mvmul_period, pacr_period)` positionally, so
   `setup_cycles` stays at its 0.0 default and `--setup-cycles` is rejected as an
   unknown argument.  Startup/drain latency is therefore not modelled at all
   today, despite the field existing.

Also note what the four constants actually are, because they do **not** track
`Mt/Kt/Nt`: `8_000 = Mt*Nt*Kt`, `400 = Mt*Nt`, and the `4 * 16` is the HiFi4 MOP
shape (1 MOP -> 4 Replay -> 16 MVMUL words each).  The predictor never reads
`asm_kernel/metadata.txt`, which is where `TT_ASM_COMPUTE_COMPILE_TIME_ARGS`
records the dimensions.  Regenerating the assembly for different compile-time
arguments leaves these counts silently stale.
