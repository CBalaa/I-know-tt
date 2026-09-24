# tt-rpm: what goes in, what comes out, how to run it

Observed on tt-rpm `3094a66` ("Initial public release", 2026-06-27), with
`ext/map` `4145671` (map_v2.0.26) and `ext/whisper` `674f645` (r1.782-5974).

**Nothing in this note was measured by running the model** — it was written
from source before the model had been built in this checkout (it is built now;
see [`build-and-run.md`](build-and-run.md)). Everything below is read out of the
source and is marked *verified* (code path traced) or *inferred* (mechanism
reasoned from Sparta's code, not executed).

## What the tool actually is

RPM is an **execution-driven, cycle-level RISC-V core model**. It is not a
trace replayer and not an analytical model:

- A hand-written pipeline model (fetch → I$ → decode → rename → issue →
  execute → LSQ/D$ → writeback/ROB) built on **Sparta/MAP**, ticked once per
  cycle by `PipelineClock::tick()` (`models/cpu/src/core/Pipeline.cpp`).
- **Whisper ISS is linked in-process** (`librvcore.a`, `--perfapi` mode) and
  supplies functional results. The model calls `perfApi.fetch/decode/execute/
  retire` instruction by instruction via `ExecutionDriver`.
- Scope is **one core** (`top.core0`). `models/{cluster,fabric,memory_model,
  profiler,soc,sys}` and `models/analysis/*` contain only `.gitkeep`
  placeholders — reserved, not implemented. So are
  `models/cpu/predictors/{tage,btb,ittage,nfp,ras}` and `models/cpu/prefetch/*`.

Two binaries: `build/core/core` (the real model) and `build/simple/simple`
(a toy that runs a fixed 20 ticks and reads a *flat* `config.yaml`, not
Sparta's parameter tree — `models/cpu/src/simple/TopSim.cpp`).

## Inputs

| # | Input | Where it comes from | Notes |
|---|-------|--------------------|-------|
| 1 | **ELF binary** | `--target-elf F` (or legacy positional) | The program to execute. |
| 2 | **`whisper.json`** | Auto-discovered **in the ELF's directory** | **Mandatory.** `buildWhisperArguments()` asserts it exists. Sets `xlen`, `isa`, `memmap` (inst region + PMA attribs). |
| 3 | **Sparta config YAML** | `-c CONFIG.yaml` | Paths are `top.core0.<unit>.params.<param>`; 136 `PARAMETER()` declarations. See `models/cpu/src/core/CONFIG_README.md`. |
| 4 | **Parameter overrides** | `-p PATH VALUE` (repeatable) | e.g. `-p top.core0.writeback.params.retire_width 8`. |
| 5 | **Run bounds** | `-r N` (cycles), `-i N` (instructions) | `-i 0` = no limit, run to program exit. |
| 6 | **Clock** | `--cpu-freq GHZ` (default 3.0) | Creates the single `cpu_clock`. |
| 7 | Whisper flags | `whisper_flags` param | Extra `--flag` tokens passed through to Whisper. |
| 8 | *(trace mode)* `--trace-file F` | see `trace-file-mode-is-snapshot-resume.md` | **Not** an instruction stream. |

### The ELF has hard constraints (verified)

- **Bare-metal only.** `tests/Makefile` builds with `-ffreestanding
  -nostartfiles -mcmodel=medany -march=rv64gc -mabi=lp64d -T link.ld`,
  linked at `0x80000000`.
- **No `ecall`.** The Makefile's `run_model` greps the disassembly and warns:
  an `ecall` "will stall the model". Exit is signalled by a **store to the
  HTIF `tohost` address**, which Whisper turns into a `CoreException` that
  `ExecutionDriver::retireInstruction` catches and converts into a graceful
  finish (`mFinishedOverride = true`), not an error.
- `whisper.json`'s `isa` must cover the binary. `tests/whisper.json` declares
  `rv64imac`; the toolchain builds `rv64gc`, which is fine only because the
  shipped workloads never execute FP.
- A glibc cross-compiler (`riscv64-unknown-linux-gnu-gcc`) does **not** work —
  it emits unresolved `strcmp@GLIBC` and similar.

## Outputs

**The headline output is a summary block written to `stderr`** (not stdout) by
`PipelineClock::reportStats()`, emitted every `kReportInterval = 1_000_000`
cycles and once more at termination:

```
=== Pipeline Stats @ cycle <N> ===
  IPC:       <ipc>  heartbeat=<hb_ipc>  (retired=<N>, cycles=<N>)
  Speed:     <kips> KIPS  <khz> KHz  (heartbeat: ... , wall=<s>s)
  Fetch:     fetched=...  buffer_full_stall_cycles=...
  I-Cache:   hits=...  misses=...  (hit_rate=...%)
  FetchQ:    enqueued=...  branch_wait_cycles=...
  Decode:    decoded=...  mispred_stall_cycles=...
  DecodeQ:   enqueued=...
  BP:        correct=...  mispredicted=...  (accuracy=...%)
  Rename:    dispatched=...  prf_stall_cycles=...
  Issue:     issued=...  stall_cycles=...
  Execute:   executed=...
  LSQ:       loads=...  stores=...
  D-Cache:   hits=...  misses=...  (hit_rate=...%)
  Writeback: retired=...
```

This line is the contract `scripts/parse_scripts/parse_stats.py` relies on: it
regexes `IPC:\s+([\d.]+)\s+heartbeat=[\d.]+\s+\(retired=(\d+),\s+cycles=(\d+)\)`
and reports `total_cycles`, `total_instructions`, `ipc`, `retire_samples`,
`setup_time_s`. **`ipc` is the number the tool exists to produce.**
### The stat names behind the summary

The counters are registered per Sparta unit, so `--report-all` keys them as
`top.core0.<unit>.<stat>`. The set is small enough to list (~45):

| Unit | Stats |
|------|-------|
| `fetch` | `num_fetched`, `num_buffer_full_stall_cycles`, `num_mispred_penalty_stalls`, `num_redirects` |
| `icache` | `num_hits`, `num_misses` |
| `fetch_queue` | `num_enqueued`, `num_dequeued`, `num_branch_wait_cycles` |
| `decode` | `num_decoded`, `num_mispred_stall_cycles` |
| `decode_queue` | `num_enqueued`, `num_dequeued` |
| `branch_predictor` | `num_correct`, `num_mispredicted` |
| `rename` | `num_dispatched`, `num_prf_stall_cycles`, `num_checkpoint_stall_cycles`, `num_squashed` |
| `issue` | `num_issued`, `num_stall_cycles`, `num_speculative_wakeups`, `num_not_oldest_selected`, `num_oldest_selected`, `num_dep_local_routes`, `num_dep_local_fallback`, `num_least_occupied_routes`, `num_round_robin_routes`, `num_type_affinity_routes` |
| `execute` | `num_executed`, `num_writebacks`, `num_mispredictions` |
| `lsq` | `num_loads`, `num_stores`, `num_forwards`, `max_lq_occupancy`, `sum_lq_occupancy`, `max_sq_occupancy`, `sum_sq_occupancy` |
| `dcache` | `num_hits`, `num_misses` |
| `writeback` | `num_retired` |
| `l2cache` | `num_hits`, `num_misses`, `num_coalesced`, `num_writebacks` |
| `flush_arbiter` | `num_bp_flushes`, `num_exe_flushes`, `num_dropped_bp_flushes` |
| `edriver` | `instructions_fetched`, `instructions_committed` |

Note `num_hits`/`num_misses`/`num_enqueued` are reused across units — always
qualify by unit path. The `lsq` `max_*`/`sum_*` occupancy pairs are the ones to
reach for when IPC looks wrong.

Other outputs, all opt-in:

| Output | Enabled by | Format |
|--------|-----------|--------|
| Full Sparta stats report | `--report-all F [format]` | text (default); a second token selects a formatter (e.g. `html`) |
| Per-unit logs | `-l top.core0.<unit> <level> <file>` | e.g. `-l top.core0.writeback info wb.log`. `parse_stats.py` also understands the per-retire `[writeback] cycle N retired tag=T pc=0x..` lines this produces |
| Pipeline visualizer | `visualizer_enabled: true` (+ `visualizer_format: table\|waterfall\|log`, `visualizer_output_file`) | `pipeline.txt`; optional per-cycle debug dump |
| Cache tracer | `cache_viewer_enabled: true` (+ `cache_viewer_format: log\|waterfall\|summary`) | to file / stdout / stderr |
| Deadlock diagnostic | automatic | to `stderr`, after 1000 consecutive cycles with no retirement |
| Wrapper chatter | always | to **stdout**: `[chipsim] ...`, the assembled `Whisper command line args: ...`, `[doSetupHelper] Spent: N seconds on setup` |

`scripts/run_scripts/run_sim.sh -m core -n <cycles> -e <elf>` wraps a run and
writes `build/core/output/<timestamp>/out.txt` (stdout) and `ilog.txt`
(stderr). **Because the summary goes to stderr, the IPC block lands in
`ilog.txt`, not `out.txt`** — despite `run_sim.sh`'s own comment describing
`out.txt` as the "retire trace".

## How to invoke

Build (needs CMake ≥3.17, g++-13+/C++23, Boost ≥1.74, yaml-cpp ≥0.7,
RapidJSON, SQLite3, zlib, HDF5):

```bash
git submodule update --init --recursive
bash scripts/build_scripts/build_all.sh          # deps + models + tests
# or: bash scripts/build_scripts/build_model.sh --model core [--clean --type Debug]
bash ci/docker_build.sh --test                   # or all-in Docker
```

Run, the supported path (execution-driven):

```bash
# Run to completion: program exit is detected via the HTIF tohost store.
./build/core/core -i 0 -c models/cpu/src/core/config.yaml \
    --target-elf tests/build/coremark.bare.elf

# Bounded run, in-order config, parameter sweep, stats report.
./build/core/core -r 20000000 -c models/cpu/src/core/config_inorder.yaml \
    --target-elf tests/build/coremark.bare.elf --report-all stats.txt
./build/core/core -r 20000000 -c build/core/config.yaml \
    -p top.core0.writeback.params.retire_width 8 \
    --target-elf tests/build/coremark.bare.elf
./build/core/core --show-tree          # dump the resource/parameter tree
```

Legacy positional syntax still works (`core [cycles] [elf|trace]`); `main.cpp`
rewrites it into flags before handing off to Sparta. `run_sim.sh` uses it.

Workloads need the bare-metal toolchain:

```bash
bash tests/install-toolchain-conda.sh   # creates conda env 'riscv'
conda activate riscv
make -C tests run_coremark              # builds + runs
```

## Gotchas worth knowing before trusting a number

- **`-r` is cycles, not ticks — the README is wrong.** `README.md` says
  "`-r TICKS` (~3 ticks/cycle at 3 GHz)". Sparta's `-r` is documented as
  `[CLOCK] CYCLE` and is resolved against a runtime clock; RPM creates exactly
  **one** clock, `cpu_clock`, so `-r N` = N core cycles. Corroboration:
  `scripts/run_scripts/slurm/run.py` maps the field named **`num_cycles`** to
  `-r`, and `configs/experiment_configs/core_baseline.yml` uses
  `num_cycles: 20000000`. (Sparta's timebase is 1 tick = 1 ps, so a 3 GHz
  cycle is 333 ticks — the README's "3" looks like a dropped factor of ~100.)
  *Inferred from source, not executed.* When in doubt use `-i`, which is
  unambiguous.
- **`whisper.json` must sit next to the ELF.** Put the ELF somewhere else and
  setup aborts. `tests/Makefile` copies `tests/whisper.json` into
  `tests/build/` for exactly this reason.
- **The branch predictor is a coin flip, not a predictor.** The only
  implementation (`Frontend/BranchPredictor/BranchPredictor.hpp`) has
  `accuracy` (default 0.95, `config.yaml` sets 0.9) and `rng_seed` — it draws
  mispredictions from an RNG. The TAGE/BTB/RAS/ITTAGE directories are empty.
  Any IPC number is therefore only as meaningful as that one probability.
- **Caches are two different things depending on `cache_mode`.** `structural`
  (the default in `config.yaml`) models a real set-associative cache with
  size/line/associativity and LRU/PLRU; `probabilistic` just rolls `hit_rate`
  against an RNG. `config.yaml` ships `icache`/`dcache` in `structural` mode.
- **L2 is off by default** (`l2cache.enabled: false`); misses fall through to a
  fixed `miss_latency`.
- **`--show-tree` and `-p` use Sparta tree paths**, so the parameter namespace
  (`top.core0.<unit>.params.<name>`) must match the built tree exactly. Note
  the tree units are named without the `params` level: `top.core0.fetch.params.
  initial_pc`, etc. `ChipSim::configureTree_` overwrites `fetch.initial_pc`
  from the ELF's entry PC, so setting it in YAML has no effect.

## Environment note for this host

A conda env **`rpm`** already exists with the full host build dependency set
(gcc 14.4, cmake, boost 1.92, hdf5 2.2, yaml-cpp 0.8, rapidjson, sqlite,
zlib) — `conda activate rpm` is the way to build here. The conda env **`riscv`**
exists but is **empty of the toolchain** (no `riscv64-unknown-elf-gcc`), so
`tests/` cannot be built until `tests/install-toolchain-conda.sh` is re-run.
