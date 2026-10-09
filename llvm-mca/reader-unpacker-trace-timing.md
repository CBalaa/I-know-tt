# Reader/unpacker timing with trace-supplied hardware events

**Finding (source-inspected and frontend-run, not hardware-measured).** On
2026-10-09, using the consuming project's `matmul_single_core` Blackhole
assembly generated against tt-metal `4f9fa9e0` with `Mt/Kt/Nt = 20/20/20`,
compile-time reader bounds permit regular scalar loops but do not make the
current `predict.py` a correct two-RISC simulator.

The intended model can time reader NCRISC and unpack TRISC0 from assembly,
while supplying NoC and Tensix backend events from hardware traces. Both
RISC timelines must share CB0/CB1 state: simulated reader pushes make data
visible; simulated unpacker pops return credits. The host allocates two
tiles per input CB, so omitting credit feedback permits invalid producer
run-ahead. Backend acceptance, context release, and completion are distinct;
math-side bank ownership may also block unpack work and must be represented
or supplied as external events.

Fixed absolute event timestamps support replay of a measured run, with a
common clock origin and the observed per-RISC start offsets. To let requests
move with predicted scalar execution, attach measured service intervals to
their matching simulated acceptance/dispatch events. Do not use a residual
polling wait or an entire kernel duration as a device service interval.
Service reuse assumes compatible routes, workload, and resource state;
queue/resource contention must be represented or remain trace-conditioned.
CB push timestamps already contain reader work, NoC time, and backpressure;
in a joint model these are outputs rather than independent fixed inputs.

Current consuming-project implementation gaps:

- `events.py` only queries ready timestamps and supplies CB0/CB1 data and
  CB16 space. NoC, unpack idle/space, and other targets default to ready now;
  there is no issue/dispatch/push/pop publication interface.
- The scalar fast path executes all spin queries before charging all scalar
  work in an iteration. It loses the ordered request/wait/push boundaries
  needed for reader and backend event generation.
- Running `analyze_assembly` on the reader produces an incorrect chain:
  M(dynamic), N(dynamic), K(20), A burst(dynamic), B burst(dynamic). The two
  bursts are actually siblings and are skipped for 2048-byte pages. The CFG
  omits `bleu`; the analyzer forces all natural loops into one nesting chain,
  misses latch-defined constant bounds, and rejects CB reserve polling due
  to its self-destination `add`. The execution fallback defaults unknown
  counts to 20, hiding failed inference. Reader prediction must wait for
  these issues to be resolved.

For why endpoint-derived input events do not independently validate timing,
see [CB trace endpoint leakage](../tt-metal/matmul-cb-probe-overhead-and-endpoint-leakage.md).

## Shared-header transfer and polling cycles

Inspected the current analyzer on the same reader: `_29` itself is recognized
as a natural loop with blocks `{6, 7}`. Block 6 (assembly lines 98-99) is
also recognized as a generic MMIO SpinBlock. Block 7 (100-117) dispatches a
request, advances pointers, and either returns to block 6 or exits. Both the
busy retry (6 -> 6) and next-chunk edge (7 -> 6) target the same header.
The incorrect whole-reader IR is therefore a hierarchy/body construction
failure, not an inability to find this individual cycle.

Standard natural-loop analysis groups backedges by header and unions their
regions. For this CFG, the maximal loop is `{6, 7}` with latches `{6, 7}`;
the poll is not an independent LLVM LoopInfo subloop with the same header.
LLVM's `llvm/docs/LoopTerminology.md:108` explicitly describes logical loops
sharing a header and `LoopSimplify` separating their headers. The performance
IR can use the existing `LoopBlock`, with `SpinBlock(6)` and `LinearBlock(7)`
as its ordered body. Transfer is the loop's purpose, not a new IR node type.
The internal retry edge remains an event wait. Repeated blocks must not also
be charged through a second copy of the polling loop.

For the reader, build the loop forest by node-set containment, preserving
the disjoint A/B transfer loops as siblings inside K. A flat sort by region
size is not a nesting relation; maximal SCCs alone also do not recover
nesting. Induction/trip-count analysis is a separate phase: the dispatch
edge advances `a6` by the loop-invariant `a3 = 16384`, whereas the poll only
reloads external state. Unknown polling counts require event input. The
2048-byte entry guard proves zero full-chunk iterations in this configuration.
Execution must distinguish `trip_count=0` from unknown (`None`); the current
fallback expression `node.trip_count or 20` incorrectly replaces zero with
20. Also, transfer-loop iterations are packets, not matmul tiles; the
current fallback's leaf-loop `tile_idx` increment is not generally valid for
such loops.
