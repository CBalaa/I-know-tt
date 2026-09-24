# The L0 data cache, and what a baby RISCV `fence` actually does

**Finding.** Every Blackhole baby RISCV sits behind a **64-byte, per-core, non-coherent
L0 data cache**. It caches **L1** loads only. A `fence` instruction **flushes the whole
thing**. Wormhole has no such cache and treats `fence` as a **no-op** — which is why
tt-metal's `invalidate_l1_cache()` is `#if defined(ARCH_BLACKHOLE)` and compiles to
nothing on Wormhole.

Read from the vendored upstream ISA docs at `tt-isa-documentation` `b0cfdc43`
(2026-09-18). **Documented, not measured** — see the end of this note.

## What `fence` does on Blackhole

**`fence` is not a cache-flush instruction.** In standard RISC-V it is a pure *memory-ordering*
instruction: it constrains the order in which memory operations become visible to other harts
and devices, and says nothing about caches. The L0 data cache flush described below is a
**Blackhole implementation side effect**, not the instruction's purpose — this core implements
"strongest ordering" crudely, by stalling the pipeline and throwing the cache away rather than
tracking precise ordering. (It works as ordering precisely *because* the L0 D$ is
non-coherent: flushing it is how this core approximates "I will now see what others wrote
before the fence".) Do not carry the flush mental model to any other RISC-V core.

Upstream: [`BlackholeA0/TensixTile/BabyRISCV/InstructionSet.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/InstructionSet.md)
(the `fence` section) and
[`MemoryOrdering.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/MemoryOrdering.md)
("Enforcing stronger ordering").

Upstream does not expand the encoding, so it is spelled out here. `fence pred, succ` carries
two 4-bit sets, each with four bits — **i** = device input, **o** = device output, **r** = memory
read, **w** = memory write (bit values `i=8 o=4 r=2 w=1`). The sets say which accesses are
*ordered*: nothing in `succ` may be observed by another hart or device before anything in
`pred`. It constrains **what others can observe**, not this core's cache and not this core's
later loads. Verified encodings, assembled with the tt toolchain (`riscv-tt-elf-as`):

| assembly | encoding | fm | pred | succ |
| --- | --- | --- | --- | --- |
| `fence` | `0ff0000f` | 0 | `f` | `f` |
| `fence iorw, iorw` | `0ff0000f` | 0 | `f` | `f` |
| `fence rw, rw` | `0330000f` | 0 | `3` | `3` |
| `fence r, r` | `0220000f` | 0 | `2` | `2` |
| `fence w, w` | `0110000f` | 0 | `1` | `1` |
| `fence i, i` | `0880000f` | 0 | `8` | `8` |
| `fence o, o` | `0440000f` | 0 | `4` | `4` |
| `fence r, w` | `0210000f` | 0 | `2` | `1` |
| `fence.tso` | `8330000f` | 8 | `3` | `3` |
| `fence.i` | `0000100f` | — | — | — |

Two things worth taking from the table:

- **Bare `fence` *is* `fence iorw, iorw`** — both sets saturated. So tt-metal's
  `asm("fence")` already asks for the maximum, and there is no weaker form hiding in it.
- **`i`/`o` vs `r`/`w` is a platform-defined split.** Whether an address counts as "memory" or
  "device I/O" is a physical-memory-attribute decision, so `fence rw, rw` does **not** cover a
  region classified as device I/O — only the `i`/`o` bits do. That is a real reason to prefer
  bare `fence` for a poll loop over an MMIO register such as the CB counters. It also lines up
  with the documented conformance caveat: the babies' `fence` is conformant *"for device input
  and memory reads and memory writes, but **not for all cases of device output**"* — the one
  documented gap is the `o` bit.

`fence.i` is a **different instruction** (funct3=1, instruction-fetch fence) and the baby RISCV
cores do **not** implement `Zifencei`: the Wormhole docs state that executing it behaves as a
`nop` and that this is non-contractual, and the Blackhole extension list omits it too. tt-metal
only uses it on the Quasar (tt-2xx) path (`invalidate_l1_icache`). Do not reach for `fence.i` to
flush the L0 **data** cache.

**But on this hardware none of that matters.** The babies **ignore all eight bits** and execute
every `fence` in the strongest form the hardware supports — so `fence r, r` and
`fence iorw, iorw` are indistinguishable here, in behaviour and in cost:

1. the next instruction does not leave the frontend until the `fence` retires;
2. the `fence` does not enter the Load/Store Unit until the store queue has drained and
   every in-flight load has determined its result;
3. **entering the Load/Store Unit flushes the entire L0 data cache.**

The documented **gap**: (2) only means a store's write-request has been *sent*, not that
it was *processed*. Reordering is still possible between two requests on opposite sides
of a `fence` when the first is a store and the two target **different** memory regions. So a
`fence` is not a general cross-region store barrier; the read-back idiom that does close that
gap is described in the tt-llk `.claude/skills/mailbox-sync-audit/SKILL.md` (inside
`thirdparty/tt-metal`).

### Which effect serves which direction

Worth separating, because only one of the three effects is the cache flush:

| direction | what must hold | which effect does it |
| --- | --- | --- |
| **writer** — my stores must become visible in order | my write-requests are on their way | (2) drain the store queue |
| **reader** — I must observe others' stores in order | my later loads stop hitting a stale copy | (3) flush the L0 cache |

So **the cache flush only covers the read half.** The write half is the store-queue drain — and
per the gap above, it is only half a guarantee.

### `fence` is the *only* cache-maintenance tool here, and it is a sledgehammer

The baby RISCV cores implement neither `Zifencei` nor `Zicbom`, so there is **no**
`cbo.inval` / `cbo.clean` / `cbo.flush` and no `fence.i`. If you need the L0 data cache out of
the way, `fence` is the only instruction that does it — and it always discards **all four
lines**, never a single line. There is no targeted invalidate on this core.

Also note that "the L0 was flushed" does **not** imply a `fence` executed. The whole cache is
also discarded by:

- any atomic instruction (`Zaamo`), as a side effect;
- ~0.8% of cache hits, unless `cfg0.DisLowCachePeriodicFlush` is set;
- a core's own store, for the single line it hits.

The second of those is why a poll loop missing its `fence` still usually works.
### How long does a `fence` take? No documented number, and not a constant

**The docs give no cycle count for `fence`.** Searched: no latency appears anywhere near it.
The only quantitative statements that exist are about *occupancy*, not duration:

| statement | source |
| --- | --- |
| spends **one cycle in EX1** | `README.md:71` |
| spends **at least one cycle in the Load/Store Unit** | `README.md:18` |
| requires **one retire-order queue entry** (queue is 8 deep) | `README.md:89,91` |
| an instruction cannot enter EX1 until all earlier `fence`s have **retired** | `MemoryOrdering.md:72` |
| will not enter the Load/Store Unit until the **store queue has drained** and **all in-flight loads have determined their result** | `MemoryOrdering.md:73` |

That last row is the whole answer: the latency is **a wait on other events**, so it is not a
constant. Its components, and why each varies:

- **store-queue drain** — 0 when the queue is empty. Otherwise up to 4 entries, and with
  coalescing enabled the drain waits `cfg0.StMergeTimer` cycles (default **16**) to see whether
  the next store merges. A single pending store can therefore cost ~16+ cycles.
- **all in-flight loads must determine their result** — and per-region load latency ranges from
  **2** cycles (local RAM, or an L0 hit on L1) to **>= 12** (L1 atomic), with the NoC overlay
  registers at **>= 7**. One slow load in flight sets the floor.
- **pipeline drain** — the next instruction waits for the `fence` to retire, so the frontend
  serializes behind it.
- **the flush itself** — 4 lines, unquantified.

Note it is a **max over those waits, not a sum** — they overlap. Modelling it as a sum will
over-estimate.

#### Measured on p150a (BRISC, tt-metal `4f9fa9e0`) — added later

A bare `fence` between two wall-clock reads, 2560 samples per variant
(harness: `tt-loop-scheduler/test/zone_cost/fence_cost.cpp`):

| variant | mean | median | p50 | p95 | min | max | net |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `nop` (control) | 1.998 | 2.0 | 2 | 2 | 1 | 2 | — |
| **`fence`** | **11.000** | **11.0** | **11** | **11** | **11** | **11** | **9.00** |
| `fence.i` | 10.000 | 10.0 | 10 | 10 | 10 | 10 | 8.00 |
| `fence rw,rw` | 11.000 | 11.0 | 11 | 11 | 11 | 11 | 9.00 |
| `2 x fence` | 15.000 | 15.0 | 15 | 15 | 15 | 15 | 13.00 |
| `4 x fence` | 23.000 | 23.0 | 23 | 23 | 23 | 23 | 21.00 |
| `8 nop + fence` | 18.000 | 18.0 | 18 | 18 | 18 | 18 | 8.99 |
| `8 nop` (control) | 9.007 | 9.0 | 9 | 9 | 9 | 20 | — |

- **The distribution is degenerate**: all 2560 `fence` samples are exactly 11, so
  min = max = mean = median = p50 = p95. Three independent runs agree. On a core with no
  cache or memory contention, a bare `fence` is **deterministic**, not noisy.
- **Single `fence` = 9 cycles; back-to-back fences cost 4 cycles each after the first**
  (15−11 = 4, and 23−15 = 8 for two more). So the *first* fence pays a serialization cost
  the later ones do not.
- **The 9 is not "waiting for an in-flight load."** The hypothesis above predicts the cost
  should drop once the pipeline is drained; it does not: with 8 `nop`s inserted before the
  `fence` (so the preceding MMIO read has long resolved) the net cost is **8.99** — the same.
  In this loop the 9 cycles are the fence's own pipeline serialization plus its flush.

⚠️ **Scope of the measurement**: the store queue was **empty** (the loop's sample store lands
*after* the timed interval). The doc's other component — waiting for the store queue to drain,
with a default 16-cycle merge timer — is therefore **not** exercised, and a `fence` with an
unretired store can be considerably more expensive. That variant is still unmeasured.

**In the one loop we care about, it should be close to constant.** `cb_reserve_back`'s spin body
contains **no stores** (so the store queue drains to empty and stays there) and its only load is
consumed by the branch that closes the loop (so no load is left in flight across the back edge).
Both variable components collapse. That is inference from the loop's shape, not a measurement.

The nearest measured anchor the consuming repo has: the profiler zone around
`cb_reserve_back::in0` in the matmul reader measured **46-53 cycles, flat** (p50 48-51, max 53)
over every `kt` and all 4 tiles, against an **18-cycle** empty-zone floor — so the whole
reserve-with-`fence` call is ~30 cycles. See
[`../tt-metal/device-profiler-zone-budget.md`](../tt-metal/device-profiler-zone-budget.md). That
number **contains** a `fence` but does not isolate it.

To isolate it: measure a zone around nothing (floor), around a bare `fence`, around a `fence`
preceded by 1..4 stores (store-queue dependency), and around a `fence` preceded by a load from a
slow region (in-flight-load dependency). The existing `test/zone_cost/` harness already does
exactly this kind of per-op measurement on p150a.

### Consequence for a timing model: a `fence` is a wait node, not a fixed-latency op

Because the latency is a wait on *outstanding work at issue time*, it is **context-dependent —
but not unpredictable**. A model that tracks what is in flight predicts it exactly; a model that
assigns `fence` a constant cycle count does not. The context that matters:

- store-queue occupancy at issue (0..4 entries), and whether `DisStMerge` is clear so the drain
  waits `StMergeTimer` (default 16);
- how many loads are in flight and **which regions** they target (2 to >= 12 cycles each);
- whether any of those loads is still blocked on a FIFO or wait condition, which can hold a
  request unprocessed indefinitely;
- `cfg0.DisCsrSync` — if **set**, the frontend serialization for `fence` is disabled, which
  removes one of the three components entirely (default is clear, i.e. serialization on).

Three properties follow, and they are what a scheduler should encode:

1. **It is a max, not a sum.** The waits overlap. Adding the components over-estimates.
2. **Its marginal cost is sub-additive with the work before it.** Issuing a `fence` right after a
   long-latency load is nearly free — that load was going to take that long anyway. Issuing it
   after a run of ALU work costs the full drain. So place a `fence` as late as possible.
3. **Unlike a load, its latency cannot be hidden by later instructions.** The docs' standard
   latency-hiding recipe ("N-1 independent instructions need to follow the load") does not apply:
   an instruction cannot even enter EX1 until all earlier `fence`s have retired, so nothing after
   the `fence` can fill its shadow. The only way to hide a `fence` is with work *before* it.

There is still a floor when nothing is outstanding: EX1 = 1 cycle, Load/Store Unit >= 1 cycle,
plus the retire/frontend handoff.

## What the L0 data cache is

Upstream: [`BlackholeA0/TensixTile/BabyRISCV/README.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/README.md)
("L0 Data Cache" section) and the memory-latency table in the same file.

| property | value |
| --- | --- |
| size | 64 B = 4 lines x 16 B |
| scope | private per baby RISCV |
| policy | write-through; **never dirty** (an L1 store flushes a line it hits) |
| coverage | loads **whose address targets L1** can hit — but this is *permissive*, not a scope limit; the cache demonstrably serves other regions too (see the retraction and the git history below) |
| coherence | **none** — another client writing L1 does not invalidate this core's L0 |
| flush triggers | any `fence`, any atomic, and a ~0.8% random chance per hit |
| disable | `cfg0` CSR bit 3 (`DisLowCash`) turns the cache off entirely |

**"Non-coherent" here means stale, and only stale.** The divergence is one-directional: the
L0 copy can be **older** than the backing store, and can never be **newer**. That follows from
write-through + never-dirty — there is nothing in the L0 that the backing store has not already
seen. So the symptom is a stale *read*, not two replicas diverging. Which direction applies
depends entirely on **who wrote**:

| who wrote the backing store | is this core's L0 stale? |
| --- | --- |
| **this core**, via a store or atomic | no — its own store discards the line it hits |
| **any other client** — another RISCV, the NoC, an unpacker/packer, a Tensix instruction | **yes** — nothing invalidates this core's L0 |

Upstream states exactly that carve-out: care is required reading L1 from a baby RISCV *"unless
the data in question was written by that same baby RISCV using RISCV store instructions or
RISCV atomic instructions"*.

**Coherence is a design choice, not a physical necessity** — the same Blackhole has a coherent
example. On the L2CPU (SiFive) tile the L1D is *coherent with and inclusive of* L2/L3, and a NoC
access to a cacheable address is performed *through* L3 and is therefore coherent too. The Tensix
L0 D$ sits *beside* that path instead: a NoC write to L1 simply does not tell it. (The L2CPU's
coherence is itself bounded — it does not extend across L2CPU tiles.)

A second, separate "pending" layer exists even for this core's own view: the store queue is
*always dirty*, and a load overlapping it drains it rather than bypassing.

Two consequences the upstream doc states outright, and which are the whole reason this
note exists:

- A core may read a **stale** value from L1 written by *any other client* — another
  RISCV, the NoC, an unpacker/packer, a Tensix instruction.
- **Polling loops are called out by name**: they should either contain a `fence` or poll
  with an atomic instruction. The upstream `MemoryOrdering.md` gives the canonical
  shape — pop a mailbox, then `fence`, then load the L1 address the mailbox pointed at.

The flush is **one-shot, not a mode change**: the cache is empty for exactly one
instruction and then refills on the next miss. So a poll loop must execute the `fence`
**every iteration** — hoisting it out of the loop silently reintroduces the stale read.
That is why the compiler's output puts `fence` as the first instruction of the loop body
with the polled load immediately after it, and why the `~0.8%` random flush below makes
such a mistake look intermittent rather than broken.

The `~0.8%` random flush is worth internalising: a missing `fence` in a poll loop
**usually still works**, which is exactly what makes the bug class hard to see.

## ⚠️ RETRACTED: "the cache covers L1 only"

**An earlier version of this note argued, from the pipeline diagram, that the L0 data cache
covers the L1 scratchpad and nothing else — so a load from Local Data RAM or from the
NoC-overlay registers could never be served stale. That conclusion was WRONG.** It is kept
here, struck down, because the reasoning failure is instructive.

What the diagram actually says (and still says) is only about *routing*.
`Diagrams/Src/BabyRISCV.lua` wires two muxes:

```lua
-- the mux that feeds L1: exactly two inputs, the two L0 caches
for _, c in ipairs{ic, dc} do ... end                  -- ic = L0 I$, dc = L0 D$

-- the mux the Load/Store Unit fans out through
for _, box in ipairs{dc, lram, mop_cfg} do ... end     -- lram = Local Data RAM
```

That much is true: `dc` (L0 D$) and `lram` (Local Data RAM) are sibling destinations on the
Load/Store Unit's fan-out, and `dc` is on the branch that reaches L1.

**The invalid step was going from "sibling in the diagram" to "not cached".** A block
diagram draws request routing, not the cache controller's fill policy. The hardware
falsified the inference — see the history in the next section: the L0 data cache demonstrably
serves loads to the NoC-overlay register region, which is one of the twelve MMIO siblings on
that same fan-out.

Consequences of the retraction:

- The ISA prose ("loads *whose address targets L1* can be satisfied by the cache"; "a tiny L0
data cache *between it and L1*") is **permissive, not restrictive**. It says L1 loads *can*
hit; it never says other addresses cannot. Reading it as an exhaustive scope was a mistake.
- tt-metal's own comment — "the cache covers all of L1 (no MMU or range registers)" — is
  closer to the truth than this note was, misnomer aside. The operative clause is **"no MMU
  or range registers"**: nothing bounds what may be cached. The original 2024 wording was
  blunter still: "If cache is enabled, the entire L1 is cached (no MMU)".
- **The Local Data RAM claim is therefore unproven too.** I have no evidence either way for
  local RAM, and the diagram argument that was supposed to settle it is void. Do not repeat
  the claim that a `fence` cannot affect a local-RAM read.

### What would actually settle coverage

Not a diagram. Either an explicit statement upstream, or the experiment: set `cfg0` bit 3
(`DisLowCash`) to disable the L0 data cache and compare behaviour, or write a poll loop over
a known-remote-written address in the region of interest and see whether it needs a `fence`.
## What address range does the cache serve? The docs do not say.

Asked directly — *a load to which addresses gets cached?* — the vendored ISA documentation has
**no answer**. Searched exhaustively (`BlackholeA0/` + `WormholeB0/`, every `.md`):

- **The TensixTile memory map has no cache column.** The table header is `Name and address
  range | RISCV B | RISCV T0 | RISCV T1 | RISCV T2 | RISCV NC | NoC` — per-RISC access and NoC
  access only. No region anywhere is annotated cacheable or uncacheable.
- **`cfg0` has no range, way or tag configuration.** Only two cache-related bits exist:
  bit 3 `DisLowCash` (disable the *whole* cache) and bit 24 `DisLowCachePeriodicFlush` (the
  ~0.8% whole-cache flush). There is no base/limit register to configure.
- **Nothing says any region is *not* cached.** Every hit for `cacheable` / `uncached` /
  `not cached` in the tree belongs to the **L2CPU tile** — the SiFive cluster's L1D/L2/L3 —
  which is a different piece of silicon entirely.
- **Wormhole has no L0 data cache at all** — the string "L0" never appears in the Wormhole
  BabyRISCV docs (it has instruction caches only).

### Every scope statement the docs do make

Six sentences, all tying the cache to L1, none of them bounding it:

| where | what it says |
| --- | --- |
| `README.md:140` | the cache sits *between the core and L1* |
| `MemoryOrdering.md:59` | loads *whose address targets L1* **can be** satisfied by it; misses fill *from L1* |
| `README.md:142`, `MemoryOrdering.md:61` | *stores to L1* flush a hit line |
| `MemoryOrdering.md:65` | atomics operate on *L1 addresses* and flush the whole cache |
| `L1CacheTagSearchAccel.md:57-59` | a read to an *L1* address that is "not an L0 data cache hit" is intercepted, and its result is used to populate the L0 |
| `L1.md` | L1 is plain RAM, not a hardware-managed cache |

The second is **permissive** — "can be satisfied by" — not a scope limit. Reading it as
exhaustive is the mistake this note already made once.

### The latency table is an omission, not a rule

`README.md:76-83` gives only L1 a hit/miss pair (`L1 ... with L0 data cache hit` = 2 cycles,
`... with L0 data cache miss` = >= 8). Every other region gets a single latency with no
hit/miss distinction, and the NoC overlay registers are listed at **>= 7**. That *looks* like
"only L1 is cached" — and it is exactly the trap. The Whisper hang proves the NoC-overlay
region **is** served by the cache. The table gives per-region base/uncached latencies; it is
not a cacheability declaration.

### What can actually be pinned down

| region | served by the L0 D$? | evidence |
| --- | --- | --- |
| L1, `0x0000_0000`-`0x0017_FFFF` | **yes** | documented; and tt-metal keeps an invalidate in `noc_semaphore_wait` / `noc_semaphore_wait_min`, which poll L1 semaphores |
| NoC overlay registers, `0xFFB4_0000`+ | **yes** | `7e6553092db` fixed a Whisper hang by re-adding the invalidate to `cb_reserve_back` |
| Core-local data RAM | **unknown** | no doc statement; the diagram argument that claimed "no" is void (see retraction) |
| Mailboxes, Tensix GPRs, semaphores, other MMIO | **unknown** | not documented. `noc_semaphore_wait` invalidates, but *absence* of an invalidate proves nothing — `cb_wait_front` polls a cacheable region with no invalidate |

### The working assumption

The absence of any range register, plus the "no MMU or range registers" comment, points to a
cache that fills **by address with no configured bounds** — the docs describe the L1 case
because that is the only case where caching is semantically meaningful (software-managed
memory that other clients write). Mark that as inference, not documentation.

The engineering rule that follows, and the one tt-metal's history converged on:

> **Do not assume any load address is uncached.** If another client writes an address and this
> core polls it, put a `fence` in the poll loop. That is what tt-metal does for L1 semaphores
> and for the NoC-overlay CB counters.

`cb_wait_front` violates that rule, which is why it stays on the open-questions list rather
than being cited as precedent.

## ⚠️ L1 is not a cache, and tt-metal's comment says it is

`thirdparty/tt-metal/tt_metal/hw/inc/internal/tt-1xx/cache.h` describes
`invalidate_l1_cache()` as invalidating *"Blackhole's entire L1 cache"* and calls the
4x16B lines *"L1 lines"*. **Both are misnomers.**

- **L1 is a 1536 KiB scratchpad RAM** at `0x0000_0000`-`0x0017_FFFF`, not a hardware
  cache. It is software-managed; the "L1 Cache Tag Search Accelerator" exists precisely
  because L1 is used *as* a software cache
  ([`L1CacheTagSearchAccel.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/L1CacheTagSearchAccel.md)).
- The 4x16B write-through cache is the **L0 data cache**.

The memory map in `BlackholeA0/TensixTile/BabyRISCV/README.md` is the authority: L1
scratchpad and the `0xFFB*` register regions are listed as separate address ranges with
different load latencies (2 cycles for an L0 hit on L1 vs >=7 cycles for the NoC overlay
register space).

The function's *effect* is right and its *name* is wrong: it is the L0 data cache that
gets flushed.

## The `cb_reserve_back` fence is LOAD-BEARING — the git history proves it

This is the empirical answer to the coverage question, and it is why the section above had to
be retracted.

tt-metal stores CB credit counters in the NoC overlay "stream" registers at
`0xFFB4_0000 + n*0x1000` — **not** in L1 (see
[`../tt-metal/cb-credit-counters-in-noc-stream-regs.md`](../tt-metal/cb-credit-counters-in-noc-stream-regs.md)).
`cb_reserve_back` flushes the L0 cache on **every** iteration of its poll loop, and nothing
else in tt-metal's CB API does. The history of that one line, from the `tt-metal` history:

| commit | date | what it did |
| --- | --- | --- |
| `fcca8fe79d4` | 2024-07-29 | *"Enable BH l1 cache and add api to invalidate cache"* — **removes `disable_lowcache()`** from all four firmware RISCs (i.e. turns the cache **on**), and adds `invalidate_l1_cache()` to `cb_reserve_back`, `cb_wait_front`, `cb_pages_*`, the NOC barriers and the semaphore waits |
| `09a77513d37` | 2024-12-12 | *"Minor perf tweaks/cleanup to device NOC code"* — **removes** it again from `cb_reserve_back`, `cb_wait_front` and `cb_pages_*` (keeps it in `noc_async_read_barrier`) |
| `7e6553092db` | 2025-05-06 | *"**Add missing l1 cache invalidation to resolve Whisper hang**"* — **adds it back, to `cb_reserve_back` only**. One line changed. Body: *"Whisper was hanging on P100a due to missing cache invalidation"* |

So the sequence is: cache off → cache on **and** flush everywhere → a "perf cleanup" strips the
flushes → **a model hangs** → the flush comes back in exactly one place.

Two conclusions:

1. **The fence is not defensive.** It was restored to fix an observed hang on real silicon.
   Removing it is known-bad, not a micro-optimisation. (The pre-2024 world where it was absent
   was a world where `disable_lowcache()` was in effect.)
2. **The L0 data cache serves loads to the NoC-overlay register region.** That region is one of
   the twelve MMIO siblings on the Load/Store Unit's fan-out — the very fact this note
   previously used to argue the opposite. The counter read *is* cacheable, and a stale line
   there is what hung Whisper.

### What is still open

- **Why `cb_wait_front` does not need it.** It polls the sibling counter (`pages_received`)
  written by the producer, from the same 16-byte cache line as `pages_acked`, and it has been
  flush-free since 2024-12 without a matching hang report. That is now a sharper question than
  before, because the mechanism is proven real: either the wait path is genuinely safe for a
  reason nobody wrote down, or it is latent and only the ~0.8% random flush hides it.
- **Whether Local Data RAM is covered.** Unproven either way; see the retraction above.
- **Which value was stale — RESOLVED: it must be `pages_acked`.** The loop reads three things:
  `pages_acked` and `pages_received` from the NoC-overlay registers, and `fifo_num_pages` from
  `cb_interface` in L1. `fifo_num_pages` cannot go stale: this RISC wrote it itself during
  kernel setup, and a core's own store flushes the L0 line it hits, so the next read refills
  correctly (and it is constant within a kernel anyway). `pages_received` is the producer's own
  counter, snapshotted once outside the spin. Only `pages_acked` is written by *another* client
  — the consumer core, or a Tensix instruction on the compute side — and nothing invalidates
  this core's L0 when that happens. So the stale read is `pages_acked`.

  That generalises into a rule worth carrying: **each side read-modify-writes only the counter
  it owns and polls only the counter the other side owns, so the polled one is always the one
  that can be stale.** `cb_reserve_back` (producer) snapshots `received` and polls `acked`;
  `cb_wait_front` (consumer) snapshots `acked` and polls `received`.

- **Why the first iteration needs the flush too.** The L0 cache is *not* cleared between kernel
  runs (which is exactly why TRISC0 has to zero the counter registers explicitly via
  `init_sync_registers`). So on entry the line may already hold a stale copy left by an earlier
  kernel or by an earlier point in this one — the flush at the top of the loop body covers that
  case as well as the within-loop case.

### Lesson worth keeping

The reasoning error was treating a **block diagram's routing** as a statement about **cache
fill policy**, and treating a **permissive** ISA sentence ("L1 loads *can* hit") as an
exhaustive scope. Both felt like strong evidence and both were wrong. What settled it was
`git log -S` on the one line — cheap, and available the whole time.

## Where this shows up in tt-metal

| site | flush? | note |
| --- | --- | --- |
| `invalidate_l1_cache()` (`hw/inc/internal/tt-1xx/cache.h`) | is the flush | `asm("fence")`, BH-only |
| `cb_reserve_back` | every poll iteration | the only CB wait that flushes — and **load-bearing**: restored by `7e6553092db` to fix a Whisper hang |
| `cb_wait_front` | never | |
| `llk_wait_tiles`, `llk_wait_for_free_tiles` (TRISC) | never | |
| `noc_async_read_barrier` / `noc_async_write_barrier` | once, after the spin | `dataflow_api.h` |

Detail worth knowing: the BH definition is **basic asm** — `asm("fence")`, no `:`, no
clobber list — so it is implicitly volatile but is **not** a compiler memory barrier.
Every Quasar equivalent in tt-metal writes `__asm__ __volatile__("fence" ::: "memory")`
instead. It is harmless at the current call sites only because they read through
`volatile` pointers; reusing `invalidate_l1_cache()` around a non-volatile access would
let the compiler hoist the load out of the loop.

## Not measured

Everything above is **read from the upstream ISA documentation, the vendored pipeline
diagram source, and the tt-metal source and git history**. Nothing here was measured on a
device by me — but note that the key claim (the fence is load-bearing) *was* measured
upstream: it is the stated cause of a fixed Whisper hang on P100a.

Still unverified:

- whether the L0 data cache really behaves as documented (size, flush, the ~0.8% random
  flush);
- ~~what a `fence` actually costs in cycles~~ — **now measured**: a bare `fence` is
  **9 cycles** (deterministic; 4 cycles each for back-to-back fences), and the cost does *not*
  drop when the pipeline is drained, so it is the fence's own serialization rather than a wait
  on in-flight loads. Still unmeasured: a `fence` with an **unretired store** behind it (the
  store-queue-drain component, default merge timer 16 cycles). See the table above;
- **how far the cache's coverage actually extends.** All that is proven is that it reaches
  the NoC-overlay register region (`0xFFB4_0000`). Whether it also covers Local Data RAM is
  open, and the `cfg0.DisLowCash` experiment would settle it;
- **why `cb_wait_front` is safe without a flush** while `cb_reserve_back` is not. This is
  the sharpest open question in this note.

### What NOT to conclude from this note

Do not remove the flush from `cb_reserve_back` on the grounds that the counter "shouldn't be
cached". That exact reasoning was wrong once already, and the cost of being wrong is a hang.
