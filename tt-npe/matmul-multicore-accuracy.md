# Measured accuracy: matmul_multi_core on Blackhole p150a

**Measured** on `tt-npe` @ `0d2d4cc2`, tt-metal `4f9fa9e0`, arch **blackhole** (p150a,
`CHIP_FREQ[MHz]: 1350`), 100-core matmul, congestion model `fast`. Trace:
`example/standalone/matmul_multi_core/`.

> **Correction (same session).** An earlier revision of this note headlined
> "+0.17 % / 100 cores within ±2 %" as if that characterised the model. It does not —
> that figure holds **only at `cycles_per_timestep = 32`**, and the metric is far less
> informative than the percentage suggests. Both points are now sections 1 and 2 below.
> Reproduce with `example/standalone/matmul_multi_core/npe_analysis.py`.

## 1. The error depends steeply on `cycles_per_timestep`

Same trace, only the timestep varied:

| `cycles_per_timestep` | kernel error | per-core mean \|error\| | per-core worst | cores within ±2 % |
|---|---|---|---|---|
| 8 | −0.05 % | 0.55 % | 1.42 % | 100 / 100 |
| 16 | −0.06 % | 0.55 % | 1.40 % | 100 / 100 |
| **32** | **+0.17 %** | **0.67 %** | **1.65 %** | **100 / 100** |
| 64 | +1.00 % | 1.19 % | 2.47 % | 78 / 100 |
| 128 | **+5.83 %** | **9.42 %** | **20.6 %** | **7 / 100** |

32 is the default for the **Python** entry points (`tt_npe.py:20-21`,
`npe_analyze_noc_trace_dir.py:218`); the **C++ config default is 128**
(`npeConfig.hpp:50`). Never quote an accuracy figure without the timestep it was
measured at.

## 2. The metric is insensitive, so a small percentage is weak evidence

tt-npe is trace-driven: it consumes the actual NoC issue timestamps and predicts only
the NoC service time; the golden reference comes from the same trace (in-sample fit).

Splitting each core at its last NoC operation:

| quantity | measured | model |
|---|---|---|
| "last NoC op → kernel end" tail | 415 … 594 cycles, **σ = 30** | 118 … 3882 cycles, **σ = 1360** |

Pearson **r = −0.02** — the model's tail is uncorrelated with the measured tail. The
percentage is small only because the kernel is ~2·10⁵ cycles long.

Absolute per-core error: median **+525** cycles, p90 **+3000**, max **+3352**.

Whole-kernel decomposition:

```
last NoC operation (global) = 210250
measured kernel end         = 210752  -> measured tail = 502 cycles
tt-npe predicted            = 211115  -> model tail    = 865 cycles
overestimate                = 363 cycles = +0.172 %
```

**Accurate phrasing:** *tt-npe overestimates the post-NoC tail by a few hundred cycles
per core — under 1 % of this kernel's runtime.* Not "the model predicts per-core timing
to 2 %".

## 3. Reference numbers at `cycles_per_timestep = 32`

| metric | value |
|---|---|
| whole-kernel error | **+0.17 %** (run A) / **+0.49 %** (run B) |
| per-core completion error | mean **+0.48 %**, median **+0.25 %**, mean\|err\| **0.67 %** |
| per-core range | −0.40 % … +1.65 % (σ = 0.71 %) |
| cores within ±1 % | 64 / 100 |
| cores within ±2 % | 100 / 100 |

Workload scale: measured kernel ≈ **210 653 cycles ≈ 156.0 µs**; tt-npe estimated
**211 115**, golden **210 752**. 14 682 NoC transfers, 6599 timesteps, ~17 s wallclock.

Run-to-run spread comes from `std::random_device` input generation; per-core statistics
were stable across runs (mean \|err\| 0.67 % vs 0.68 %).

The model is **slightly pessimistic** — it predicts cores finishing a fraction of a
percent *later* than they do.

### Error is structured, not uniform

* the **left half of the grid** (cols 1-6) carries a consistent **positive** bias, up to
  +1.65 %;
* the **right half** (cols 12-15) sits essentially at **zero** (−0.2 % … +0.3 %);
* there is visible row-to-row structure too.

Consistent with a spatially asymmetric model, but this run does not isolate the cause.

## 4. What "per-core error" means here

tt-npe reports statistics per **device**, not per core, so per-core numbers must be
derived:

* **measured completion** = per core, `max(timestamp)` over every event carrying
  `proc`/`sx`/`sy` — replicating tt-npe's own window (`computeGoldenCyclesAndT0`,
  `npeWorkloadIngest.cpp:221-269`), which uses *all* such events, not just
  `BRISC-KERNEL`;
* **predicted completion** = per core, `max(end_cycle)` over the transfers that core
  issued, from the emitted timeline (`noc_transfers`).

**Attribution matters**: tt-npe swaps src/dst for READs so data always flows src→dst
(`npeWorkloadIngest.cpp:448-451`). The issuing core is **`dst` for READs** and **`src`
for WRITEs**. Using `src` for everything mis-attributes every read to DRAM.

**Key ordering matters**: trace fields are `x = col`, `y = row`; timeline `src`/`dst` are
`[device, row, col]`. Keying one side `(x, y)` and the other `(row, col)` matches only
the diagonal cores (25 of 100 here) and **inflates the apparent error ~3×** (mean
+1.21 %, max +5.19 %). Always assert the core count matches.

**Shape asymmetry**: timeline `src` is a **flat** `[device,row,col]`; `dst` is a **list**
of `[device,row,col]`. `t['src'][0]` is the device id, not a coordinate.

### Verify your golden reconstruction first

It must reproduce tt-npe's reported `golden_cycles` exactly:

```
t0            = min(timestamp) over ALL events
golden_start  = min over cores of min(timestamp) - t0        -> 0
golden_end    = max over cores of max(timestamp) - t0 - 20   -> 210752
```

The `-20` is a flat correction for the last-NoC-event → kernel-end gap
(`npeWorkloadIngest.cpp:263-265`). Using only `BRISC-KERNEL` events gives 210653 instead
— a 99-cycle mismatch that silently corrupts a per-core comparison.

## 5. Secondary metric: per-core *duration* error

Comparing predicted NoC activity span against measured kernel **duration**: mean
**−0.75 %**, mean\|err\| **1.27 %**, range −3.07 % … +1.16 %. Expected — duration
includes TRISC compute, which tt-npe does not model. The **completion-time** metric is
the meaningful one.

## 6. Caveats on this measurement

1. **The trace needed cleaning.** Raw captures contained 10 stale cores with ~1-hour-old
   timestamps; unfiltered, tt-npe hangs. See
   `../tt-metal/noc-trace-stale-profiler-buffers.md`. Numbers are from the filtered trace
   (43 318 of 48 138 events).
2. **The reference is inflated by the profiler.** tt-metal warns on every NoC-collecting
   run: `Profiler NoC events are enabled; this can add 1-15% cycle overhead to typical
   operations!` tt-npe does not model that overhead, so the golden side is biased high.
3. **The kernels are Tracy-instrumented.** `ENABLE_TRACY` defaults to 1 and
   `PROFILE_KERNEL` is injected whenever `TT_METAL_DEVICE_PROFILER=1` — which NoC
   collection requires. See `../tt-metal/device-profiler-zone-budget.md`.
4. **Device sync was off** (analyzer warning). This is a 1×1 mesh, so likely moot, but it
   was not silenced.
5. **Collecting traces needs a one-time header fix** — the runtime tree is missing three
   profiler headers. See `../tt-metal/runtime-tree-missing-profiler-headers.md`.

## 7. Bottom line

Use tt-npe here for ranking congestion, spotting hot links and estimating NoC occupancy.
At `cycles_per_timestep = 32` its end-to-end timing lands within ~0.5 % of a
profiler-inflated reference — but that is a weak claim, and at the C++ default of 128 the
same workload is off by ~6 % overall and ~9 % per core.
