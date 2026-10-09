# DeviceZoneScopedN marker calibration must use the exact RISC and code shape

Observed on tt-metal `4f9fa9e0`, Blackhole p150a, 2026-10-07. The independent
`test/probe_calibration` kernel measured the current profiler implementation
(`mark_time_at_index_inlined`: live high `0x1F4`, then low `0x1F0`) on BRISC and
NCRISC.

For each RISC, mode 0 repeated an empty `DeviceZoneScopedN` for 0/40/80/120
iterations; mode 1 repeated the same compiler-shaped loop without a marker. A
least-squares slope difference identifies the complete marginal perturbation,
including constructor, start/end timestamp code, destructor, and surrounding
loop effects:

| RISC | marked slope | no-marker slope | net marker perturbation |
|---|---:|---:|---:|
| BRISC | 53.543 | 2.040 | **51.503 cycles/zone** |
| NCRISC | 49.600 | 2.035 | **47.565 cycles/zone** |

The CSV empty-zone windows in this exact binary were 21 cycles (BRISC) and 18
cycles (NCRISC), all 720 samples per RISC were constant. These values differ
from one another and from earlier harnesses (51/18), so a single old constant
cannot be reused across RISC roles or code shapes.

Mode 2 inserted independent low-word reads immediately before scope entry,
immediately before scope close (the operation-completion proxy), and immediately
after scope close. Pairing those reads with the profiler CSV gave:

| RISC | manual-before → START | operation-boundary → END (event phase) | END → manual-after |
|---|---:|---:|---:|
| BRISC | median 24 (24–28) | **median 14 (14–17)** | median 9 (9–12) |
| NCRISC | median 23 (22–29) | **median 11 (11–13)** | median 10 (10–14) |

The operation-boundary→END number is the meaningful event offset when an
operation finishes immediately before scope destruction. END→after is work after
the event (destructor bookkeeping and loop code), and must not be included in
that event offset. The before→START number is constructor/setup before the
START timestamp. It is not safe to infer a portable `a`/`b` split by subtracting
these values: register allocation, instruction scheduling, and code around the
scope alter the boundaries. Even this binary's mode-2 windows are 22 (BRISC)
and 19 (NCRISC), versus 21 and 18 for the empty loop.

The matmul probe also has a profiled-push counter branch around each candidate
zone. A matched helper loop with and without `DeviceZoneScopedN` measured net
costs of **53.020 cycles/event on BRISC** and **51.050 on NCRISC** (helper-only
slopes 9.049 and 9.055). These are the appropriate `C` values for a dense
prefix that contains that exact helper; the bare scope-loop values above omit
its branch and counter instructions.

The actual JIT ELF was inspected. In BRISC mode 2, the manual before read,
profiler start low read, operation-boundary read, profiler end low read, and
manual after read appear in order at disassembly addresses `0x5718`, `0x5750`,
`0x575c`, `0x5780`, and `0x5798` respectively. This confirms that the boundary
samples are where the calibration claims, rather than source-level assumptions.

Raw run output and the parser are in the consuming repository under
`test/probe_calibration/` (the test directory is ignored by default, so force
add it when committing). The full CSV remains under `/tmp` on the measurement
machine.
