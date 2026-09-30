# llvm-mca as a streaming timing engine: what the library API can and cannot do

**Finding.** `llvm/lib/MCA/` is a real component library (`LLVMMCA`,
`llvm/lib/MCA/CMakeLists.txt:1`) and it already contains a *streaming* entry
point — `mca::IncrementalSourceMgr` plus the `InstStreamPause` protocol — that
lets an out-of-tree driver inject instructions over time and preserve pipeline
state across `Pipeline::run()` calls. There is **no** cycle-level stepping API:
`Pipeline::runCycle()` is private (`llvm/include/llvm/MCA/Pipeline.h:67`).
External stalls *can* be injected with zero upstream changes through
`CustomBehaviour::checkCustomHazard()` returning 1 (re-probed every cycle).
Multi-stream arbitration does not exist at all.

Read against `llvm-project` `34152bb7d2d` (LLVM 24.0.0git) in this workspace.

## The streaming protocol (verified by building a driver)

An out-of-tree driver was built against the in-tree `libLLVMMCA.a` in
`build/llvm-mca` (`-fno-rtti` is required, matching the LLVM build). Protocol:

```cpp
mca::IncrementalSourceMgr ISM;
MyCB cb(*STI, ISM, *MCII);                 // subclass CustomBehaviour
auto P = MCA.createDefaultPipeline(PO, ISM, cb);
for each batch:
  ISM.addInst(IB.createInstruction(MCI, {}));   // or addRecycledInst()
  auto C = P->run();                            // returns InstStreamPause error
ISM.endOfStream();
P->run();                                       // drains to retirement
```

Measured on `riscv32/rocket-rv32`, 8 instructions
(`addi` ×7 + `mul a3,a3,a3`):

| mode | cycles | retired |
| --- | --- | --- |
| one `run()`, `CircularSourceMgr` | 11 | 8 |
| stream, 1 instruction per `run()` | 11 | 8 |
| stream, 2 / 4 / 8 per `run()` | 11 | 8 |

Cycle counts are **identical** to the batch run at every tile size, and each
`run()` with one staged instruction advances roughly one simulated cycle. This
matches upstream's own `TestIncrementalMCA.cpp` (`TestResumablePipeline`).

## Injecting external stalls

`checkCustomHazard` is called from exactly one place:
`llvm/lib/MCA/Stages/InOrderIssueStage.cpp:136`. The out-of-order path
(`Scheduler`/`ExecuteStage`) has **no** CustomBehaviour hook at all. So this
works only for `MicroOpBufferSize = 0` models — which `tt-bh` is
(`MicroOpBufferSize = 0`, `IssueWidth = 1`).

Returning `N` = blind wait of N cycles (hook not re-consulted). Returning `1`
while an external condition holds = **re-consulted every cycle**, i.e.
block-until-arbitrary-external-event. Measured, stall injected after the 3rd
tile:

| gate | cycles | CB probes |
| --- | --- | --- |
| none | 11 | 0 |
| 5 | 16 | 5 |
| 20 | 31 | 20 |
| blind 20 | 31 | 0 |

The stall cost is added 1:1. This is the mechanism a "hardware FIFO is full"
model should use. The CB holds a pointer to the driver's own state; it is
constructed by the caller, so no registry is needed.

## Cost of a run

`llvm-mca` CLI, `riscv32/rocket-rv32`, `-iterations=1`, independent `addi`:

| instructions | wall | maxRSS |
| --- | --- | --- |
| 100 k | 0.73 s | 138 MB |
| 300 k | 2.05 s | 390 MB |
| 1 M | 6.77 s | 1.25 GB |
| 3 M | 25.2 s | 3.58 GB |

Linear, ~8 µs/instruction, ~1.2 KB/instruction. The same trace through the
**library-only** streaming driver is ~3.1 µs/instruction and ~0.82 KB per
instruction (2 M instructions in 6.2 s). Memory, not time, is the binding
constraint: both O(N) structures (`CodeRegion::Instructions`,
`LoweredSequence` at `llvm/tools/llvm-mca/llvm-mca.cpp:645`) hold every
instruction alive. `InstrBuilder::setInstRecycleCallback` +
`IncrementalSourceMgr::setOnInstFreedCallback` are the intended fix
(`TestInstructionRecycling`).

`-iterations=N` loops the same instruction sequence N times through
`CircularSourceMgr`; `EntryStage::getNextInstruction()` **copies** the template
`Instruction` each iteration, but `RegisterFile` state is *not* reset — so
register definitions genuinely carry across iterations. `mul a0,a0,a0` on
`rocket-rv32` (latency 4) gives `4N+1` cycles, i.e. real loop-carried RAW
behaviour. There is no `LoopCarriedDependency` class anywhere in MCA; this is a
side effect of the persistent `RegisterFile`, and `BottleneckAnalysis` merely
*labels* cross-iteration edges (`Views/BottleneckAnalysis.cpp:312`).

## `# LLVM-MCA-LATENCY` is a region, and it is quadratic

The directive starts/ends an **InstrumentRegion** covering *all following*
instructions until the next `LLVM-MCA-LATENCY` comment
(`InstrumentRegionCommentConsumer::HandleComment`,
`llvm/tools/llvm-mca/CodeRegionGenerator.cpp:174-178`). `InstrumentManager::
customize()` overwrites `W.Latency` for **every** write and `ID.MaxLatency`
(`llvm/lib/MCA/CustomBehaviour.cpp:62-76`). ReadAdvance / forwarding is *not*
disabled — it is subtracted from the overridden latency at use time
(`RegisterFile::checkRAWHazards`, `llvm/lib/MCA/HardwareUnits/RegisterFile.cpp:601`).

Verified: a single `lw` with `LATENCY L` followed by 11 independent
instructions gives `max(14, L+12)` cycles (L=0→14, 2→14, 4→16, 8→20, 20→32,
40→52), so the value is honoured exactly and per instruction. `mul` (model
latency 4) forced to 8 gives `8N+1` over N iterations.

Two traps:

1. **Quadratic blow-up.** Every marker opens a new region; then
   `InstrumentRegions::getActiveInstruments(Loc)` and
   `CodeRegions::addInstruction` both scan *all* regions per instruction
   (`llvm/tools/llvm-mca/CodeRegion.cpp:161-171` and `:27-32`). Measured, one
   marker per instruction:

   | instructions | wall (per-instruction markers) | wall (one marker) |
   | --- | --- | --- |
   | 5 000 | 0.17 s | — |
   | 10 000 | 0.64 s | — |
   | 20 000 | 2.73 s | 0.16 s |
   | 100 000 | — | 0.85 s |

   4× per doubling ⇒ O(N²). Extrapolated to 3 M instructions: hours.

2. **No descriptor sharing and no recycling.** `canCustomize()` forces
   `createInstrDescImpl()` per instruction and returns early *before*
   `ID->IsRecyclable` is set (`llvm/lib/MCA/InstrBuilder.cpp:626-629` vs `:632`),
   so every instrumented instruction allocates its own `InstrDesc` in
   `CustomDescriptors` and can never be recycled. Latency instrumentation and
   streaming memory reclamation are mutually exclusive.

Also: `-instruction-info`'s "Latency" column is computed straight from the
scheduling model (`Views/InstructionInfoView.cpp:213`), so it does **not** show
the overridden value — the report and the simulation disagree.

## What has no representation

* Multiple streams / cross-stream arbitration: `grep -rniE
  "multistream|multi-stream|multiplex|arbitrat|cross-stream" llvm/lib/MCA
  llvm/include/llvm/MCA llvm/tools/llvm-mca` → **0 hits**. One `Pipeline`, one
  `SourceMgr`, one `EntryStage`.
* Sharing a backend across pipelines: `InOrderIssueStage` owns `ResourceManager
  RM` **by value** (`Stages/InOrderIssueStage.h:57`) and its only constructor is
  4-arg (`:115`); `Context::createInOrderPipeline` always builds fresh `PRF` and
  `LSU` (`Context.cpp:76-78`). Passing an external `ResourceManager` is a
  compile error.
* Injection of synthesised instructions by a hook: `mca::Instruction` is created
  in exactly two places — `EntryStage.cpp:40` (copy of a `SourceMgr` entry) and
  `InstrBuilder.cpp:696`. No hook can add one.
* Changing write-back latency *at issue time*: `getLatency()` is
  `Desc.MaxLatency` (`Instruction.h:544`), `Desc` is a `const InstrDesc &` with
  no setter, and `InstrDesc`'s copy ctor is deleted (`Instruction.h:491`). Once a
  `mca::Instruction` exists its latency is frozen — but see the next section for
  the channel that does exist *before* that point.

## Per-instance latency from an external function IS expressible

Not at issue time — at **feed** time, with zero LLVM patches, through
`LatencyInstrument`. Verified chain:

1. The driver owns the `InstrBuilder`. `createInstruction(MCI, IVec)`
   (`InstrBuilder.h:123`) takes `IVec` as **caller-supplied raw `Instrument *`s**;
   the CLI does exactly this at `llvm-mca.cpp:650-653`.
2. Put one `LatencyInstrument(std::to_string(N))` in `IVec`.
3. The stock `InstrumentManager::canCustomize()` returns true when it sees
   `LatencyInstrument::DESC_NAME` **and** `hasValue()` (`CustomBehaviour.cpp:52`).
4. `createInstruction` therefore takes the `createInstrDescImpl()` branch instead
   of `getOrCreateInstrDesc()` (`InstrBuilder.cpp:675-677`) — the
   `(Opcode, SchedClassID)` descriptor cache is bypassed.
5. `createInstrDescImpl` ends with `IM.customize(IVec, *ID)`, which writes `N`
   into **every** `W.Latency` and into `ID.MaxLatency`
   (`CustomBehaviour.cpp:62-76`), then parks the descriptor in
   `CustomDescriptors` (`InstrBuilder.cpp:626-629`).
6. `Instruction::execute()` does `CyclesLeft = getLatency()` at issue, and
   `WriteState::onInstructionIssued()` pushes `max(0, N - ReadAdvance)` to
   subscribers — so the bypass network still applies to the external value.

`N` is a genuine per-dynamic-instance write-back latency, and `ReadAdvance` is
*not* disabled.

### The hard constraint: the oracle is consulted at feed time

The driver feeds instructions **ahead of** the simulated clock, so this works
only for an oracle that does not need to know the current cycle. That covers the
case that matters most here — **address-dependent load latency**. In a
dynamic-trace driver the address (hence the local-RAM / L0 / L1 / NoC region, and
which L0 line) is known when the instruction is fed, so this closes part of gap
#1 in [`tt-bh-model-accuracy.md`](tt-bh-model-accuracy.md).

It does **not** cover a latency that depends on the simulated cycle itself (a
line evicted at a time determined by MCA's own clock). That case can only go
through `checkCustomHazard` as a *pre-issue* wait — a different semantic channel.

### Traps, all verified

1. **Silent fallback.** `LatencyInstrument` parses `Data` with
   `getAsInteger(10, L)`. An empty string, `-1`, or `3.5` leaves `Latency` unset →
   `hasValue()` false → `canCustomize()` false → the **table latency is used and
   nothing is reported**. Clamp and assert in the driver.
2. **One value for all writes.** `customize()` sets `W.Latency` in a loop; the
   source carries `// TODO Allow to customize a subset of ID.Writes`.
3. **Recycling is destroyed.** The `canCustomize()` branch returns *before*
   `ID->IsRecyclable` is set (`InstrBuilder.cpp:626-629` vs `:632`), so an
   instrumented instruction can never be recycled. Per-instance external latency
   and streaming memory reclamation are **mutually exclusive** — and memory is the
   measured binding constraint (~1.2 KB/instruction; 3 M instructions = 3.58 GB).
   Instrument only the instructions whose latency actually varies.
4. **`CustomDescriptors` is never freed.** It is
   `SmallVector<std::unique_ptr<const InstrDesc>>` (`InstrBuilder.h:83`), and
   `InstrBuilder::clear()` resets only `Descriptors` / `VariantDescriptors`
   (`:110-115`). It grows for the process lifetime, across regions.
5. **The report disagrees.** `-instruction-info`'s Latency column reads the
   scheduling model (`Views/InstructionInfoView.cpp:213`), not the descriptor. A
   library driver printing its own report avoids this.

## Useful bonuses

* `LSUnitBase` is a pure-virtual interface (`LSUnit.h:88-131`) and
  `InOrderIssueStage` takes `LSUnitBase &` — a custom load/store unit is
  injectable if you hand-build the pipeline, and `LSUnit::isReady()` is
  explicitly documented as the override point for extra memory-type knowledge
  (`LSUnit.h:156-174`). This is the clean place to model a full store FIFO.
* `Pipeline::appendStage()` is public (`Pipeline.h:74`) and `Stage` is exported
  with virtual `isAvailable`/`execute`/`cycleStart`/`cycleEnd` — a custom stage
  can gate the entry stage per cycle.
* `LQSize`/`SQSize` of 0 means *unlimited* (`LSUnit.h:100-101`), so
  `-lqueue`/`-squeue` must be set explicitly or memory ops never back-pressure.
* `Pipeline::Cycles` has no getter (`Pipeline.h:65`); on pause `run()` returns
  the `InstStreamPause` error and discards the count, so a streaming driver must
  count cycles itself via `HWEventListener::onCycleEnd()`.
