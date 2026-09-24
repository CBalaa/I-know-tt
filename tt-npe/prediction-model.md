# tt-npe prediction model — how it actually computes performance

**Observed on:** `tt-npe` @ `0d2d4cc26575edae6e2729a4c2ac88f0ba17792e` (branch `main`), built with Clang-20.
**Architectures:** wormhole_b0 (12×10), blackhole / P150 (12×17). Constants differ per arch — both listed below.

## The finding

tt-npe is **not** a cycle-accurate event-driven simulator. It is a **fixed-timestep
fluid/throughput model**. There is no event queue, no flit, no queue depth, and
**no per-hop latency inside the simulation loop**. Per timestep it gives each live
transfer a rate and multiplies that rate by a **proportional-share congestion
derate** computed from summed per-link demand.

The whole model is essentially:

```
rate(transfer)  = min(injection_rate, table_bw(packet_size, num_packets))
rate           *= congestion_derate              # only when a link exceeds 100% demand
duration        = ceil(total_bytes / rate)       # accumulated per timestep
```

## 1. Time model — fixed timestep, no events

- Loop advances by a constant: `curr_cycle += cfg.cycles_per_timestep`
  (`tt_npe/cpp/src/npeEngine.cpp:684`). Timestep `k` is `[k*cpt, (k+1)*cpt)`.
- `cycles_per_timestep` default is **128** in C++ config (`npeConfig.hpp:50`) but
  **32** in the Python CLI (`tt_npe/py/pycli/tt_npe.py:20-21`) and hardcoded 32 in
  the tt-metal batch driver (`tt_npe/py/util/npe_analyze_noc_trace_dir.py:218`).
- **Measured:** timestep size is a real accuracy knob but a small one for
  dependency-free workloads — the same 4-transfer workload gave
  `est_cycles` 1171 (cpt 16/32) → 1168 (64) → 1167 (128/256/512), i.e. ~0.3%.
  It matters more when dependency edges are dense, because activation is tested at
  the **end** of a timestep (`npeEngine.cpp:536`), quantizing every dependency
  release by up to `cpt` cycles.
- Hard cap `MAX_CYCLE_LIMIT = 50'000'000'000` → `EXCEEDED_SIM_CYCLE_LIMIT`
  (`npeEngine.hpp:93`, `npeEngine.cpp:679-681`).

## 2. Per-transfer rate — the bandwidth table

`updateTransferBandwidth()` (`tt_npe/cpp/include/npeDeviceModelUtils.hpp:60-76`):

```cpp
lt.curr_bandwidth = std::fmin(lt.params.injection_rate, noc_limited_bw);   // :74
```

`noc_limited_bw` comes from `interpolateBW()` (`:16-58`), which linearly
interpolates a **packet-size → bytes/cycle** table and blends in a peak value for
the first packet:

```
steady_state_bw   = linear interp of table at packet_size
first_transfer_bw = max_transfer_bw            # peak table value
blended           = (1/num_packets)*peak + ((num_packets-1)/num_packets)*steady_state_bw
```

Tables (hardcoded in the device headers — the `tt_npe/data/device/layout/*.yaml`
files are **dead**, nothing reads them):

| packet_size | wormhole_b0 B/cyc | blackhole B/cyc |
|---|---|---|
| 128 | 5.5 | 6.0 |
| 256 | 10.1 | 12.1 |
| 512 | 18.0 | 24.2 |
| 1024 | 27.4 | 48.0 |
| 2048 | 30.0 | 57.7 |
| 4096 | — | 58.7 |
| 8192 | 30.0 | 60.4 |
| 16384 | — | 60.9 |

Source: `wormhole_b0.hpp:466-467`, `blackhole.hpp:914-915`. Link bandwidth is
**30 B/cyc** (WH, `wormhole_b0.hpp:236`) and **60.9 B/cyc** (BH, `blackhole.hpp:679`).

Injection / absorption rates (B/cyc) — these cap the table, since `min()` is taken:

| core type | WH inject | WH absorb | BH inject | BH absorb |
|---|---|---|---|---|
| WORKER / UNDEF | 28.1 | 28.1 | 60.9 | 60.9 |
| DRAM | 23.2 | 24.0 | 40.0 | 40.0 |
| ETH | 28.1 | 28.1 | 999.9 | 999.9 |

(`wormhole_b0.hpp:469-478`, `blackhole.hpp:917-926`)

### The `legacy` vs `latency_floor` single-packet trap

The blend gives **any 1-packet transfer the peak table bandwidth regardless of its
size**. On WH peak (30.0) exceeds the WORKER injection rate (28.1), so in the
default `legacy` mode a 1-packet transfer *always* runs at 28.1 B/cyc.

**Measured** (wormhole_b0, 1 packet, `cpt=32`):

| packet_size | legacy cycles | legacy B/cyc | latency_floor cycles | latency_floor B/cyc |
|---|---|---|---|---|
| 128 | 5 | 25.6 | 24 | 5.33 |
| 256 | 10 | 25.6 | 26 | 9.85 |
| 512 | 19 | 26.95 | 29 | 17.66 |
| 1024 | 37 | 27.68 | 38 | 26.95 |
| 2048 | 73 | 28.05 | 73 | 28.05 |
| 8192 | 292 | 28.05 | 292 | 28.05 |

Reading: `latency_floor` tracks the table exactly (`128/5.5 → 24`, `256/10.1 → 26`,
`512/18.0 → 29`); `legacy` charges everything at the injection rate
(`ceil(size/28.1)`). The legacy column's apparent sub-28.1 "B/cyc" values at small
sizes are just `ceil()` quantization, not a lower rate.

**Use `--single-packet-bw-model latency_floor` for small/atomic transfers.**
The default `legacy` is systematically optimistic for them.

## 3. Congestion — proportional sharing, no queues

Per timestep, each live transfer contributes its *requested* (pre-derate) rate to
every link on its route plus its source and sink NIU
(`wormhole_b0.hpp:75-128`):

```
effective_demand = ((end_timestep - max(start_timestep, start_cycle)) / cpt) * curr_bandwidth
link_demand_grid[link] += effective_demand     # for EVERY link in the route
```

Then a single multiplicative derate (`wormhole_b0.hpp:130-187`):

```
min_link_bw_derate = LINK_BANDWIDTH / max_link_demand_on_route
src_bw_derate      = injection_rate  / src_niu_demand
sink_bw_derate     = absorption_rate / sink_niu_demand
overall            = min(link, src, sink)
if (overall < 1) curr_bandwidth *= overall
```

`NUM_ITERS = 1`, `grad_fac = 1.0` — first-order only; the header states gradient
descent is unnecessary (`wormhole_b0.hpp:69-73`).

**Consequences:**

- Every flow over the same bottleneck gets the **same factor `C/D`** where `D` is
  summed demand. Equal-rate flows therefore split the link **equally** — this is
  demand-proportional (weighted) sharing, **not** max-min fair, not round-robin,
  not FIFO, no priority.
- **No derate at all until demand exceeds 100%.** Measured: a single flow at
  93.7% link demand → `congestion impact 0.0%`; the golden `example_wl.json` peaks
  at 99.7% demand → also `0.0%`.
- **Measured scaling** (N equal 8192 B flows sharing one link, WH):

  | N | link demand | est cycles | cong impact | cong-free |
  |---|---|---|---|---|
  | 1 | 93.7% | 292 | 0.0% | 292 |
  | 2 | 187.3% | 584 | 50.0% | 292 |
  | 3 | 281.0% | 877 | 66.7% | 292 |
  | 4 | 374.7% | 1171 | 75.1% | 292 |
  | 6 | 562.0% | 1760 | 83.4% | 292 |

  `est_cycles ≈ 292 × N` — the bottleneck is divided equally, and congestion impact
  saturates toward 100% as `1 − 1/N`.
- **There is no queueing delay.** Congestion costs throughput, not latency growth.
  Blackhole adds hand-tuned corrections on top (flow-diversity alphas
  0.024→0.014 over 4096 cycles, plus a backlog/pressure term with real
  cross-timestep state; `blackhole.hpp:98-624`) — these are empirical, not derived
  from a queueing model.
- **Known WH quirk:** the multicast sink derate is dead — `sink_demand` is
  initialized to `0` and then only ever `std::min`'d (`wormhole_b0.hpp:170-177`),
  so the derate is `+inf` and multicast writes never get sink-NIU derating.
  Blackhole does it correctly (`blackhole.hpp:605-618`). *Assumed* to be a latent
  bug; no comment or test covers it.

## 4. Latency is not modelled in the engine

Duration is `bytes / rate` only. There is **no per-hop latency and no pipeline
fill/drain**: the entire route is charged for the transfer's whole duration
(`wormhole_b0.hpp:118-128`), so a long route occupies its first hop as long as its
last.

Latency enters **only at trace-ingest time**, as a constant added to the start
offset (`npeWorkloadIngest.cpp:465-483`):

- WH read: 70 (same coord) / 154 (same col) / 170 (same row) / 270 (`wormhole_b0.hpp:423-433`)
- WH write: `40 + 10*hops` (`wormhole_b0.hpp:435-441`)
- BH read: 65 / 177 / 410 / 536 (`blackhole.hpp:870-880`)
- BH write: `40 + 11*hops` (`blackhole.hpp:882-888`)

**The simplified workload-JSON path adds no latency at all** — `phase_cycle_offset`
is taken verbatim (`npeWorkloadIngest.cpp:171-177`). Programmatically built
workloads have zero startup latency unless you encode it yourself.

## 5. Routing — dimension-order torus, one direction per NoC

`unicastRoute()` (`wormhole_b0.hpp:322-359`, `blackhole.hpp:765-802`):

- **NOC0**: east first (wrapping), then south (wrapping)
- **NOC1**: north first (wrapping), then west (wrapping)

Hop count `= modulo(dx−sx, cols) + modulo(dy−sy, rows)` (`wormhole_b0.hpp:406-421`).
Multicast is the **deduplicated union of unicast routes** over the rectangle
(`wormhole_b0.hpp:361-389`); all destinations are served concurrently at one rate.

DRAM and ETH cores are ordinary coordinates with different injection/absorption
rates — they are **not** routed differently. DRAM *controllers* exist in the model
but are used **only for reporting** (`npeStats.cpp:168-188`), never in the timing
loop, so two DRAM cores sharing a controller do **not** share a bandwidth pool.

## 6. Multichip — the fabric is a constant, not a model

There is **no ethernet link in the engine**. Cross-device traffic is decomposed
into per-device segments by trace ingest (`npeWorkloadIngest.cpp:519-640`), and the
inter-segment cost is a single constant (`npeEngine.cpp:360-394`):

```
ETH_HOP_CYCLE_DELAY_BASE     = 600
ETH_HOP_CYCLE_DELAY_PER_BYTE = 0.1055
checkpoint_delay = write_latency(src,dst,noc) + (600 + 0.1055 * packet_size)
```

applied only when the segment's source device differs from its parent's.
`getEthBandwidthPerLink()` (WH 12.5, BH 50 B/cyc) feeds **stats only**
(`npeStats.cpp:207`).

`tt_npe/data/fabric/topology/examples/*.json` is **not** read by the simulator —
`cfg.topology_json` is used solely to lay chips out for the visualizer
(`npeStats.cpp:409-431`). Real fabric routing lives in Python trace
post-processing (`tt_npe/py/util/fabric_post_process.py`).

## 7. Determinism and the "ideal" baseline

- **Fully deterministic.** No sampling, no RNG, no averaging of passes. The only
  `random` in the repo is in a synthetic-data generator script.
- `estimated_cong_free_cycles` comes from a **genuine second full simulation**
  with `congestion_model_name = "none"` (`npeEngine.cpp:417-441`) — not an
  analytical shortcut.
- `congestion impact = 100*(estimated − estimated_cong_free)/estimated`
  (`npeStats.cpp:887-893`).

## 8. Dependencies — heuristics, not semantics

There is **no barrier, semaphore, atomic, or kernel-phase construct** in the
engine. Two dependency sources are synthesized in `genDependencies()`
(`npeEngine.cpp:302-415`):

1. **Stride-2 per-source-NIU serialization** — transfers bucketed by
   `(noc_type, src.row, src.col, first_link_type)`, sorted by start cycle, and
   transfer `i` depends on `i-2` (`constexpr int stride = 2`, `:346`). The comment
   calls this "roughly approximate to 2-VC effects" (`:345`). So **at most 2
   outstanding transfers per source NIU**.
2. **Transfer-group chaining** for multichip segments, using the constant above.

**Phases are inert.** `npeWorkloadPhase` is just a transfer container
(`npeWorkload.hpp:76-94`); the engine "assume[s] all phases start at cycle 0"
(`npeEngine.cpp:267`) and the phase-unlock step is an unimplemented TODO (`:659`).
`phase_cycle_offset` is effectively an **absolute** cycle. Encode barriers
yourself via that offset.

## What this means for accuracy

1. Short/small transfers are **optimistic** under the default `legacy`
   single-packet model — switch to `latency_floor`.
2. Congestion reduces throughput but **never adds queueing latency** (except the
   empirical BH backlog term).
3. Route charging is whole-duration on every hop — no pipeline effects.
4. Multicast sink contention is unmodelled on WH.
5. Fabric cost is a constant, so multichip predictions are coarse.
6. The model is exact when uncongested: **measured** a single 8192 B worker→worker
   transfer predicted 292 cycles vs 291.5 ideal (`8192/28.1`).

## Reproducing

```bash
# 从 tt-loop-scheduler 仓库根目录执行：
./tools/npe_build.sh                 # 见 build-and-run-gotchas.md 为何需要包装器
cd thirdparty/tt-npe
export PYTHONPATH="$PWD/install/lib:$PWD/install/bin"
python3 ../../tools/npe_probe.py     # 上表所有测量数据的来源
```

See `build-and-run-gotchas.md` for why the `env -u` is mandatory, and
`metrics-semantics.md` for what the reported numbers actually mean.
