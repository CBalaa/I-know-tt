# I-know-tt

A shared, continuously growing knowledge base for the **Tenstorrent (TT)** stack.

This repository is consumed as a git submodule by `tt-loop-scheduler`, mounted at
`thirdparty/I-know-tt`. It exists so that knowledge gained while working in a
consuming project is captured once, in one place, instead of being rediscovered.

## Where to put knowledge

Anything learned about Tenstorrent goes here, routed by topic:

| Folder            | Scope                                | Typical contents                                                                                              |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `isa/`            | TT **hardware** and ISA              | Tensix core architecture; RISC-V roles (BRISC/NCRISC/TRISC); NoC; circular buffers; `Dst` registers; tile layout; instruction semantics; memory maps |
| `tt-metal/`       | The **`tt-metal`** runtime & kernels | Kernel APIs; JIT/build; `cb_*` and `noc_*` semantics; profiler and Tracy; watcher; build flags; debugging      |
| `tt-npe/`         | **TT-NPE**                           | NPE usage, traces, performance modelling, what-if analysis                                                     |
| `llvm-mca/`       | **`llvm-mca`** on Tensix asm         | Building llvm-mca for the baby RISCV ISA; running it over Tensix kernel `.S`; scheduling-model gaps; what the numbers do and don't mean |
| `tt-rpm/`         | **tt-rpm** RISC-V performance model  | RPM usage: build and config, Whisper ISS + Sparta/MAP integration, pipeline configs, Konata traces, CoreMark/Dhrystone runs, and where the model diverges from real Tensix behaviour |
| `unclassified/`   | Everything else                      | Tooling, environment setup, links, notes that don't fit above                                                  |

If a note genuinely spans topics, put it where it is most useful and cross-link
rather than duplicating it.

### `isa/` is not `tt-isa-documentation/`

- `tt-isa-documentation/` is the **upstream** Tenstorrent ISA documentation,
  vendored as a nested submodule. Treat it as a read-only reference.
- `isa/` is **ours**: distilled, task-driven notes written from what we actually
  ran into.

Prefer **linking** into `tt-isa-documentation/` over copying text out of it.
Upstream files carry their own licenses (see `LICENSE-APACHE` and
`LICENSE-CC BY-ND 4.0` inside that directory) — copying content here would
change its licensing, linking does not.

### `tt-metal/` here is not `thirdparty/tt-metal`

The `tt-metal` submodule at the consuming project's `thirdparty/tt-metal` is the
**source code**. The `tt-metal/` folder in this repository is **notes about it**.
Same name, different thing.

## Writing notes

- One topic per file, `kebab-case.md`.
- Lead with the concrete finding, then the evidence that supports it.
- **Record the version you observed it on.** Behaviour differs between tt-metal
  releases and between Wormhole and Blackhole. A note without a version is a
  liability.
- Distinguish **measured** from **assumed**. Say which is which explicitly.
- Keep it short. A note nobody reads is worth less than no note.

## Notes

Everything below was observed on tt-metal `4f9fa9e0`, Blackhole p150a, unless the note says
otherwise. Measured vs assumed is marked inside each note.

### `isa/`

| Note | What it answers |
| --- | --- |
| [`l0-data-cache-and-fence.md`](isa/l0-data-cache-and-fence.md) | What a baby RISCV `fence` actually does (it flushes the 64 B **L0 data cache**), why the cache is non-coherent, why Wormhole treats `fence` as a no-op, and why "L1 cache" in tt-metal's comment is a misnomer |
| [`add2-noc-barrier-semantics.md`](isa/add2-noc-barrier-semantics.md) | Blackhole `add_2_integers_in_riscv` NoC command-buffer addresses, read-response/write-ACK barrier events, and measured service-time anchors |
| [`baby-risc-timing-model.md`](isa/baby-risc-timing-model.md) | Persistent timing-layer state and event boundaries used to predict the checked-in `add_2_integers_in_riscv` kernel |

### `tt-metal/`

| Note | What it answers |
| --- | --- |
| [`device-profiler-zone-budget.md`](tt-metal/device-profiler-zone-budget.md) | Why only 125 zones per RISC per kernel run, why the overflow is **silent**, and what does *not* lift the budget |
| [`measuring-device-instrumentation-cost.md`](tt-metal/measuring-device-instrumentation-cost.md) | What a marker actually costs, and why the recorded window and the marginal cost are different numbers (three independent measurement methods) |
| [`wall-clock-registers-and-latch.md`](tt-metal/wall-clock-registers-and-latch.md) | The three wall-clock MMIO addresses, the latch, the fact that it is **per-tile shared**, and why the profiler reads `0x1F4` |
| [`out-of-tree-program-cmake-and-runtime-root.md`](tt-metal/out-of-tree-program-cmake-and-runtime-root.md) | Building and running a standalone program against an existing build tree: `TT-Metalium_DIR`, `TT_METAL_RUNTIME_ROOT`, `OVERRIDE_KERNEL_PREFIX`, kernel paths, and the compile flags host sources from `programming_examples/` expect |
| [`runtime-tree-missing-profiler-headers.md`](tt-metal/runtime-tree-missing-profiler-headers.md) | Why a **profiler** run's `TT_METAL_RUNTIME_ROOT` must point at the **source** tree, not `build_Release/libexec/...` (plain runs work with either) |
| [`noc-trace-stale-profiler-buffers.md`](tt-metal/noc-trace-stale-profiler-buffers.md) | NoC traces containing hour-stale events from cores you never ran |
| [`cb-credit-counters-in-noc-stream-regs.md`](tt-metal/cb-credit-counters-in-noc-stream-regs.md) | `cb_reserve_back` end to end: where the CB credit counters really live (NOC-overlay stream registers, **not** L1), the 16-bit wrap contract, and who zeroes them between kernel runs |
| [`launching-asm-kernels.md`](tt-metal/launching-asm-kernels.md) | Running a hand-written `.S` as a tt-metal kernel via `CreateKernelFromString` + inline-asm `.include`; where the kernel source is spliced in, and the four things that break (LTO dropping asm-referenced globals, LLK per-TU `static` state, `R_RISCV_32`-into-text rejected at load, `.L<n>` label collisions); plus why editing a `.S` does **not** invalidate the JIT cache, and why `DPRINT` never survives into a spliced `.S` by default |

### `tt-npe/`

| Note | What it answers |
| --- | --- |
| [`prediction-model.md`](tt-npe/prediction-model.md) | How NPE actually computes performance |
| [`metrics-semantics.md`](tt-npe/metrics-semantics.md) | What the reported numbers mean |
| [`workload-format.md`](tt-npe/workload-format.md) | Workload schema and traps |
| [`build-and-run-gotchas.md`](tt-npe/build-and-run-gotchas.md) | Build/run gotchas |
| [`matmul-multicore-accuracy.md`](tt-npe/matmul-multicore-accuracy.md) | Measured accuracy of `matmul_multi_core` on p150a |

### `llvm-mca/`

| Note | What it answers |
| --- | --- |
| [`analysing-tensix-asm-with-llvm-mca.md`](llvm-mca/analysing-tensix-asm-with-llvm-mca.md) | How to run `llvm-mca` over Tensix kernel `.S`: the `-mcpu=tt-bh` recipe, the RISC-V-only build, the `.ttinsn`/`TTREPLAY` parse gap, and what the numbers do *not* mean |
| [`tt-bh-model-accuracy.md`](llvm-mca/tt-bh-model-accuracy.md) | The `-mcpu=tt-bh` scheduling model itself, where every latency comes from, its four known gaps, and the measured accuracy: 109 predicted vs 910 actual on `add_2_integers_in_riscv`, and why that gap is structural rather than a model defect |
| [`how-llvm-mca-simulates.md`](llvm-mca/how-llvm-mca-simulates.md) | What the simulator actually is: a cycle-by-cycle *timing* state machine with no data values, no addresses and no PC, whose entire hardware description is TableGen — and why that is the root cause of the model's accuracy ceiling |

### `tt-rpm/`

| Note | What it answers |
| --- | --- |
| [`build-and-run.md`](tt-rpm/build-and-run.md) | Why RPM's own `build_all.sh` does not work on this box (conda g++ does not search `/usr/include`; whisper hardcodes `-l:libboost_program_options.a`), and why Dhrystone deadlocks until `misal_ok` is added to the PMA `attribs` in `tests/whisper.json` |
| [`what-it-models.md`](tt-rpm/what-it-models.md) | What RPM covers and what it does not (no NoC, no device L1/DRAM, MMIO as an ordinary load, RV64-only ELF loading), and the measured-vs-predicted numbers for the runtime-argument prologue of `add_2_integers_in_riscv` |
| [`io-contract-and-invocation.md`](tt-rpm/io-contract-and-invocation.md) | What goes in and what comes out: the flags that matter, the ELF/whisper.json contract, and what the model will and will not accept (source-read, not executed) |
| [`trace-file-mode-is-snapshot-resume.md`](tt-rpm/trace-file-mode-is-snapshot-resume.md) | `--trace-file F` never reads `F`: RPM has no trace parser, the argument only names a Whisper snapshot directory to resume from |

Reproducible harnesses and raw output for the profiler notes live in the consuming repo under
`test/wall_clock_latch/` and `test/zone_cost/`.

## Layout

```text
.
├── isa/                   # TT hardware / ISA notes
├── tt-metal/              # tt-metal runtime & kernel notes
├── tt-npe/                # TT-NPE notes
├── llvm-mca/              # llvm-mca notes
├── tt-rpm/                # tt-rpm notes
├── unclassified/          # everything else
└── tt-isa-documentation/  # upstream ISA docs (nested submodule, read-only)
```

The six knowledge folders carry a `.gitkeep` so git tracks them while empty.
Delete the `.gitkeep` once a folder has real content.

## Consuming this repository

In a fresh checkout of the consuming project:

```bash
git submodule update --init --recursive thirdparty/I-know-tt
```

`--recursive` is required: `tt-isa-documentation/` is a nested submodule and is
not populated by a non-recursive update.

### Adding knowledge from a consuming project

This repository is a submodule, so a file written into `thirdparty/I-know-tt/`
is **not** part of the consuming project's history. Committing it there does
nothing for anyone else. The full sequence is:

```bash
# 1. commit and push in the knowledge base itself
cd thirdparty/I-know-tt
git add -A && git commit -m "isa: document DstRowValid behaviour for MVMUL"
git push origin main

# 2. move the consuming project's pointer to the new commit
cd ../..
git add thirdparty/I-know-tt
git commit -m "Update I-know-tt submodule pointer"
```

Skipping step 2 leaves the consuming project pinned to an older commit, and the
new note silently will not appear for anyone else. Skipping step 1 means the
note exists only in your working copy.
