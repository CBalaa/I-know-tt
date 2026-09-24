# Wall clock registers on Tensix: `0x1F0` latches the high word into `0x1F8`, and that latch is per-tile shared

**Observed on**: tt-metal `4f9fa9e0`, Blackhole p150a (ARCH: blackhole, CHIP_FREQ 1350 MHz).
**Status**: register semantics are from the ISA spec; the *sharing* result is **measured**
(reproducible test + raw output in `tt-loop-scheduler/test/wall_clock_latch/`).

## The three registers

`RISCV_DEBUG_REGS_START_ADDR = 0xFFB1_2000` on tt-1xx (Blackhole). Three 32-bit MMIO
addresses expose one free-running 64-bit counter, with *different* read semantics:

| Address | tt-metal macro | Read behaviour |
| --- | --- | --- |
| `0xFFB1_21F0` | `RISCV_DEBUG_REG_WALL_CLOCK_0` / `WALL_CLOCK_L` | returns `counter & 0xffffffff`, **and latches `counter >> 32` into `counter_high_at`** |
| `0xFFB1_21F4` | `RISCV_DEBUG_REG_WALL_CLOCK_1` | returns the **live** `counter >> 32` |
| `0xFFB1_21F8` | `RISCV_DEBUG_REG_WALL_CLOCK_1_AT` / `WALL_CLOCK_H` | returns the **latched** `counter_high_at` |

`BlackholeA0/TensixTile/DebugTimestamper.md` is a one-line stub pointing at the Wormhole
tree and stating the behaviour is identical between the two architectures.

## The latch is per-tile, not per-RISC — measured

This is the non-obvious part, and it is **not** in the ISA doc (the doc only warns that the
low-then-latched-high sequence is safe "if there is only one agent simultaneously reading").

Test: BRISC reads `0x1F0` (latching high = H₀), then spins ~4.69e9 cycles without touching
any wall-clock register, then reads `0x1F8`. Because the high word only advances every
2³² ticks (~3.19 s at 1.348 GHz), the spin is what makes the experiment discriminating.

| NCRISC behaviour during BRISC's spin | BRISC's `0x1F8` | meaning |
| --- | --- | --- |
| idle | H₀ — **stale**, while `0x1F4` had advanced to H₀+1 | latch works, and it is BRISC's own |
| hammers `0x1F0` 3.6e8 times | H₀+1 — **NCRISC's latch, not BRISC's** | **the latch is shared tile-wide** |

So on a Tensix tile running BRISC + NCRISC + TRISC0-2 concurrently, the low-then-latched-high
sequence lets RISCs clobber each other's latch, and a reader can pick up **another RISC's**
high word. Re-reading `0x1F0` refreshes the latch to the current value.

## Why this matters for the device profiler

`tt_metal/tools/profiler/kernel_profiler.hpp` (`mark_time_at_index_inlined`, the code behind
`DeviceZoneScopedN`) reads `p_reg[WALL_CLOCK_HIGH_INDEX]` = `p_reg[1]` = **`0x1F4`** (the
*live* high) **first**, then `p_reg[WALL_CLOCK_LOW_INDEX]` = `0x1F0`. Reading the live high
first does dodge the shared latch above, but it is **not free** — see the cost note below.

### Reading `0x1F4` (live high) costs ~7 cycles more per read than `0x1F8` (latched high)

Measured on Blackhole p150a with an empty-`DeviceZoneScopedN` loop (slope fit over
N = 0/40/80/120 zones, all inside the 125-zone quota):

| `mark_time_at_index_inlined` read order | marginal cost / zone | recorded window of an empty zone |
| --- | --- | --- |
| `0x1F4` (live high) → `0x1F0` (low) — **upstream** | 51.0 cycles | 18 cycles |
| `0x1F0` (low, latches) → `0x1F8` (latched high) | **40.0 cycles** | **14 cycles** |
| `0x1F8` (latched high) → `0x1F0` (low) — *wrong semantics, cost control only* | 37.0 cycles | — |

Solving the three rows: the register swap alone (upstream vs control, same order) is
**14.0 cycles/zone = ~7 cycles per read** — a live snapshot of a free-running counter costs
real logic, a latched flop does not. The *order* is worth 3.1 cycles/zone in the other
direction (reading the low word first, with its latch write, lengthens the dependency chain).
Net: the upstream code pays **11 cycles/zone purely to read the live high**.

Trade-off of switching to low-then-latched-high: you lose immunity to cross-RISC latch
clobbering, but you gain the same ~1e-9-per-marker hazard you already had (a 2^32 wrap inside
a few-cycle window) *and* you remove the tear window entirely for single-reader use. Net: 21%
cheaper, no worse on correctness. Validated on the real 5-RISC matmul workload: 16,700 samples,
0 unnamed, 0 unpaired, no 2^32-scale outliers.

Do **not** change `WALL_CLOCK_HIGH_INDEX` itself — it is also used by the `quick_push` / flush
/ 64-bit-reassembly sites, which read high-first; pointing them at `0x1F8` would hand them a
stale latch. Add a separate index for `0x1F8` and change only `mark_time_at_index_inlined`.

These two do use the shared latch, and are therefore only safe under the ISA doc's
single-reader assumption:

- `tt_metal/hw/inc/internal/tt-1xx/risc_common.h:237` — `get_timestamp()`
- `tt_metal/hw/inc/internal/tt-1xx/blackhole/c_tensix_core.h:505` — `read_wall_clock()`

Both hazard windows are a few cycles out of every 3.19 s, so the practical failure rate is
~1e-9 per call — worth knowing, not worth panicking about.

## Timestamp width in the profiler is 44 bits, not 64

The profiler keeps `(high & 0xFFF) << 32 | low` — 12 + 32 bits. So the *marker* timestamp
wraps every 2⁴⁴ ticks. At the measured 1.348 GHz that is **≈ 3.6 hours**, not the ≈ 4.9 hours
you get if you assume a nominal 1 GHz. Sorting or subtracting marker timestamps must be done
modulo 2⁴⁴.

## Cross-references

- [`measuring-device-instrumentation-cost.md`](measuring-device-instrumentation-cost.md) —
  how to measure what a marker costs, why the recorded window and the marginal cost are
  different numbers, and **what the recorded window actually contains** (the two samples are
  taken by the same instruction, so the read's own latency cancels; only its 1-cycle issue
  slot counts).
- [`device-profiler-zone-budget.md`](device-profiler-zone-budget.md) — the 125-zone-per-RISC
  budget and the profiler's silent failure modes.

## Bonus: the wall clock ticks at the core clock

A 46,000,000-iteration spin loop of 100 `nop`s + loop overhead burned
4,692,204,077 / 4,692,204,090 / 4,692,204,092 ticks across runs — a spread of 15 ticks in
4.7e9. That is 102.004 cycles per iteration, i.e. 100 `nop` + ~2 cycles of loop overhead,
confirming 1 wall-clock tick = 1 core cycle.
