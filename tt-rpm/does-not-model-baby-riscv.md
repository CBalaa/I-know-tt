# RPM does not model the Tensix baby RISCV

Observed on tt-rpm `3094a66` (stock `config.yaml`) against the vendored upstream
spec `tt-isa-documentation` `b0cfdc43`,
`BlackholeA0/TensixTile/BabyRISCV/README.md`. Device facts below are
**documented** (upstream spec), not measured here.

## The finding

RPM is not an inaccurate model of the p150a baby RISCV — it is a model of a
**different machine**. Its stock configuration describes a 4-wide
out-of-order RV64 core with a 256 KB L1I and a 64 KB L1D. The baby RISCV is a
32-bit **in-order single-issue** core that retires one instruction per cycle
at 1.35 GHz behind a **64-byte L0 data cache**.

RPM never claims otherwise: `grep -rniE 'ascalon|tensix|brisc|ncrisc|trisc|baby'`
over `thirdparty/tt-rpm/` (excluding `ext/`) returns **nothing**. Nothing in the
repo names a target core.

## Side by side

| | baby RISCV (documented) | RPM stock (`models/cpu/src/core/config.yaml`) |
|---|---|---|
| XLEN | 32 | 64 (`whisper.json` `xlen` is deprecated and ignored) |
| issue | in-order, **single-issue**, 1 instr/cycle | `dispatch_width 4`, `issue_width 8`, `fetch_width 16` B |
| ordering | frontend and EX1 in-order; reordering only after EX1, resolved by the Retire Unit | `ooo_enabled: true`, `rob_capacity 128`, `num_phys_regs 128` |
| clock | 1.35 GHz | `--cpu-freq` default 3.0 GHz |
| L0 D$ | **64 B, per-core, non-coherent; `fence` flushes it** | **no L0 concept at all** (`grep -rniE '\bl0\b\|l0cache\|scratchpad' models/` is empty) |
| L1 | 1536 KiB scratchpad | I$ 256 KB / D$ 64 KB, 64 B lines, 4-/8-way set-associative |
| load latency | **by address region** (table below) | one scalar `lat_load` (1 by default) |
| `mul` | 2 (EX1 + EX2) | `lat_mul 3` |
| `div` | 2 (÷0, ÷1, or -2³¹/-1) to 6–33, data-dependent | `lat_div 12` flat |
| branch mispredict | **4-cycle bubble** (looks like 5 in EX1) | `misprediction_penalty_cycles 15`, `bp_mispred_penalty 15` |
| branch prediction | "The Frontend _predicts_ all control flow" | probabilistic coin flip: `accuracy 0.9` + `rng_seed` |
| NoC / Tensix GPR / MMIO regions | first-class address regions with their own latencies | absent; MMIO is an ordinary cached load |
| store coalescing | same 16-byte L1 region, start addresses within ±4 | store queue + optional forwarding, different rule |

### The documented load-latency table RPM cannot express

| load address range | latency |
|---|---|
| core-local data RAM via `MEM_LOCAL_BASE`; L1 with **L0 D$ hit** | **2** (the minimum possible) |
| mailboxes, PCBufs, Manual TTSync, Tensix semaphores | ≥ 3 |
| Tensix GPRs, Tensix backend config, TDMA-RISC config | ≥ 4 |
| tile control/debug/status, PIC, NoC0/1 config, NoC overlay | ≥ 7 |
| core-local data RAM via slow path | ≥ 8 |

Latency here means "N−1 independent instructions must follow the load to hide
it". This table — not a scalar — is the load model this core needs.

## Why the `lat_load 4` calibration does not rescue it

`test/rpm_vs_measured/` gets the empty-window body to 12.99 vs 13 measured by
setting `lat_load 4`. That number is weaker than it looks:

1. **It is circular.** 4 was reverse-engineered from the same measurement it is
   then compared against. You need the device number to pick the constant, so
   the model contributes nothing you did not already measure.
2. **It is not a portable latency.** An L1 hit with an L0 D$ hit is **2 cycles**;
   the ~4 cycles/load seen in the runtime-argument probe is an effective cost
   for that particular six-load sequence, including its cache state and nearby
   instructions. Separate fence-based probes show L0 misses can be ≥8 cycles.
   A sequence that hits L0, or one with different outstanding requests, will not
   have the same cost.
3. **The one clean agreement is partly luck.** The empty window contains a
   `0x1F8` MMIO read that costs ~7 cycles of cross-clock-domain latency on the
   device and is an ordinary cached load to RPM — the note says so itself.

## The "trust it for long integer code" claim is unvalidated

The CoreMark figure (IPC 1.61, 4,111,373 cycles for 6,607,744 instructions) is
**RPM running RPM's own test suite**. `grep -rln 'coremark' test/ tools/`
returns only `test/rpm_vs_measured/README.md` and `tools/rpm_build.sh`, both
reporting that same self-run. **There is no device CoreMark measurement in this
workspace**, and no baby-RISCV CoreMark to compare against.

So "RPM is accurate on long straight-line integer code" is an *assumption about
a different machine*, not a result. It should not be repeated as guidance.

## What RPM would need to model this core

Structural changes, not knobs:

- an L0 data cache (64 B, non-coherent, flushed by `fence`);
- load latency as a function of address region, not a scalar;
- a 4-cycle mispredict bubble and a real (or at least deterministic) predictor;
- RV32 (RPM's core model only loads 64-bit ELFs);
- region models for Tensix GPRs / MMIO / NoC config.

Of these, only the issue width and ordering are reachable through the in-order
preset plus width knobs.

### The in-order preset still pipelines several operations

`ooo_enabled: false` does remove RPM's issue-queue path: `ChipSim::bindCore`
connects `Rename::inorder_out` directly to `Execute`, and `Rename::tickInorder_`
checks source readiness in program order.  It does **not** make the LSU a
single-outstanding-operation machine, however.  The shipped in-order config
keeps the typed Execute group's `int_max_in_flight: 6`, the LSQ has a 32-entry
load queue, and the structural D-cache can accept one request per bank per
cycle.  `decode.params.lat_load` is also a fixed uop execution latency; it is
separate from the D-cache's hit/miss timing and does not model the baby core's
address-region latency, L0 state, or fence-induced drain.

Therefore setting `ooo_enabled: false`, `dispatch_width: 1`, and
`lat_load: 4` narrows the front end but still allows multiple loads to be in
flight and hides their latency.  This is why the calibrated probe can match an
empty marker window yet predict six consecutive runtime-argument loads at
8 cycles versus roughly 25 cycles on BRISC.

## What to use instead

- The **upstream latency table above** — documented per-region behaviour beats
  any model in this workspace for this core.
- `test/mca_vs_measured` (`llvm-mca -mcpu=tt-bh`) for scheduling-level
  estimates, with the same caveat (`I-know-tt/llvm-mca/tt-bh-model-accuracy.md`).
- The device profiler for ground truth, and TT-NPE only for NoC-bound
  whole-kernel time — NPE has no core instruction timing.

For a baby RISCV, a small analytical model built from the documented latency
table (in-order, 1 IPC, per-region load latency, 4-cycle branch bubble) would
likely beat RPM.

## Related

- `what-it-models.md` — RPM's coverage and the measured prediction-vs-device
  table for `add_2_integers_in_riscv`.
- `../isa/l0-data-cache-and-fence.md` — the L0 data cache and what `fence` does.
