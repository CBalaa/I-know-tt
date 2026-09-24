# The tt-metal runtime tree is missing three device-profiler headers

**Finding.** The install/runtime tree that tt-metal JIT-compiles device kernels against
is **missing three headers** from `tt_metal/tools/profiler/`. Any JIT that pulls in the
NoC-event profiler fails to compile, and the failure surfaces as a host-side abort rather
than a build error.

Observed on tt-metal `4f9fa9e0`, arch **blackhole** (p150a), runtime tree
`thirdparty/tt-metal/build_Release/libexec/tt-metalium/` (gitignored, `build_*`).

## The three files

Comparing source against the runtime tree:

| header | in `tt_metal/tools/profiler/` (source) | in `.../libexec/tt-metalium/tt_metal/tools/profiler/` |
|---|---|---|
| `event_metadata.hpp` | yes | **missing** |
| `kernel_profiler_streaming.hpp` | yes | **missing** |
| `noc_event_profiler_utils.hpp` | yes | **missing** |

## Symptom

Running with `TT_METAL_DEVICE_PROFILER_NOC_EVENTS=1` aborts the host process:

```
tt_metal/tools/profiler/noc_event_profiler.hpp:12:10: fatal error: event_metadata.hpp: No such file or directory
compilation terminated.
 (assert.hpp:104)
Test failed with exception!
TT_THROW @ tt_metal/jit_build/build.cpp:149: tt::exception
terminate called after throwing an instance of 'std::runtime_error'
```

The abort comes from the **ncrisc** firmware compile (`-DCOMPILE_FOR_NCRISC`,
`-DPROFILE_NOC_EVENTS=1`), i.e. it fails before any kernel of yours is built. Note the
trigger is `PROFILE_NOC_EVENTS`, which tt-metal sets from
`TT_METAL_DEVICE_PROFILER_NOC_EVENTS=1` — so **plain `TT_METAL_DEVICE_PROFILER=1` runs
fine** and only NoC-trace collection breaks. That makes it look like a NoC-tracing bug
rather than a missing-file problem.

## Fix

Copy the three headers from the source tree into the runtime tree:

```bash
cd thirdparty/tt-metal
SRC=tt_metal/tools/profiler
DST=build_Release/libexec/tt-metalium/tt_metal/tools/profiler
for f in event_metadata.hpp kernel_profiler_streaming.hpp noc_event_profiler_utils.hpp; do
    cp "$SRC/$f" "$DST/$f"
done
```

`build_Release/` is gitignored (`.gitignore:17: build_*`), so this does not dirty the
submodule. After copying, the NoC trace collection run completes and writes
`noc_trace_dev0_ID0.json` + `topology.json`.

## Related

`out-of-tree-program-cmake-and-runtime-root.md` covers *finding* the runtime tree; this
note covers a file that is missing **inside** it. Both bite the same workflow.
