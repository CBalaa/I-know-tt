# Single-core matmul CB zones perturb the trace, and endpoint interpolation leaks the answer

Observed on tt-metal `4f9fa9e0`, Blackhole p150a, 2026-10-06. The standalone
`matmul_single_core` ran the same generated TRISC assembly in both arms. The
reader/writer were built from the same probe C++ sources with either 0 or 16
zones per CB stream, and the legacy device profiler was enabled in both arms.

**Measured:** Eight alternating 0-zone/16-zone pairs each ran successfully.
Every 16-zone run had exactly 16 begin/end pairs for each of `CB0_PUSH_BACK`,
`CB1_PUSH_BACK`, and `CB16_POP_FRONT` (48 zones total); every 0-zone run had
none. TRISC0 `TRISC-KERNEL` durations differed by these cycles, 16 minus 0:

`6488, 21773, 20177, 16096, 15609, 15732, 19236, 10643`

Mean difference: **15,719 cycles (0.172% of ~9.14 million)**; median 15,914,
range 0.071–0.238%. These numbers include marker work, the probe's conditional
branch/counter code, and run-to-run variation. They do **not** isolate the
marginal cost of a `DeviceZoneScopedN` call (measured separately as ~51 cycles
in [`measuring-device-instrumentation-cost.md`](measuring-device-instrumentation-cost.md)).
The 48 calls alone would contribute only ~2,450 RISC cycles if all were on the
critical path. This controlled comparison does not support attributing a 5%
end-to-end error to this probe.

**Source-verified and measured:** The old extractor used the *same instrumented
NCRISC/BRISC kernel endpoint* to estimate all the unobserved CB event times.
In a prior trace, the instrumented NCRISC endpoint relative to TRISC0 start was
9,095,988 cycles; the reported simulated total was 9,097,754, only 1,766
cycles later. The old uninstrumented reference was 9,051,869 cycles, so 44,119
of the 45,885-cycle reported difference was already present in the supplied
endpoint. This comparison used runs from different dates and is not itself a
probe-overhead measurement; it reveals **answer leakage**, not independent
predictive accuracy. The old endpoint-derived period was 1137.0686 cycles;
rounding it once to 1137 accumulates about **548 cycles** over the 7,984
extrapolated intervals, not ~44,000. The README claim that a 5.52-cycle
rounding error per tile explains the total is incorrect.

The corrected extractor requires a separate **zero-zone** endpoint to extend
past the observed 16 events and rounds each interpolated event independently.
For one pair it produced a 9,142,756-cycle replay versus the zero-zone TRISC0
duration of 9,142,748 (+8). That close agreement is expected: the replay's
tail is anchored to the measured zero-zone endpoint. It is useful for checking
the simulator's event-consumption logic, but it is **not** evidence that the
NoC or backend was predicted to within 8 cycles. A kernel endpoint is later
than the last CB operation and is only an upper-bound proxy for it.

Raw CSVs, run logs, and a short analysis live under `/tmp/ttloop_probe_audit/`
on the measurement machine. Reproduce the alternating builds and extraction
with the standalone example's `README.md` CB/NOC visibility section.
