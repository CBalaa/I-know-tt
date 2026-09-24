# Baby-RISC timing model used by `add_2_integers_in_riscv`

This note records the small timing core in `ttloop/baby_risc.py`.  It is a
timing layer for the scalar Blackhole Baby RISC-V pipeline, not an ISA
interpreter.  The assembly parser supplies enough register values to resolve
branches and addresses; a real RV32 functional engine can replace that part
without changing the timing state or event interface.

## What is modelled

The state persists across basic blocks and asynchronous waits:

* one instruction per cycle into in-order EX1;
* the eight-entry retire-order queue and four-entry store queue;
* forwarding dependencies, L0 data cache (four 16-byte lines), and the
  documented address-region load latencies (including the local-RAM two-cycle
  class);
* `fence` draining the LSU and flushing L0;
* persistent load/store queues (the first version uses emitted-store ordering
  rather than a full banked memory-system model); and
* separate NoC events: RISC store, request emitted, NIU command accepted,
  response/ack received, and barrier release.

The NoC service parameters are experiment inputs, not ISA constants.  The
checked-in defaults (`read_response_latency=400`, `write_ack_latency=265`,
three cycles to command acceptance) are inferred values tuned so the model
lands near the recent hand-written assembly measurements documented in
[`add2-noc-barrier-semantics.md`](add2-noc-barrier-semantics.md).  Override
them for another route, DRAM state, or board.  The read barrier samples varied
by roughly 478--534 cycles and the uninstrumented write zone was about 350
cycles (NOC-event instrumentation raised its raw window to roughly 370--372),
so a single universal NoC latency would be misleading.
The three-cycle acceptance value is also a model assumption; the existing
profiler evidence does not isolate the `NOC_CMD_CTRL` write from the command
accept event.

## Running the checked-in example

From the project root:

```bash
PYTHONPATH=. python3 example/predict_add_2_integers_in_riscv/predict.py
```

The same API can be called directly when supplying a disassembled JIT
`brisc.elf` or changing the route calibration:

```python
from ttloop.baby_risc import EventKind, predict_add2

core, events = predict_add2()
for event in events:
    if event.kind in {
        EventKind.NIU_COMMAND_ACCEPTED,
        EventKind.READ_RESPONSE_LANDED,
        EventKind.WRITE_ACK_RECEIVED,
        EventKind.BARRIER_RELEASED,
        EventKind.CORE_EXIT,
    }:
        print(event.kind.value, event.cycle)
print("cycles", core.state.cycle)
```

With the documented branch recovery cost enabled, the deterministic default
run (the checked-in hand-written assembly path) currently reports.  The chosen
backward-taken policy charges five outcome transitions in this program; the
hardware's initial predictor state is not documented.

```text
read command accepted       55, 86
read response landed       455, 486
read barrier released      500
write command accepted     574
write acknowledgement      839
write barrier released     850
core exit                  865
```

The recent measured hand-written assembly path is 861--863 net cycles (an
earlier run was 854).  To predict the C++ JIT binary rather than this
checked-in assembly, pass an instruction dump extracted from its `brisc.elf`
to `load_assembly`; that is a different instruction stream.  The default's
400/265-cycle service values
are calibration inputs, not a claim that the NoC has fixed latency.
`llvm-mca` remains the static
straight-line baseline, while this model is the persistent, target-specific
backend that can represent polling and asynchronous completion.

## Event boundary to use in a scheduler

Use `NIU_COMMAND_ACCEPTED` when creating an external NoC/NPE token.  Use
`BARRIER_RELEASED` only when modelling the Baby-RISC core becoming runnable
again.  A store entering EX1 or a profiler `READ` record is earlier than both
events and must not be used as the token timestamp.

Sources: Blackhole Baby-RISC
[`README.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/README.md),
[`MemoryOrdering.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/MemoryOrdering.md),
and the NoC request lifecycle in
[`NoC/MemoryMap.md`](../tt-isa-documentation/BlackholeA0/NoC/MemoryMap.md) and
[`NoC/Counters.md`](../tt-isa-documentation/BlackholeA0/NoC/Counters.md).
