# Device profiler zones: 125-zone-per-RISC budget, and three silent failure modes

**Finding.** With the DRAM device profiler (`TT_METAL_DEVICE_PROFILER=1`), each RISC can
record at most **125 user zones per kernel run**. Past that, zones are dropped
**silently** — no device flag, no host warning, and the surviving rows look perfectly
well-formed (every `ZONE_START` has its matching `ZONE_END`). An over-instrumented kernel
therefore produces plausible, complete-looking, *biased* data.

Observed on tt-metal `4f9fa9e0`, arch **blackhole** (p150a), `CHIP_FREQ[MHz]: 1350`.

## The budget, and why it is 125

`tt_metal/hw/inc/hostdev/profiler_common.h:164-174`:

```c
PROFILER_L1_MARKER_UINT32_SIZE      = 2
PROFILER_L1_GUARANTEED_MARKER_COUNT = 4     // consumed by the firmware FW/KERNEL zones
PROFILER_L1_PROGRAM_ID_COUNT        = 2
PROFILER_L1_OPTIONAL_MARKER_COUNT   = 250   // the user budget, counted in markers
PROFILER_L1_VECTOR_SIZE = (250 + 4 + 2) * 2 = 512 uint32 words
```

One zone costs 2 markers (`ZONE_START` + `ZONE_END`), so 250 optional markers =
**125 zones per RISC per kernel run**.

## Why the drop is silent

`tt_metal/tools/profiler/kernel_profiler.hpp:201-213`: `bufferHasRoom()` returns false
once `wIndex` would overrun, and the `profileScope` constructor (`:715-747`) then records
nothing at all. It does **not** set `DROPPED_ZONES`.
`mark_dropped_timestamps` / `DROPPED_ZONES` (`:453`, `:584`, `:588`, `:688`) is only
reached from the L1→DRAM push path, so the host's
`"Profiler DRAM buffers were full, markers were dropped!"` warning
(`tt_metal/impl/profiler/profiler.cpp:1703-1705`) does **not** fire for an L1 overflow.

## Measured evidence

A reader kernel that genuinely needed 640 zones per core (80 loop iterations × 8 zones)
produced exactly **254 records** per NCRISC in `.logs/profile_log_device.csv`:

```
254 = 4 guaranteed (NCRISC-FW + NCRISC-KERNEL) + 250 optional
250 optional markers = 125 zones = 15 full iterations x 8 zones + 5 zones of the 16th
```

That decomposition reproduces the observed per-zone sample counts exactly, so the
mechanism is proven rather than inferred. Only **19.5%** of the intended data was
captured, and the captured slice was the **first** 15 iterations (head-truncated) — i.e.
the pipeline fill phase, not steady state.

## Marker cost, measured

Measured on the DRAM backend by timing a loop of N empty zones at one call site and taking
the slope (p150a, tt-metal `4f9fa9e0`):

| marker | cost |
| --- | --- |
| empty `DeviceZoneScopedN`, recorded | **51 cycles/zone** (slope over N=0..125; 56.6 in a realistic kernel shape) |
| empty zone once the buffer is full (dropped) | **12 cycles/zone** (re-measured 12.21) |
| `DeviceRecordEvent` | +22 cycles |
| `DeviceTimestampedData` | +28 cycles |
| `DeviceZoneScopedNIf(name, false)` | +50 — the `active` arg is ignored (see below) |

A zone is therefore ~2.3x the cost of a point event, and 125 zones x 51 cycles ~= 6.4k
cycles of added latency per RISC — not negligible against a ~167k-cycle kernel.

⚠️ **The 51 is not a property of the zone — it is a property of which wall-clock register the
profiler reads.** `mark_time_at_index_inlined` reads the *live* high word at `0x1F4`, which
costs ~4–6 cycles more per read than the *latched* high word at `0x1F8`. Reordering the two
reads to `0x1F0` (low, which latches) → `0x1F8` (latched high) drops the marginal cost to
**40 cycles/zone** and the recorded window to **14 cycles**, with no loss of accuracy. Full
derivation, the three-way decomposition and the caveats:
[`measuring-device-instrumentation-cost.md`](measuring-device-instrumentation-cost.md) and
[`wall-clock-registers-and-latch.md`](wall-clock-registers-and-latch.md).

Also note the 51 in the first row and the ~12 in the second row are **not** measured the same
way: the first is a *slope* over recorded zones, the second is the slope once the budget is
exhausted. Mixing them in one fit produces a meaningless average.

The clock was measured, not assumed: 4.0e9 ticks / 2.9669 s = **1.348 GHz**, and a 100-deep
dependent-`addi` chain measured **1.0200 ticks/instruction** (= 102/100), i.e. single-issue,
so **1 tick = 1 core cycle ~= 0.741 ns** and the CSV's `time[cycles since reset]` column can
be read as core cycles directly. A later, tighter bound from a deterministic spin loop puts
the tick rate at **>= 1349.3 MHz**, consistent with the `CHIP_FREQ[MHz]: 1350` in the CSV
header — so 1.348 GHz is ~0.1% low but the right magnitude.

There is also a fixed **per-launch** tax, separate from the per-zone cost: firmware init plus
BRISC pushing every RISC's L1 buffer to DRAM. Measured from the CSV: `BRISC-FW` ~=
**7136 cycles** (~5.3 us), `NCRISC-FW` ~= 988 cycles, per program launch.

**Which number is for what:** 51 cycles/zone is the *marginal total cost* — use it to reason
about how much instrumentation perturbs the program. It is NOT the number to subtract from an
op's measured duration. The recorded window (`ZONE_END.time - ZONE_START.time`) contains the
START marker write plus the op but not the timestamp reads, so the in-window overhead is
smaller than 51. **The exact floor is the recorded duration of an empty zone: 18 cycles**
(upstream read order; 14 with the `0x1F8` reorder). That is the number to subtract — measure
it, do not reuse the slope. It is not noise: it decomposes into issue slots plus the stall
from the high-word MMIO read inside the window — see
[`measuring-device-instrumentation-cost.md`](measuring-device-instrumentation-cost.md#what-the-recorded-window-actually-contains).

## What does NOT lift the budget

- **The streaming profiler** (`TT_METAL_STREAMING_PROFILER=1`) would be the natural fix, but
  it **does not work on this card**. `streaming_profiler_device.cpp:220` requires
  `hal.has_programmable_core_type(HalProgrammableCoreType::DRAM)`, and this FW bundle does
  not expose the DRISC relay, so `boot()` returns {} and every producer stays unarmed:
  `[streaming profiler] no DRAM programmable cores (card FW below the DRISC gate?);
  producers stay unarmed and markers are DROPPED`. The subscriber saw 0 zones and
  `TT_METAL_STREAMING_PROFILER_ZONE_CSV` was never created. Its `--bench` numbers (e.g.
  177.7 cycles for an empty zone) are the *streaming* producer path measured with the relay
  disarmed — do not use them for the DRAM profiler, which is 51.

- **`TT_METAL_PROFILER_ACCUMULATE=1`** — measured, no help for sub-zones; the kernel was
  still capped at ~246 rows/core. Accumulate mode changes main-scope
  (per-program-iteration) handling, not the within-kernel marker budget.
- **Flushing L1→DRAM mid-kernel** — not possible from a kernel. `finish_profiler`
  (`kernel_profiler.hpp:376-470`) only pushes at main-scope end, and the comment at `:449`
  states that in normal profiling nothing services `signal_host_buffer_full`, so it drops
  rather than stalling.
- **`DeviceZoneScopedSumN1/N2`** — `SUM_COUNT = 2` (`profiler_common.h:18`), so there are
  only 2 accumulating slots per RISC. Not enough to accumulate many ops.

## What to do instead

Sample a bounded window: keep instrumented zones per RISC per run well under 125, put the
arithmetic in a comment, and add a `static_assert` on
`PROFILER_L1_OPTIONAL_MARKER_COUNT` so a future tt-metal change fails the build instead of
silently truncating. Always report **per-core sample counts** and compare them against the
expected op count — that is the only way to prove nothing was dropped.

## Trap: `DeviceZoneScopedNIf`'s `active` argument is ignored by the DRAM profiler

`kernel_profiler.hpp:1145-1147`:

```c
#ifndef DeviceZoneScopedNIf
#define DeviceZoneScopedNIf(name, active) DeviceZoneScopedN(name)
#endif
```

The `active` predicate is **discarded**. Only the streaming profiler
(`kernel_profiler_streaming.hpp:514`) honours it. With the DRAM profiler, conditional
zones need an explicit `if` in kernel code.

## Trap: a multi-line zone invocation loses its name, silently

A zone's identity is a 16-bit hash of `"<name>,<__FILE__>,<__LINE__>"`, and the host
recovers the readable name from the compiler's `#pragma message` (`PROFILER_MSG_NAME`).
For a **multi-line** macro invocation clang resolves `__LINE__` for the hash to the line
the invocation **starts** on, while the `#pragma message` is emitted with the line it
**ends** on — so the hash matches no name.

The kernel still compiles, runs, and records timestamps; the CSV row just has an **empty
zone-name column and source line 0**. No error, no warning.

Measured: a two-line invocation put line 117 in the pragma while the device emitted
`hash(..., mm.cpp, 116, ...)` → exactly 1600 unnamed rows. Collapsing every invocation to
a single physical line → 0 unnamed rows.

So `{ TRACY_ZONE("name"); op; }` must be on **one physical line**. A helper macro's
*definition* may span lines; only its call sites must be single-line. Verify with
`count(CSV rows with an empty zone-name column) == 0`.

## Readout notes

- Artifacts dir: `$TT_METAL_PROFILER_DIR` if set, else `$TT_METAL_HOME/generated/profiler/`,
  else `./generated/profiler/` relative to CWD
  (`tt_metal/impl/profiler/profiler_paths.hpp:21`). Device log:
  `.logs/profile_log_device.csv` (`DEVICE_SIDE_LOG`).
- Drained automatically — `ReadDeviceProfilerResults` is called from the
  `WaitProgramDone` / `EnqueueProgram` path (`tt_metal/impl/host_api/tt_metal.cpp:1095`,
  `:1105`).
- The timestamp column is `time[cycles since reset]`, and the CSV's first line carries
  `CHIP_FREQ[MHz]`, so on p150a 1 cycle ≈ 0.741 ns.
- Reading the CSV does **not** require host-side Tracy, and neither does getting device
  zones into a Tracy *timeline*: `pushTracyDeviceResults` lives in `libtt_metal.so`, which the
  build tree compiles with `ENABLE_TRACY=ON`, so the `#if defined(TRACY_ENABLE)` guards in
  `tt_metal/impl/profiler/profiler.cpp` (e.g. `:2697`) are satisfied **in the library**.
  Measured: a host program NOT built with TRACY_ENABLE, run under `tools/tracy_capture.sh`,
  produced a 179 KB trace with 1618 zones, and the device zone names showed up in
  `tracy-csvexport`. TRACY_ENABLE on *your* program is only needed for your own host-side
  zones (and then the Tracy token-fingerprint trap applies).
- Conversely, `TT_METAL_DEVICE_PROFILER=1` `TT_FATAL`s unless `libtt_metal.so` was built with
  TRACY_ENABLE (`tt_metal/llrt/rtoptions.cpp:967`) — a property of the library, not of your
  program.
- A profiled run reports `JIT cache stats: 0/N hits`, which is a useful signal that kernels
  were genuinely rebuilt with `-DPROFILE_KERNEL` rather than silently compiled out.

## Trap: a DPRINT-based budget warning can never fire during a profiled run

`DPRINT` is structurally unavailable exactly when it is needed.
`tt_metal/llrt/rtoptions.cpp:397-399`:

```c
TT_FATAL(
    !(get_feature_enabled(RunTimeDebugFeatureDprint) && get_profiler_enabled()),
    "Cannot enable both debug printing and profiling");
```

Debug printing and the DRAM profiler are mutually exclusive at `MetalContext`
construction. With the profiler *off*, `PROFILE_KERNEL` is undefined, so
zone-accounting code guarded on it compiles out anyway — a DPRINT self-check is
dead in **both** configurations. Use `ASSERT` for the runtime half.

## Trap: `ASSERT` is a no-op unless the watcher is enabled

`tt_metal/hw/inc/internal/debug/assert_common.h:19-43` has three branches; the
default build takes the third:

```c
#if defined(WATCHER_ENABLED) && !defined(WATCHER_DISABLE_ASSERT) && !defined(FORCE_WATCHER_OFF)
#define ASSERT(condition, ...) (void(not(condition) ? assert_and_hang(__LINE__, ##__VA_ARGS__), 0 : 0))
#elif defined(LIGHTWEIGHT_KERNEL_ASSERTS)
#define ASSERT(condition, ...) (void(not(condition) ? lightweight_assert_trap(), 0 : 0))
#else
#define ASSERT(condition, ...) (void(sizeof(decltype(not(condition)))))  /* no-op, ASSERT_ENABLED 0 */
#endif
```

So a kernel self-check is only loud under `TT_METAL_WATCHER=1` (or
`LIGHTWEIGHT_KERNEL_ASSERTS`). The watcher and the DRAM profiler are *not*
mutually exclusive, so `TT_METAL_WATCHER=1 TT_METAL_DEVICE_PROFILER=1` is valid.

Signature trap: it is `assert_and_hang(uint32_t line_num, debug_assert_type_t)`,
so the variadic arguments are a `debug_assert_type_t`, **not** a printf-style
message. `ASSERT(cond, "text")` does not compile. Write `ASSERT(condition)` and
put the explanation in a comment.

Corollary: a **compile-time `static_assert` is the only always-on guard**. Layer
it with the runtime `ASSERT` and with host-side per-core sample-count checking.

## Working pattern for conditional zones under the DRAM profiler

Because `DeviceZoneScopedNIf` ignores `active` and `DPRINT` is unavailable, a
bounded window needs a local helper macro. The op must appear exactly once at
the call site, and **every invocation must be one physical line**:

```c
#if defined(PROFILE_KERNEL) && ENABLE_TRACY
#define TRACY_ZONE_IF(cond, name, ...) \
    do { if (cond) { TRACY_ZONE(name); __VA_ARGS__; } else { __VA_ARGS__; } } while (0)
#else
#define TRACY_ZONE_IF(cond, name, ...) do { __VA_ARGS__; } while (0)
#endif

// clang-format off   <- a formatter re-wrapping these silently empties the names
TRACY_ZONE_IF(READER_INSTRUMENT_KT(k, Kt), "reader::cb_reserve_back::in0", cb_reserve_back(cb_id_in0, 1));
// clang-format on
```

The `if/else` shape is required: the zone object must be constructed *and*
destroyed around the op, so the op text appears in both branches — but it is
written once at the call site, so the instrumented and uninstrumented paths
cannot drift. With the profiler off the macro expands to the bare statement (no
branch, no counter).

Wrap the call sites in `// clang-format off` / `// clang-format on`: these lines
exceed `ColumnLimit: 120`, and a formatter would otherwise re-wrap them straight
into the silent-name-loss failure mode above.

Measured (reader, window = first and last `kt` of every output tile, 2 of 20):
64 zones per core against the 125 budget, **0 unnamed rows**, and exactly
`70 cores x 8 + 40 cores x 6 = 800` samples per reader zone — the grid splits
400 output tiles as 70 cores x 4 + 40 cores x 3, so the expected count is fully
predicted in advance. Wrapping 6 of the 8 invocations produced exactly 9600
unnamed rows = 6 zones x 800 samples x 2 markers, so the sample-count arithmetic
and the name-loss arithmetic agree independently.

## A bounded window can hide the stall it was meant to find

Sampling only the first and last `kt` of each output tile is affordable, but it
is **not phase-neutral**. Measured on the same reader (M = N = K = 640, Kt = 20,
4 output tiles per core, 2-deep CBs, kernel = 166560 cycles on NCRISC):

The 8 windowed samples of one core are spread across the **whole** kernel — the
last `cb_reserve_back` sample starts at 164880 of 166560. The reader is in
lockstep with the program, so it never finishes early. Reading tile boundaries
off the sample order, the gap between the `kt=0` and `kt=19` samples implies
**1400-2970 cycles per iteration** for the 19 unsampled iterations in between,
versus ~670 cycles for the sampled boundary iterations. The window looked at the
cheap iterations.

Fix: a **single-site diagnostic mode**. One zone on every `kt` costs
1 x 20 kt x 4 tiles = **80 zones/core**, still inside the 125 budget, so any one
op can be profiled on every iteration. Instrumenting one site at a time is the
whole trick — two sites on every kt would cost 160 and not fit.

Result for this reader, per-iteration `cb_reserve_back::in0` on **every** kt,
all 4 tiles: **46-53 cycles, flat**, p50 48-51, max 53. Total ~1000 cycles per
tile. So there is **no circular-buffer backpressure at all** — the reader never
waits for CB space, even with 2-deep CBs.

The stall is entirely in `noc_async_read_barrier`. With both read barriers
(sites in0 + in1) instrumented on `kt < 13` (2 x 13 x 4 = 104 zones/core), the
barrier pair accounts for **92% / 88% / 86% / 84%** of the iteration time on
tiles 0/1/2/3, and the p50 barrier pair converges from ~3500 cycles/iteration on
tile 0 to ~1150 by tile 3. That matches the tile-gap arithmetic independently.

Generalisable lesson: a bounded window must be justified against the *stall
structure*, not just the budget. Cheap to check — look at whether the sampled
timestamps span the whole kernel, and compare the gap between consecutive
samples against the sum of the measured ops. If the gap is much larger, the
window is looking in the wrong place.

## Name recovery only reads pragma lines containing `KERNEL_PROFILER`

Zone-name resolution is a literal **token grep** over the JIT's compiler logs, not a parse of
all diagnostics (`tt_metal/jit_build/build.cpp:960`):

```c
auto cmd = fmt::format("grep KERNEL_PROFILER {}*.o.log", out_dir);
```

The hits are appended to `new_zone_src_locations.log`, which the host reads to map
hash -> name. So a `#pragma message` **you** add is inert as long as its text does not
contain the token `KERNEL_PROFILER`: it never enters the name table and cannot collide
with a zone hash. That is what makes a self-announcing diagnostic build safe.

## Verifying a source-level zone switch without a device run

`ENABLE_TRACY`-style switches are `#ifndef`-guarded so the default lives in the header,
and the JIT passes no custom `-D`. But `#ifndef` cuts both ways: you can override the
switch on a **replayed** compile command, with no device and no `flock`:

```bash
# lift the kernel's own compile line out of a previous run's log, then
# drop `-include <pch>`, swap `-c -o <obj>` for `-E`, retarget the TU, add the -D
riscv-tt-elf-g++ <flags from log> -DPROFILE_KERNEL=1 -DENABLE_TRACY=0 -E reader.cpp
```

Measured on the `matmul_multi_core` reader/writer (same version as above): with
`ENABLE_TRACY=1` the preprocessed body holds the 8 `reader::` and 4 `writer::` names
plus their `KERNEL_PROFILER` pragmas; with `=0` it holds **0 of either**, and each op
collapses to a bare statement:

```c
do { cb_reserve_back(cb_id_in0, 1); } while (0);
```

That is the cheap way to prove a zone switch really compiles out *before* spending a
device run on it.
