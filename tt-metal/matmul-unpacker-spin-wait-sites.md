# Matmul single-core unpacker spin waits

On tt-metal `4f9fa9e0` / Blackhole, the generated `mm_trisc0_unpack.S` has
three kinds of BabyRISCV polling loops in the matmul kernel body:

* the initial `pc_buf_base + 52` loop (`sem5`, `UNPACK_SYNC`) waits for the
  unpacker semaphore to reach zero;
* the `0xFFB40000 + 40` and `0xFFB41000 + 40` loops poll the received counters
  for CB0 and CB1 respectively (`cb_wait_front`);
* the later `pc_buf_base + 52` loop with `andi ...,254` waits for
  `UNPACK_SYNC < 2`, allowing another unpack configuration context.

The source for the latter loop is `wait_for_next_context(2)` in
`tt-llk_blackhole/common/inc/cunpack_common.h:162-168`; the initial loop is
`wait_for_idle()` at lines 191-197. `kernels/compute/mm.cpp:60-62` calls the two
CB waits. The loops appear at approximately lines 25-28, 259-266, and 269-272
in the generated assembly.

`llk_io_unpack.h:18-32` wraps each `llk_wait_tiles` call in
`DeviceZoneScopedSumN1("CB-COMPUTE-WAIT-FRONT")` when profiling is enabled.
`profileScopeAccumulate` only accumulates the total duration in `sums[]`; it
does not emit a timestamp per CB wait, and it does not cover
`wait_for_next_context`.

For per-event instrumentation, capture the wall clock immediately after each
wait returns (and once at the beginning of `kernel_main`) rather than reading
the clock in the polling body. `ckernel::read_wall_clock()` is available from
`tt-llk_blackhole/common/inc/ckernel.h:589-594`; the generic
`get_timestamp_32b()` is in `internal/tt-1xx/risc_common.h:243-246` but is not
included by the default ckernel headers. The timestamp stream needs a separate
L1/profiler export buffer; the 16-bit debug mailbox helpers are not a suitable
32-bit cycle trace.

The example implementation defines cycle zero in the unpack TRISC startup just
before `run_kernel()`. This includes the generic wrapper setup, including
`zerosrc()`, and therefore aligns the relative timestamps more closely with the
`TRISC-KERNEL` window. The hook is guarded by `UCK_CHLKC_UNPACK` and the example
compile define, so ordinary compute kernels are unchanged.

On Blackhole p150a, a 640x640x640 run with the trace enabled recorded 24,001
events (one startup idle wait and 8,000 each of CB0, CB1, and context waits).
With the device profiler enabled, two trace-off runs measured 9,051,041 and
9,047,658 TRISC0 cycles; two trace-on runs measured 9,057,929 and 9,053,111.
The paired marginal costs were 6,888 and 5,453 cycles (0.076% and 0.060%), or
6,171 cycles / 0.068% on average for 24,000 per-wait records. The trace's own
final timestamp was 32 cycles before the profiler's kernel-end marker in both
runs. This is an A/B result for this binary and workload, not a portable
per-record constant; use a matched trace-off run to calibrate the overhead
before correcting predictions. After moving the origin into the wrapper, a
profiler-enabled run reported 9,057,918 TRISC0 cycles and the trace final word
reported 9,057,887 cycles, a 31-cycle endpoint difference.
