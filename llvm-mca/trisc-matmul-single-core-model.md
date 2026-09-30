# TRISC timing model for `matmul_single_core`

This note defines the model boundary for the Blackhole
`example/standalone/matmul_single_core` kernels. It assumes that the Reader and
Writer NoC timelines are already known and that their circular-buffer
visibility events are supplied as inputs. It does **not** attempt to predict
NoC service time.

Observed on tt-metal `4f9fa9e0`, Blackhole p150a, with the generated assembly in
the consuming repository. Values marked as measured or documented are kept
separate from timing constants that still need calibration.

## The exact workload

The compute source (`kernels/compute/mm.cpp`) uses `Mt=Kt=Nt=20`, two input CBs
(`c0`, `c1`) and one output CB (`c16`). It acquires Dst, performs 20
`cb_wait_front(c0/c1) + matmul_tiles + cb_pop_front(c0/c1)` operations per
output tile, then commits/waits Dst, reserves `c16`, packs one tile, pushes it,
and releases Dst. There are 400 output tiles and 8,000 source-level matmul
operations.

The generated kernels contain no scalar `mul`, `div`, `amo`, or `fence`; their
scalar instructions are ALU, branch, `lw`/`lhu`, `sw`/`sh`, and calls in the
prologue. The important dynamic counts are:

| thread | static constructs | dynamic meaning |
| --- | --- | --- |
| TRISC0 / unpack | 29 literal `.ttinsn`, `TTREPLAY 0,12,0,1` | records 12 words with no execution, then emits UNPACR/config work while polling input CBs and local state |
| TRISC1 / math | 41 literal `.ttinsn`, `TTREPLAY 16,16,0,1` | records 16 MVMUL words; the MOP loop issues `20*20*20=8,000` MOP pushes |
| TRISC2 / pack | 36 literal `.ttinsn` | polls output CB, emits PACR/MOP work, and pushes 400 packed tiles |

For the current HiFi4 MOP configuration, one math MOP expands to four Replay
operations, each replaying 16 MVMUL words. Thus the math stream contains
`8,000*4*16=512,000` MVMUL backend instructions. This count must come from
decoding the MOP configuration and the instruction words; it should not be
hard-coded for other compile-time arguments or fidelity settings.

## What LLVM-MCA can own

The existing `-mcpu=tt-bh` model is the right scalar baseline. It models the
Blackhole Baby-RISC single-issue EX1 path, register dependencies, the scalar
EX2/LSU resources, and the documented scalar latencies. It should remain a
scalar model. Making a RISC-V `sw` have the latency of an MVMUL would make the
model less meaningful.

Plain llvm-mca cannot provide the complete result:

* It analyzes a static instruction sequence and does not execute the branch
  path or polling loop.
* It has no architectural register values or effective addresses, so it cannot
  tell a CB status load, PCBuf TTSync load, MOP configuration load, local RAM
  load, or an L0 hit apart.
* A normal MCA run resets its scheduling state at each region/run; splitting a
  kernel at every event loses register readiness, outstanding loads/stores,
  cache state, FIFO occupancy, and branch state.
* TableGen has no representation of CB counters, semaphores, Dst ownership,
  MOP/Replay expansion, Wait Gates, or Backend completion.

Therefore the production tool should be a `trisc-mca` driver using LLVM-MCA
libraries and the `tt-bh` schedule model, rather than three independent CLI
runs over the source files.

## Proposed state machine

Run one global cycle clock and one persistent state per TRISC thread. The three
threads share the Tensix frontend/backend state for the tile, while each
thread has its own scalar Baby-RISC pipeline and instruction FIFO.

```text
RV32 functional trace / event input
              |
              v
       scalar MCA scheduler  <---->  Baby-RISC state
              |
              v
       TRISC push/FIFO model
              |
              v
       MOP expander -> Replay expander -> Wait Gate
              |
              v
       UNPACK / MATH / PACK / Sync / Dst model
              |
              v
       CB visibility events and output event trace
```

### Inputs

The input contract should use visibility semantics, not raw NoC completion:

```json
{
  "initial_cycle": 0,
  "initial_cb": {"c0_front": 0, "c1_front": 0, "c16_free": 2},
  "cb_events": {
    "c0_push_visible": [/* cycle, count */],
    "c1_push_visible": [/* cycle, count */],
    "c16_space_visible": [/* cycle, count */]
  },
  "flags": {"gathering": false, "auto_ttsync": false}
}
```

If only a NoC barrier timestamp is available, the conversion to CB visibility
must be explicit and calibrated before it enters this interface. For this
kernel the required events are reader `c0/c1` pushes and writer `c16` pops
(free slots). The model emits compute `c0/c1` pops and `c16` pushes as outputs.

### Dynamic scalar execution

The driver executes the actual generated assembly path with a small RV32
functional interpreter (or Spike behind an adapter). It must resolve register
values, branch outcomes, symbol addresses, and stores to `__instrn_buffer`.
The existing `ttloop/baby_risc.py` register/event machinery is a suitable
starting point, but the TRISC model needs a real memory map for CB overlays,
PCBufs, semaphores, MOP configuration, GPR/configuration registers, and the
instruction FIFO.

For each scalar instruction, lower an MC instruction using the `tt-bh` model
and preserve the MCA dependency/resource state across the entire dynamic
trace. The scalar timestamp is the time at which the instruction enters EX1;
LSU request emission, response readiness, store retirement, and MMIO/FIFO
processing are separate timestamps.

`CustomBehaviour::checkCustomHazard` is useful for an already-annotated dynamic
trace: it can delay the next instruction for FIFO, CB, semaphore, or TTSync
state. It cannot choose a branch path, create an expanded instruction, or
preserve state between independent MCA invocations. Those responsibilities
belong to the driver or to a future streaming MCA pipeline extension.

### RISC-V pushes and frontend queues

Decode a static `.ttinsn` as a push of one 32-bit Tensix word. The ISA permits
up to four adjacent `.ttinsn` pushes to fuse in one cycle; the default tt-metal
configuration disables TRISC gathering, so the default run should enqueue one
word per encoded instruction and expose gathering as an input flag.

Classify `sw` to `INSTRN_BUF_BASE` dynamically. It is still a scalar Baby-RISC
store with normal EX1/LSU/store-queue timing, but its value becomes a Tensix
instruction only when the memory request is accepted by the thread's frontend
FIFO. Track at least:

```text
scalar_issue -> store_request_emitted -> FIFO0/FIFO1_accept
             -> expander_input -> backend/WaitGate arrival
```

Blackhole's first frontend FIFO has physical capacity 32 but normal acceptance
must fall back to occupancy 28; a full FIFO automatically stalls the RISC
core. Track downstream MOP/Replay FIFO occupancy separately. The current p150a
probe observes an eight-word Replay-side FIFO, but that depth is a device
parameter rather than a scalar LLVM scheduling resource. Do not use the `sw`
EX1 cycle as a backend issue or completion cycle.

### MOP and Replay

Use the Blackhole `assembly.yaml` decoder (or generated tables from it), not a
hand-maintained list of decimal `.ttinsn` values. At minimum decode MOP,
MOP_CFG, REPLAY, RESOURCEDECL, UNPACR, PACR, MVMUL, ZEROACC, SEMPOST,
SEMGET, SEMWAIT, and STALLWAIT.

Model the documented asynchronous behavior:

* `TTREPLAY index,count,0,1` records the next `count` words into the 32-word
  replay buffer and emits none of them to the backend.
* A replay execution emits one recorded word per cycle and blocks new input
  during the remaining expansion, except when downstream backpressure applies.
* A MOP expands according to its per-thread MOP configuration. Expansion emits
  at most one word per cycle; a non-MOP transition has a one-cycle penalty.
  A MOP whose body emits Replay must still account for Replay expansion.
* MOP/Replay resource declarations apply to the parent instruction for
  Auto-TTSync; the generated child words do not independently repair a missing
  declaration.

### Wait Gate, semaphores, and Manual TTSync

Decode block and condition masks. `SEMWAIT`/`STALLWAIT` latch a per-thread
Wait-Gate condition; selected instructions stop at the gate until all selected
conditions are satisfied, with the documented one-cycle release lag. Model the
Sync Unit's one-cycle semaphore instruction latency and one semaphore
instruction per cycle throughput.

The PCBuf sequences in these kernels are real scalar loads:

```asm
sw x0, (pc_buf + 4 or 8)
lw t0, (pc_buf + 4 or 8)
and x0, x0, t0
```

`+4` waits for all in-flight instructions of the current thread; `+8` waits
only for MOP expansion/idle. The load may start before the condition is true,
but its result is not ready until the condition is met, and the dependent ALU
instruction must then issue. `+8` must not be implemented as backend
completion.

### Backend and Dst state

The shared tile state needs independent queues/scoreboards for UNPACK0/1,
MATH, PACK, configuration, Sync, and outstanding memory operations. Each
operation has separate `accepted`, `started`, and `completed` timestamps. The
ISA documents Sync-unit timing and expansion rates, but exact Blackhole
UNPACR/MVMUL/PACR latency and cross-thread arbitration should be calibration
parameters, not guessed scalar latencies.

Track SrcA/SrcB ownership, the math-to-pack semaphore, and Dst ownership. A
Dst write blocks Matrix Unit or PACR reads of the same aligned 8x16 block for
the next four cycles. Decode the address modifiers and `DstSync::SyncHalf`
offsets so the scoreboard uses physical blocks; do not reduce this rule to a
single global Dst busy bit. `ZEROACC` and Dst release must update the same
ownership/validity state used by subsequent MVMUL and PACR operations.

### Circular buffers

Maintain CB front/back counters and capacities (two tiles for each CB in this
example). An external reader push or writer pop changes the corresponding
visibility/credit counter at its supplied cycle. A TRISC polling load sees the
new value only after its ordinary address-region load timing, so a condition
becoming true at cycle `t` does not imply that the branch exits at `t`.

`cb_wait_front` therefore blocks the dynamic scalar path until the first
polling iteration whose load result and compare are ready. `cb_reserve_back`
uses the writer-provided free-slot events. `cb_pop_front` and `cb_push_back`
update CB state only at the appropriate store/Backend visibility boundary.

## Cycle ordering

Use a fixed, documented per-cycle order so equal-cycle events are reproducible:

1. Apply external CB events and complete scalar loads/backend operations.
2. Re-evaluate Wait Gates, semaphores, Dst hazards, and TTSync requests.
3. Drain expander/FIFO stages into shared backend queues.
4. Let each Baby-RISC scalar pipeline issue at most one instruction, subject to
   MCA dependencies, LSU/store-queue state, FIFO acceptance, and Wait-Gate
   blocks.
5. Retire scalar instructions and publish CB/backend events scheduled for the
   next cycle.

### Cross-thread arbitration does not exist to be modelled

Do **not** plan a cross-thread arbiter as a free knob. The ISA documents no
hardware arbiter that serializes backend units between threads. What it does
document (`WormholeB0/TensixTile/TensixCoprocessor/README.md`) is:

* The Matrix Unit "can only start executing one instruction per cycle,
  regardless of how many threads are trying to use it." That is an issue rate —
  a hardware constant, not a policy.
* The rest is a **software contract, not arbitration**: "The majority of backend
  state exists just once... this is the case for `Dst` and for `LReg`; software
  needs to carefully manage concurrent access to these memories, as otherwise
  threads can overwrite each other's data." For `SrcA`/`SrcB` the README is
  explicit — software manages the copy flip "along with ensuring that each
  relevant backend execution unit is only in use by one thread at a time."

Violating that contract produces **data corruption, not a delay**, so there is
no timing curve a knob could fit. Model it as a checked precondition
(scoreboard violation reported as an error), and keep only two genuine
calibration parameters: the Matrix Unit's 1-instruction-per-cycle start rate and
per-unit startup / steady-state periods.

This also settles what MCA may own. The three threads reach the backend through
**three independent frontends** and the RISCs run "completely asynchronously" to
the coprocessor, so the connection is not three cores contending for one
resource — it is three push/FIFO streams feeding a shared asynchronous backend.
MCA has no representation for that shape at any layer; see
[`streaming-api-and-extension-surface.md`](streaming-api-and-extension-surface.md)
("Sharing a backend across pipelines") for the exact blockers.

## LLVM implementation plan

1. **Trace IR and decoder.** Parse the generated `.S`, preserve source PC and
   labels, evaluate RV32 values, and emit a dynamic trace with typed push,
   MMIO, CB, TTSync, and backend annotations. Add a decoder generated from
   Blackhole `assembly.yaml`.
2. **Persistent scalar MCA.** ~~Add a streaming source/step API to LLVM-MCA~~
   **Correction: the streaming source already exists upstream.**
   `llvm/include/llvm/MCA/IncrementalSourceMgr.h` is documented as a `SourceMgr`
   that adds instructions incrementally, and `EntryStage::getNextInstruction()`
   returns `InstStreamPause` when the stream is open but empty
   (`Stages/EntryStage.cpp:33-35`); `Pipeline::runCycle()` turns that into
   `State::Paused` (`Pipeline.cpp:71-74`) with every stage, the PRF, the
   `ResourceManager` and the `LSUnit` left alive.  Upstream's own test is
   `llvm/unittests/tools/llvm-mca/X86/TestIncrementalMCA.cpp`.  What is actually
   missing is only: a public `runCycle()`/`step()` and a `getCycles()` accessor
   (`Pipeline.h:65,67` are private) -- a ~10-line patch.  A driver can also
   assemble its own pipeline out of tree, since `EntryStage`,
   `InOrderIssueStage`, `RegisterFile`, `LSUnit` and `Pipeline::appendStage` are
   all public; that is where a custom `LSUnitBase` or `CustomBehaviour` goes.
   External stalls need no patch at all: `CustomBehaviour::checkCustomHazard`
   has exactly one call site (`Stages/InOrderIssueStage.cpp:136`) and returning
   1 re-probes every cycle, i.e. block-until-external-event.  Keep
   address-dependent load readiness and the Baby-RISC retire/store queues in
   this layer.
3. **Frontend model.** Implement static `.ttinsn`, dynamic instruction-buffer
   stores, FIFO thresholds, gathering, MOP_CFG, MOP, and Replay.
4. **Tensix state model.** Implement Wait Gate, Sync/semaphore handoff,
   Src/Dst ownership, Dst block hazards, and calibrated UNPACR/MVMUL/PACR
   timing.
5. **Reports.** Emit per-thread scalar issue, FIFO acceptance, expander,
   backend start/complete, TTSync release, CB pop/push, and final TRISC-exit
   cycles. These are the events needed by the global scheduler.

Adding only a `RISCVCustomBehaviour` hook is insufficient: it can stall an
already lowered instruction but cannot supply dynamic instructions or the
persistent cross-thread state. A custom pseudo-opcode/parser is useful for
reporting and for static `.ttinsn`/`TTREPLAY`, but it is not a replacement for
the event driver.

## Calibration and validation

Use separate no-NoC microkernels to fit backend timing before validating the
full example:

* record and replay 1, 2, ..., N MVMUL/UNPACR/PACR operations and fit startup,
  latency, and steady-state throughput;
* sweep FIFO occupancy and adjacent `.ttinsn` count to confirm 32/28 behavior
  and gathering mode;
* use Manual TTSync `+4` and `+8` around isolated streams to separate backend
  completion from expander drain;
* alternate Dst blocks at distances 1..5 to verify the four-cycle hazard and
  address remapping;
* validate CB polling with external visibility events at times that fall just
  before and after a polling load.

Validate in this order: `Mt=Nt=Kt=1`, then one output row with `Kt=20`, then
`20,20,20`. Compare event timestamps, not only final runtime. The useful error
breakdown is scalar MCA, scalar+CB, scalar+FIFO/expander, and full Backend/Dst
model.

Sources: [Blackhole Baby-RISC README](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/README.md),
[PushTensixInstruction](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/PushTensixInstruction.md),
[ManualTTSync](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/ManualTTSync.md),
[MOP expander](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/MOPExpander.md),
[Replay](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/REPLAY.md),
[Dst](../tt-isa-documentation/BlackholeA0/TensixTile/TensixCoprocessor/Dst.md),
[Sync Unit](../tt-isa-documentation/BlackholeA0/TensixTile/TensixCoprocessor/SyncUnit.md),
and the generated files under `example/standalone/matmul_single_core/asm_kernel/`.
