# RPM targets an application-class RV64 core — likely the L2CPU x280, not the baby RISCV

Observed on tt-rpm `3094a66` against the vendored upstream spec
`tt-isa-documentation` `b0cfdc43` (`BlackholeA0/L2CPUTile/*`). Device facts are
**documented**; the target attribution is **inferred** and explicitly flagged.

## What is certain

RPM is a model of an **application-class, out-of-order RV64 core with an
L1 I$/D$ and a unified L2**. It is not a baby RISCV model (see
`does-not-model-baby-riscv.md`). The evidence, all from RPM's own tree:

| signal | where |
|---|---|
| RV64 | whisper defaults to `xlen=64`; the core model only loads 64-bit ELFs |
| OoO, 4-wide dispatch, 8-wide issue, ROB 128, PRF 128 | `models/cpu/src/core/config.yaml` |
| L1 I$ **and** L1 D$ **and** a unified L2 fed by both | `ChipSim.cpp:328-329` — `icache->setL2(l2); dcache->setL2(l2)` |
| CoreMark + Dhrystone as the headline workloads | `README.md`, `tests/` |
| placeholders for TAGE, ITTAGE, NFP, BTB, RAS predictors | `models/cpu/predictors/*/src/.gitkeep` |

CoreMark is an EEMBC benchmark for cached, branch-predicting application
processors, and TAGE/ITTAGE/RAS are the predictor structures such a core needs.
Neither belongs on a 1-IPC baby RISCV.

## Why "the L2CPU" is a good guess

On Blackhole there is exactly **one** application-class RV64 core, and it is in
the L2CPU tile:

> Each L2CPU tile contains a coherent cluster of four **SiFive x280** CPUs …
> (`BlackholeA0/L2CPUTile/README.md`)

Documented x280 hierarchy (`L2CPUTile/Caches.md`), which matches RPM's *shape*
(L1I + L1D + a unified L2 that both L1s fill from):

| level | x280 (documented) | RPM default |
|---|---|---|
| L1I | 32 KiB, **2-way**, VIPT, 64 B lines | 256 KB, 4-way, 64 B |
| L1D | 32 KiB, **4-way**, VIPT, 64 B lines | 64 KB, 8-way, 64 B |
| L1D outstanding misses | **1 — fills are serialized, "misses cannot be handled in parallel"** | `mshr_capacity 16` |
| L2 | 128 KiB, **8-way, per-hart, unified**, with prefetcher | unified, but a single instance, `enabled: false` by default |
| L3 | 2 MiB shared per tile, 16×128 KiB configurable | **absent** |
| cluster | 4 harts, coherent **within a tile only** | single core (`top.core0`) |
| MMU | MMU + BEU present | **absent** |

So the hierarchy has the right *shape* but not the right *numbers*.

## What is NOT established

- RPM never names its target: `grep -rniE 'ascalon|tensix|brisc|ncrisc|trisc|baby'`
  over `thirdparty/tt-rpm/` (excluding `ext/`) returns nothing, and neither does
  a search for `x280`/`sifive`/`l2cpu`.
- RPM's cache sizes do **not** match the documented x280, so it is not a
  validated x280 model — it is the right *class* of core, configurable.
- **Ascalon is an equally plausible intended target.** The upstream docs go out
  of their way to separate the two:

  > The SiFive x280 CPUs found in Blackhole L2CPU tiles have no relation to the
  > Ascalon CPUs being designed by Tenstorrent, other than both being 64-bit
  > RISCV CPUs. Some future Tenstorrent products are likely to include Ascalon
  > CPUs, but Blackhole contains SiFive x280 CPUs.

  RPM is a Tenstorrent-authored *design-space exploration* tool ("letting you
  study microarchitectural trade-offs") with placeholder dirs for
  `cluster/fabric/soc/memory_model/sys` — a roadmap toward modelling their own
  SoC. Modelling a SiFive part in that much detail is less obviously their job
  than modelling their own core.

**Net:** on p150a the L2CPU hypothesis is the right answer *in kind* — if RPM
is aimed at anything on that chip, the only candidate is the L2CPU's x280.
Whether the authors meant x280 or Ascalon is not decidable from the repo.

## Practical consequences

1. **Do not use RPM for Tensix baby-RISCV kernels.** Wrong class of core; see
   `does-not-model-baby-riscv.md`.
2. **If you do want to model the L2CPU**, RPM is plausibly the right tool, and
   the documented numbers give a starting configuration:

   ```
   -p top.core0.icache.params.cache_size_kb 32
   -p top.core0.icache.params.cache_associativity 2
   -p top.core0.dcache.params.cache_size_kb 32
   -p top.core0.dcache.params.cache_associativity 4
   -p top.core0.dcache.params.mshr_capacity 1     # x280: one fill at a time
   -p top.core0.l2cache.params.enabled true       # 128 KiB, 8-way, per-hart
   ```

   The `mshr_capacity 1` line is the interesting one: the upstream doc warns
   that serialized L1D fills cause "very poor performance for code which
   suffers lots of L1 data cache misses", and RPM's default of 16 hides that
   entirely. **Not yet measured** — these are documented numbers fed to the
   model, not a validated calibration.
3. Missing entirely for L2CPU work: the MMU/BEU, the L3, per-hart L2
   partitioning, and tile-level coherence. RPM is one core.

## Related

- `does-not-model-baby-riscv.md` — the baby RISCV gap, in detail.
- `what-it-models.md` — RPM's coverage and the measured prediction-vs-device table.
- `../isa/l0-data-cache-and-fence.md` — the baby RISCV L0 data cache.
