# tt-npe metrics — what the reported numbers actually mean

**Observed on:** `tt-npe` @ `0d2d4cc26575edae6e2729a4c2ac88f0ba17792e`.

## The headline trap: "NoC Util" is a whole-chip average

```
per timestep:  avg_link_util = 100 * Σ_l min(d_l, BW) / (BW * num_links)
               max_link_demand = 100 * max_l d_l / BW
```

`num_links` is the **entire link population, idle links included**
(`npeDeviceModelUtils.hpp:98,117`). A wormhole_b0 has ~480 links, so **one fully
saturated link reports `avg Link util ≈ 0.2%`**.

**Measured** (single 8192 B worker→worker transfer, wormhole_b0):

| metric | value |
|---|---|
| `avg Link util` | **0.177 %** ← averaged over all links |
| `max Link demand` | **93.667 %** ← the actual hot spot |
| `DRAM BW Util` | 0.000 % |

**Rule: use `max Link demand` (or the per-link list in the timeline) to find hot
spots. `avg Link util` is a chip-wide occupancy figure, not a congestion figure.**

### demand vs util — the deliberate distinction

- **demand** can exceed 100% (overlapping routes sum on a shared link)
- **util** is clamped at 100% via `min(d_l, BW)`

`TimestepStats` documents this (`npeStats.hpp:20-27`). `max Link demand > 100%` is
the signal that the congestion derate is active.

## Field-by-field reference (`deviceStats`)

`computeSummaryStats()` (`npeStats.cpp:122-209`), `T` = number of timesteps.
Access from Python as `result.per_device_stats[-1]` (`MESH_DEVICE == -1`).

| field | formula | unit |
|---|---|---|
| `overall_avg_link_util` | `(Σ_t avg_link_util_t)/T` — the headline "NoC Util" | % |
| `overall_avg_link_demand` | `(Σ_t avg_link_demand_t)/T` | % |
| `overall_max_link_demand` | `max_t max_link_demand_t` | % |
| `overall_max_link_util` | `max_t avg_link_util_t` — **max of the per-timestep *average*, NOT a per-link max** | % |
| `overall_avg_noc0_link_util` / `..._noc1_...` | same, restricted to one NoC's link types | % |
| `overall_avg_mcast_write_link_util` | numerator counts only links carrying `WRITE_MULTICAST`, but **denominator is all links** | % |
| `overall_avg_niu_demand` / `overall_max_niu_demand` | NIU-side demand | % |
| `estimated_cycles` | `worst_case_transfer_end_cycle − golden_start` | cycles |
| `golden_cycles` | `golden_end − golden_start` | cycles |
| `estimated_cong_free_cycles` | `estimated_cycles` from a second full sim with congestion off | cycles |
| `cycle_prediction_error` | `100*(estimated − golden)/golden` | % (signed) |
| `dram_bw_util` / `dram_bw_util_sim` | see below | % |
| `wallclock_runtime_us` | wallclock of the sim | µs |

### Gotchas

1. **`overall_max_link_util` is not a hot-spot metric.** It is `max_t` over the
   per-timestep *average* (`npeStats.cpp:131`). Use `overall_max_link_demand`.
2. **`avg Link util` and `avg Mcast link util` share a denominator** — a trace that
   is 100% multicast writes reports the two as equal (ratio 1.0). Pinned by
   `test_npe_api.cpp:58-61`.
3. **`overall_max_niu_demand` / `overall_max_link_demand` DO use per-timestep
   maxima** — pinned by `test_npe_stats.cpp:12-48`. So max-* fields are inconsistent
   in meaning: the *demand* maxima are true maxima, the *util* max is not.
4. **`avg_niu_demand` divides by the global NIU grid size**, not the per-device size
   computed one line above (`npeDeviceModelUtils.hpp:140-141`; the code calls this a
   "Hack"). On multichip models this **under-reports by `numChips`**. The link path
   does it correctly. *Assumed bug — no test covers it.*
5. **No min fields, no histograms, no percentiles** anywhere in the C++ stats.
   The only percentile code is Python-side over `cycle_prediction_error` across ops
   (`npe_analyze_noc_trace_dir.py:113-121`).

## `estimated_cycles` measures only the golden window

`worst_case_transfer_end_cycle` is updated **only for transfers whose scheduled
`phase_cycle_offset` falls inside `[golden_start, golden_end]`**
(`npeStats.cpp:52-60`). Transfers outside the measured region do not inflate the
estimate. Devices with no qualifying transfers get `estimated_cycles = 0` and their
timestep list is cleared (`:71-76`).

**Consequence:** `estimated_cycles` is *not* "when the workload finished" — it is
"when the last in-window transfer finished". For programmatic workloads where you
invent `golden`, you control what gets measured.

## DRAM / ETH utilization are bytes-over-window, not rates

```cpp
read_bytes  += total_bytes  for transfers whose SRC is a DRAM core
write_bytes += total_bytes  for UNICAST transfers whose DST is a DRAM core
dram_bw_util     = 100 * total_bytes / (golden_cycles    * DRAM_BW_per_chip * num_chips)
dram_bw_util_sim = 100 * total_bytes / (estimated_cycles * DRAM_BW_per_chip * num_chips)
```

(`npeStats.cpp:158-190`)

- The headline `DRAM BW Util` uses **golden** duration; the `_sim` variant uses the
  **simulated** duration. Reporting the golden-based one alongside a predicted
  cycle count is a mixed baseline — be explicit about which you quote.
- Counted **once per transfer**, not per hop.
- **Multicast destinations never count as DRAM writes** — only the unicast
  `Coord` alternative is tested (`:171`).
- Per-controller breakdown available via `getDramBwUtilPerControllerStr()`.
- `getDRAMBandwidthPerChip()` / `PerController()` are **reporting-only** — never used
  in the timing loop. DRAM throughput is enforced in simulation solely through
  per-DRAM-core injection/absorption rates, so **two DRAM cores sharing a controller
  do not share a modelled bandwidth pool**.

ETH util is analogous (`:192-209`): only **destination**-ETH unicast traffic counts
("a write to an ETH core results in a tx from that core"), denominator is
`golden_cycles * ETH_BW_per_link`, and `getAggregateEthBwUtil()` is the arithmetic
mean over all ETH cores.

## Congestion impact

```cpp
if (estimated_cycles == 0 || estimated_cong_free_cycles == 0) return 0.0;
return 100.0 * (estimated_cycles - estimated_cong_free_cycles) / estimated_cycles;
```

(`npeStats.cpp:887-893`) — "% of estimated runtime recoverable without congestion".

This is a **clean isolation**: it is a genuine second full simulation with
`congestion_model_name = "none"` (`npeEngine.cpp:417-441`), sharing the same transfer
schedule and dependency graph, so the delta attributes only to the derate path.

**Measured behaviour:** it is exactly **0.0% whenever no link exceeds 100% demand** —
including the golden example at 99.7% peak demand. It then rises with the number of
equal flows sharing a bottleneck (50% at N=2, 66.7% at N=3, saturating toward 100%).
See `prediction-model.md` §3.

## Determinism

Fully deterministic — no RNG, no sampling, no averaging of passes. Unit tests assert
field-by-field equality across runs (`test_npe_engine.cpp:36-54`); `std::stable_sort`
is used deliberately in queue and dependency construction (`npeEngine.cpp:290-298,
341-347`).

## Human-readable output

`print(result)` → `npeStats::to_string` → the `MESH_DEVICE` entry
(`npeStats.cpp:37-39, 87-120`):

```
  congestion impact:   x.x%
   estimated cycles:   nnnn
      golden cycles:   nnnn
   cycle pred error:   x.x%          (only when golden_cycles > 0)
       DRAM BW Util:   x.x% (using golden)
       DRAM BW Util:   x.x% (using estimated)
        ETH BW Util:   x.x%
      avg Link util:   x.x%
 avg Mcast link util:  x.x%
      max Link util:   x.x%
    avg Link demand:   x.x%
    max Link demand:   x.x%
    avg NIU  demand:   x.x%
    max NIU  demand:   x.x%
   wallclock time:   nnnnn us       (verbose only)
```

Per-controller DRAM and per-core ETH breakdowns are **not** in this text output;
call `getDramBwUtilPerControllerStr()` / `getEthBwUtilPerCoreStr()` from Python.
