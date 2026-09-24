# tt-npe workload formats — schema and traps

**Observed on:** `tt-npe` @ `0d2d4cc26575edae6e2729a4c2ac88f0ba17792e`.

One entry point, two formats (`npeWorkloadIngest.cpp:662-679`):

```cpp
if (is_tt_metal_trace_format) return convertNocTracesToNpeWorkload(...);
else { auto r = loadJSONWorkloadFormat(...);
       if (r) return r;
       log_warn("Failed to load workload file; fallback to parsing as tt-metal noc trace ...");
       return convertNocTracesToNpeWorkload(...); }
```

So the simplified parser is tried first and a **noc trace is the automatic fallback**.
Pass `-t` / `is_noc_trace_format=True` to force the trace path (required when the
top-level JSON is an array — the simplified parser errors with a hint to use `-t`,
`npeWorkloadIngest.cpp:60-67`).

## Simplified workload JSON

```jsonc
{
  "golden_result": { "cycles": 450 },   // optional -> golden_cycles[0] = {0, 450}
  "phases": [                            // REQUIRED array
    { "transfers": [                     // only key recognised inside a phase
      {
        "packet_size": 128,              // REQUIRED, bytes
        "num_packets": 1,                // REQUIRED, >= 1
        "src_x": 1, "src_y": 1,          // REQUIRED  (x = col, y = row)
        "dst_x": 1, "dst_y": 1,          // unicast
        // OR multicast:
        // "mcast_start_x": 3, "mcast_start_y": 3,
        // "mcast_end_x":   5, "mcast_end_y":   5,
        "device_id": 0,                  // optional, default 0 (separate for src and dst)
        "injection_rate": 0,             // optional — DEAD by default, see below
        "phase_cycle_offset": 219,       // optional, earliest start cycle
        "noc_type": "NOC_0",             // REQUIRED
        "noc_event_type": ""             // optional, free-form
      }
    ]}
  ]
}
```

Parsed at `npeWorkloadIngest.cpp:71-206`.

### Traps

1. **`injection_rate` in the JSON is dead by default.** `npeAPI::preprocessWorkload`
   calls `wl.inferInjectionRates()` which **unconditionally overwrites** every
   transfer's rate from its source core type (`npeAPI.cpp:57-68`,
   `npeWorkload.cpp:145-152`). `cfg.infer_injection_rate_from_src` defaults to `true`
   (`npeConfig.hpp:53`). Pass `--no-injection-rate-inference` to make your value stick.

2. **Unicast vs multicast is inferred from absence**, not a flag:
   `if (dst_x == -1 && dst_y == -1)` (`:114-126`). The `&&` matters — supplying only
   one of the two leaves the other at `-1` and validation fails as an out-of-range coord.

3. **`x` is the column and `y` is the row, and ingest swaps them** into
   `Coord{device_id, row, col}` (`:196-197`, with the comment
   `// note: row is y position, col is x position!`). `npe.Coord` in Python is
   `(device_id, row, col)` — **not** an (x, y) pair.

4. **`noc_type` is a binary switch, not a validated enum**:
   `(noc_type == "NOC_0") ? NOC0 : NOC1` (`:201`). A typo silently becomes NOC1.

5. **No `type` field and no dependency field.** The only type-ish field is the
   free-form `noc_event_type`, which matters in exactly three places: multicast-write
   util accounting (`wormhole_b0.hpp:121`), timeline `fabric_event_type`
   (`npeStats.cpp:573`), and the experimental response scheduler (`npeEngine.cpp:75-77`).

6. **`phase_cycle_offset` is effectively an absolute cycle**, not phase-relative — the
   engine assumes all phases start at cycle 0 (`npeEngine.cpp:267`). See
   `prediction-model.md` §8: phases are inert, there are no barriers.

7. **No latency is added on this path.** The trace path injects read/write latency
   into the start offset; the simplified JSON path takes `phase_cycle_offset` verbatim
   (`:171-177`). Encode your own startup latency.

8. **Multicast is a rectangle** and `mcast_start_*` pairs with `src_device_id`,
   `mcast_end_*` with `dst_device_id` (`:159-161`).

### NOC1 multicast asymmetry

In the **trace** path, NOC1 multicast start/end are **reversed** at ingest
(`npeWorkloadIngest.cpp:503-514`, `// NOTE: noc_dest coord are reversed for NOC1`).
In the **simplified JSON** path they are **not** (`:159-161`). Authoring NOC1
multicast workloads by hand will therefore disagree with equivalent traces.

## Coordinates and routing

- **All coords are physical `(row, col)`, never logical** —
  `npeDeviceTypes.hpp:27`: `// note: all coords here are physical, NOT logical!`
  No logical→physical table exists; callers must emit physical coords.
- Grids: wormhole_b0 **12 rows × 10 cols** (`wormhole_b0.hpp:447-448`),
  blackhole **12 × 17** (`blackhole.hpp:894-895`).
- wormhole_b0 core types: DRAM at **cols 0 and 5**; ETH at **row 0** cols 1-4 and 6-9;
  everything else WORKER. Pinned by `test_npe_device.cpp:32-46`.
- **NOC0** = east then south (wrapping); **NOC1** = north then west (wrapping).
  Hop count `= modulo(dx−sx, cols) + modulo(dy−sy, rows)`.
  Pinned exactly by `test_npe_workload.cpp:137-158` (e.g. `(9,1)→(1,1)` on NOC0 = 2 hops
  because of the wrap).

## Validation

`npeWorkload.cpp:15-74`. All of: `num_packets > 0`, `packet_size > 0`, src in bounds,
dst in bounds (multicast: any rectangle corner in bounds), `phase_cycle_offset >= 0`,
src/dst device ids match. Failure returns `WORKLOAD_VALIDATION_FAILED` **without
running** the simulation. Error output is capped at 50 messages verbose / 5 otherwise.

`cfg.remove_localized_unicast_transfers` (default false) drops unicast transfers whose
`(src.row/2 == dst.row/2) && (src.col/2 == dst.col/2)` (`npeWorkload.cpp:170-171`);
multicast is always retained.

## Noc trace format (the tt-metal path)

Flat JSON array of noc events; fields consumed at `npeWorkloadIngest.cpp:303-660`:
`proc`, `type`, `num_bytes`, `sx`, `sy`, `dx`, `dy`, `src_device_id`, `dst_device_id`,
`timestamp`, `noc`, `mcast_*`, `zone`, `zone_phase`, plus a nested `fabric_send.path[]`.
Documented in `tt_npe/doc/noc_trace_format.md`.

Ingest does four notable things:

1. `t0` = min timestamp; per-device golden window = min/max kernel cycles minus a flat
   **20-cycle** correction for kernel-end overhead (`:263-265`).
2. **READ src/dst are swapped** so data always flows src→dst (`:448-451`).
3. **Read/write latency is added to the start offset** (`:459-484`) — WH read
   70/154/170/270, WH write `40 + 10*hops`.
4. Everything lands in **one phase**; multi-device transfers are split into
   `transfer_group`-linked per-device segments (`:520-640`).

**Resolved (was: unresolved).** The accepted-event set contains `"WRITE_"` with a
trailing underscore (`:321`) while `doc/noc_trace_format.md:60-71` documents bare
`WRITE`. This looked like a potential silent-drop bug.

**A real capture settles it: actual traces emit `WRITE_`.** Measured on tt-metal
`4f9fa9e0` (blackhole p150a), `example/standalone/matmul_multi_core/`: the trace contains
`READ: 14322`, `WRITE_: 360` — and tt-npe ingests all 360 write events, producing 14 682
timeline transfers. So code and data agree; it is the **doc** that is wrong, not the
parser.

If you author a trace by hand, use `WRITE_`. A bare `WRITE` would be silently skipped
(`:403-405`).

## Device names accepted

`npeDeviceModelFactory.hpp:19-50`: `wormhole_b0, N150, N300, T3K, TG, GALAXY,
blackhole, P100, P150, P300, P150_X8, BLACKHOLE_GALAXY`.
The **Python CLI's `-d` choices omit** `TG, GALAXY, BLACKHOLE_GALAXY, P300, P150_X8`
even though the C++ factory supports them — use the Python API for those.
