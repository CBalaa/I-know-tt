# Measuring Matmul CB Visibility With Device Zones

Observed on tt-metal `4f9fa9e0`, Blackhole p150a.  The standalone
`example/standalone/matmul_single_core` has an opt-in
`MATMUL_SINGLE_CORE_CB_PROFILE_KERNELS` build mode.  It uses copies of the
reader and writer C++ kernels under `kernels/profile/`; the normal kernels and
the generated `asm_kernel/*.S` are unchanged.

The reader wraps the first 16 `cb_push_back` calls for CB0 and CB1 with
`DeviceZoneScopedN("CB0_PUSH_BACK")` and `DeviceZoneScopedN("CB1_PUSH_BACK")`.
Each zone is entered after `noc_async_read_barrier()`.  The writer wraps the
first 16 CB16 `cb_pop_front` calls with
`DeviceZoneScopedN("CB16_POP_FRONT")`, after `noc_async_write_barrier()`.
With `TT_METAL_DEVICE_PROFILER=1`, call
`ReadMeshDeviceProfilerResults` after the blocking output read; the legacy
profiler CSV then contains begin/end pairs for these zones on BRISC/NCRISC.

The zone **end** timestamp is the CB event boundary used by the extractor: for
the reader, the read response has completed before `cb_push_back`, so CB0/CB1
become visible to the compute consumer at or immediately after that timestamp.
For the writer, the output write has completed before `cb_pop_front`, so the
CB16 slot is released at or immediately after its timestamp.  These are
visibility/release events, not NoC request issue timestamps or a complete
kernel duration.

`extract_cb_events.py` parses the profiler preamble/header, filters core `(0,0)`
and `ZONE_END` rows, subtracts the corresponding `TRISC-KERNEL` start (and
adds the two initial CB16 free slots), and writes `c0_push_visible`,
`c1_push_visible`, and `c16_space_visible` arrays for `predict_trisc.py`.
Missing zones indicate that profiling was not enabled at runtime or that the
CSV came from another core.  Keep the probe to 16 events:
the legacy profiler has a small per-RISC zone budget and silently drops later
markers.

References:

* `tt_metal/docs/source/tt-metalium/tools/device_program_profiler.rst`
* `tt_metal/tools/profiler/kernel_profiler.hpp`
* `example/standalone/matmul_single_core/README.md`
