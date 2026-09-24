# NoC traces can contain hour-stale events from untouched cores' profiler buffers

**Finding.** A `--collect-noc-traces` capture can silently contain a **second, stale
cluster of events** whose device timestamps are ~1 hour older than the run you just
did. The stale cores are ones the program never launched, so their L1 profiler buffers
were never cleared and still held a *previous* session's data, which this run's drain
happily collected.

This is invisible in the trace itself — same `run_host_id`, same `op_name`, well-formed
events — and it **breaks tt-npe completely**.

Observed on tt-metal `4f9fa9e0`, arch **blackhole** (p150a), `CHIP_FREQ[MHz]: 1350`,
`TT-KMD 2.7.0-rc1`.

## Symptom

`npe_analyze_noc_trace_dir.py` finishes "Trace merging complete" and then prints an
**empty summary table**, followed by:

```
ZeroDivisionError: division by zero
  ... getAvgError(): sum(...) / len(self.datapoints.values())
```

The `ZeroDivisionError` is a second-order bug (`getAvgError` has no empty guard), but the
*real* signal is **zero datapoints**: every per-trace `runNPE` failed, and
`process_trace` swallows the exception into a `log_error` that the progress-bar
redraw (`\x1b[1A\x1b[2K`) overwrites. Running the same trace directly through the Python
API is what surfaces the truth — and it **hangs** instead of erroring.

## Why it hangs

tt-npe sets `t0 = min(timestamp)` over the whole trace
(`tt_npe/cpp/src/npeWorkloadIngest.cpp:224-225`) and every transfer's start offset is
`ts - t0`. One stale event ~4.9e12 cycles in the past therefore makes the golden window
~4.9e12 cycles wide. At the default `cycles_per_timestep = 32` the engine must step
~1.5e11 timesteps — it will not finish in any practical time, and it reports nothing
while doing so.

## How to detect it

Compare the `BRISC-KERNEL` `ZONE_START` timestamps across cores. A healthy single kernel
launch is **tightly synchronized**; stale entries sit in a separate cluster with a huge
gap:

```python
starts = sorted(...)                    # BRISC-KERNEL ZONE_START per core
gaps = [(starts[i+1]-starts[i], i) for i in range(len(starts)-1)]
max(gaps)                               # the stale/current split
```

Measured on this workload:

| cluster | cores | `ZONE_START` | launch spread |
|---|---|---|---|
| current | 100 | 6175231382329 … 6175231382589 | **260 cycles (0.19 µs)** |
| stale | 10 | ~1262908485xxx | ~1 hour earlier |

The current cluster's 260-cycle spread across 100 cores is the signature of one real
launch. Anything ~1e12 cycles away is not part of this run.

In this case the stale cores were exactly `core_x = 11, core_y = 2..11` — a column the
matmul grid (`cols 1-6, 12-15`) never uses.

## Workaround

Filter the trace before handing it to tt-npe — drop events older than the gap. The
surviving trace then spans a sane window (here 210772 cycles ≈ 156 µs) and tt-npe runs
in **16.6 s** instead of hanging.

```python
GAP = 2.4e12
kept = [e for e in trace if not (e.get('timestamp') and e['timestamp'] < GAP)]
```

Note the `generated/profiler/.logs/profile_log_device.csv` from the same run shows the
same two clusters, so it is a good cross-check — and its `time[cycles since reset]`
column plus `CHIP_FREQ[MHz]` is what lets you convert to µs.

## Not yet determined

*Why* only those 10 cores retained data. They were untouched by this program, which is
consistent with "buffer not cleared because no kernel ran there", but I did not verify
whether tt-metal intends to clear all profiler buffers at device init. Treat the
detection recipe as the reliable part, not the mechanism.
