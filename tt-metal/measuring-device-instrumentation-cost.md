# Measuring what device instrumentation actually costs

**Finding.** A device marker has **two different costs** and they are routinely confused:

- the **recorded window** — `ZONE_END.time - ZONE_START.time` in the CSV. This is what you
  *subtract* to recover a real op cost.
- the **marginal cost** — how much slower the kernel actually runs. This is what you use to
  reason about perturbation.

For an empty `DeviceZoneScopedN` on Blackhole p150a they are **14 vs 40 cycles** (upstream
tt-metal `4f9fa9e0` read order: **18 vs 51**). Neither number is derivable from the other by
inspection — measure both.

Three independent methods agree, which is how you know the numbers are real rather than an
artefact of one harness. All three are implemented in the consuming repo under
`test/zone_cost/` (slope) and `test/zone_cost/decompose.py` (gap).

## Method 1 — slope over N zones (gives the marginal cost)

One call site, a loop of N empty zones, time the loop with the wall clock, fit
`loop_cycles = a*N + b`. **`a` is the marginal cost**; `b` absorbs the loop's own overhead.
Do not divide total by N — the fixed overhead makes that drift (51.65 → 51.35 → 51.22 as N
goes 40 → 80 → 120).

**Keep N ≤ 125.** Past the per-RISC marker budget the extra zones are dropped at a
*different* cost, and the fit becomes a meaningless blend of two populations.

## Method 2 — gaps between consecutive CSV samples (gives the split)

The profiler writes **two** timestamps per zone, so a run of zones at one call site gives you
the internal boundaries for free. Split the empty-zone code into:

```
X = constructor before the first wall-clock read   (bufferHasRoom guard + bookkeeping)
Y = between the two wall-clock reads               (== the recorded window)
Z = destructor after the second read               (+ the loop's own i++/compare/branch)

X1────Y1────Z1─X2────Y2────Z2─X3────Y3────Z3
   a      b       c      d       e      f
```

| equation | expression | meaning |
| --- | --- | --- |
| 1 | `c - b = start₂ - end₁` | **X + Z** |
| 2 | `e - b = start₃ - end₁` | **2(X+Z) + Y** |

Measured, patched read order, 120 zones:

```
Y = 14   (120/120 samples exactly 14)
X + Z = 26
e - b = 66 = 2*26 + 14   ✅
X + Y + Z = 40           ✅ agrees with the slope method's 40.04
```

Upstream read order, same harness: `Y = 18`, `X + Z = 33`, `e-b = 84 = 2*33+18 ✅`,
`X+Y+Z = 51` vs slope `51.02`.

**Equation 2 is not independent.** `e - b = Z₁+X₂+Y₂+Z₂+X₃ = 2(X+Z) + Y`, and equation 1
already gives `X+Z`. So equation 2 is a *consistency check*: two timestamps per zone cannot
resolve three unknowns. You get `Y` and `X+Z`, never `X` and `Z` separately.

## Method 3 — the dropped zone (gives the guard, and a third handle)

When `bufferHasRoom()` fails, control jumps straight to the end: only the **guard**
(`lui/lw/lui/lw/li/sub/bleu`) runs — `addi/sw/li`, all of Y and all of Z are skipped. So
timing a loop that overruns the budget measures *guard + loop overhead*:

```
N =   0  ->    32 cycles
N = 125  ->  5033 cycles     (5033-  32)/125 = 40.01  cycles/zone, all recorded
N = 250  ->  6559 cycles     (6559-5033)/125 = 12.21  cycles/zone, all dropped
```

12.21 matches the ~12 cycles/zone for a dropped marker reported elsewhere. Note this path
contains **no** wall-clock reads, so the number is independent of the register choice below.

## Putting it together

With `X + Z = 26` and `guard + loop = 12.21`, plus `X = guard + c₃` (the three instructions
after the branch) and `Z = Z_dtor + loop`, you get `Z_dtor = 26 - 12.21 - c₃`. Taking the loop
overhead as the usual 2–3 cycles:

| | X | Y | Z | total |
| --- | --- | --- | --- | --- |
| loop = 2 | ≈ 13.2 | **14** | ≈ 12.8 | **40.0** |
| loop = 3 | ≈ 12.2 | **14** | ≈ 13.8 | **40.0** |

So the marginal cost splits roughly into thirds. `Y` and `X+Z` and `guard+loop` are all
*measured*; the X/Z split is an *estimate* — the profiler's own timestamps cannot supply the
third boundary. (To measure it properly, add a probe that reads the wall clock at the end of
X.)

## What the recorded window actually contains

The window is `ZONE_END.time - ZONE_START.time`, i.e. the **difference of two samples taken by
the same instruction at the same address**. Any fixed issue→sample offset therefore
**cancels**, and the window equals the *issue-to-issue* distance between the two reads. The
endpoint read contributes only its 1-cycle issue slot, not its latency.

Measured directly: two strictly adjacent `lw 0x1F0` (verified adjacent in the disassembly of
the JIT-compiled kernel) sample the counter **1 cycle apart**. Back-to-back throughput of the
same read fits `delta = 1.72*N + 1.8`.

So for the empty zone, the 14 cycles are **7 issue slots + ~7 cycles of stall**, and the stall
comes from the *high-word* read inside the window feeding dependent instructions — not from
the endpoint read. The real compiled window (from `~/.cache/tt-metal-cache/.../brisc.elf`):

```asm
    5714:	lw	a7,496(a1)      # START: sample point       ┐
    5718:	lw	a4,504(a1)      # read 0x1F8 (MMIO)         │
    571c:	and	a4,a4,t6        # <- waits on a4            │ 7 issue slots
    5720:	or	a4,a4,s0        # <- waits on a4            │ measured 14 cycles
    5724:	sw	a4,0(a5)        # <- waits on a4            │ => ~7 cycles stall
    5728:	sw	a7,4(a5)        # <- waits on a7 (START)    │
    572c:	sw	t3,-1936(gp)    # wIndex += 2               │
    5730:	lw	a7,496(a1)      # END: sample point         ┘
```

**Consequence:** the 14 is a clean constant to subtract — it is code that genuinely executes
between the two samples, not measurement noise. (It is *not* an "op cost floor"; it is the
constructor tail plus the destructor head.)

**Untested idea to shrink it further.** Both window endpoints are the *low* read. If the
high-word read were moved outside the window on both sides — START: high → low, END: low →
high — the window would contain only the low read plus the real op. That needs two specialised
marker writers rather than one shared `mark_time_at_index_inlined`. Hypothesis, not measured.

## Verify your microbenchmark by disassembling it

Two traps hit while measuring the above, both invisible in the source:

- **Branch dispatch inside the timed region.** Putting `t0`/`t1` outside a `switch` put the
  jump-table lookup + `jr` (~20 instructions) *inside* the interval, and different cases took
  different paths. The result looked plausible but reported 24 cycles for two adjacent reads
  instead of 1. Move the timestamps **inside each case**.
- **The compiler merges cases by jumping into the middle of an unrolled sequence.** With
  several `case`s holding differently-sized unrolled blocks, GCC emitted *one* long sequence
  and gave each case an entry point partway in. Absolute values per case then differ by entry
  code, not by the thing under test.

Both were only found by disassembling the actual JIT object:

```bash
find ~/.cache/tt-metal-cache -path "*<kernel name>*" -name "brisc.elf"
riscv-tt-elf-objdump -d --no-show-raw-insn <that elf> | grep -n 'lw.*,496('
```

Note the `.o` in the same directory is **LTO IR**, not code — disassemble `brisc.elf`.

## Which register you read changes the cost by ~20%

`mark_time_at_index_inlined` reads a 32-bit low word and a 32-bit high word. On tt-1xx there
are **three** candidate addresses and they are not equally expensive — see
[`wall-clock-registers-and-latch.md`](wall-clock-registers-and-latch.md):

| read order | marginal cost | recorded window |
| --- | --- | --- |
| `0x1F4` (live high) → `0x1F0` (low) — **upstream** | 51.0 | 18 |
| `0x1F0` (low, latches) → `0x1F8` (latched high) | **40.0** | **14** |
| `0x1F8` (latched high) → `0x1F0` (low) — *wrong semantics, cost control only* | 37.0 | — |

Solving the three rows as simultaneous equations: the register swap alone (row 1 vs row 3,
same order) is 14.0 cycles/zone = **~7 cycles per read**; the order is worth 3.1 cycles/zone
the other way. Net 11 cycles/zone.

## Caveats

- **End-to-end this is small.** 11 cycles × ~64 zones/core ≈ 700 cycles against a ~165k-cycle
  kernel ≈ 0.4%, below the ±5% run-to-run spread. It matters for fine-grained measurement, not
  for throughput.
- **Always verify the measurement harness reproduces a known number first.** This harness was
  validated by getting 51.02 against a previously published 51; without that, a slope of 40
  could equally have been a broken loop.
- **Check the profiler's own output before trusting it.** 0 unnamed rows, 0 unpaired
  START/END, and a per-`(core, zone)` sample histogram that matches the expected op count.
  An over-budget kernel produces clean-looking, biased data — see
  [`device-profiler-zone-budget.md`](device-profiler-zone-budget.md).

## The recorded window is quantised by the enclosing loop's unrolling

Subtracting the empty-zone window from a measured zone assumes that window is constant. In a
real kernel it is not: **each unrolled copy of the enclosing loop gets its own window**, and
the copies can differ a lot.

Measured on the `matmul_multi_core` reader (kt loop, Kt = 20, unrolled by 2), one zone per
iteration on every kt, 70 four-tile cores, patched read order:

| zone | even kt | odd kt | spread | net (mean - 14) |
| --- | --- | --- | --- | --- |
| `noc_async_read_page::in0` | 57 | 56 | 1 | **+43.8** |
| `cb_push_back::in0` | 44 | 48 | 4 | **+32.2** |
| `get_write_ptr::in0` | 13 | 22 | 9 | **+3.7** |

280/280 samples per parity, so this is deterministic codegen rather than noise. Note the last
row: `get_write_ptr` is cheap enough that one unrolled copy measures **below** the empty-zone
window (13 < 14), i.e. a negative op cost -- its work has been scheduled into the shadow of
the zone's own bookkeeping.

Consequences:

- **Report the mean, not the p50, when the distribution is bimodal.** A p50 lands between the
  two modes and is not a value the hardware ever produced (`cb_push_back` reads 48 as p50 but
  only ever emits 44 or 48).
- **An op at or below roughly 2x the window is not measurable this way.** The subtraction is
  only meaningful when the body dominates the window.
- Two clean modes are a smell worth chasing: check whether they alternate with the loop index
  before reaching for a statistical explanation.
