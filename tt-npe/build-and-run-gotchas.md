# tt-npe — build and run gotchas

**Observed on:** `tt-npe` @ `0d2d4cc26575edae6e2729a4c2ac88f0ba17792e`, Ubuntu, Clang-20,
inside an **active conda env** (`miniconda3/envs/wallfacer`, Python 3.11).

## Finding 1 — the build fails in a conda env; you must strip the injected flags

Running upstream `./build-npe.sh` verbatim fails at the **last link step**:

```
clang++-20: error: linker command failed with exit code 1
undefined reference to `std::__1::basic_string<...>::append(char const*, unsigned long)'
... (hundreds of std::__1:: symbols from the test objects)
```

Everything *compiles* (132/132 objects); only the `tt_npe_ut` **executable** fails to link.

**Root cause — two stacked problems:**

1. Conda activation injects `LDFLAGS` containing `-Wl,--as-needed` **and**
   `-L$CONDA_PREFIX/lib` (a directory that has `libstdc++` but **no** `libc++`).
2. Upstream `CMakeLists.txt:66-67` adds `-stdlib=libc++` via `add_compile_options`
   — which applies to **compilation only, not linking**. Line `CMakeLists.txt:78-80`
   then compensates by putting `-lc++ -lc++abi` into `CMAKE_EXE_LINKER_FLAGS`,
   which places them **before the object files**. Under `--as-needed` the linker
   decides they are unused and drops them.

Net effect: objects are built against **libc++** (`std::__1::` mangling) but the
link resolves against **libstdc++** → undefined references.

The shared libraries (`libtt_npe.so`, `tt_npe_pybind*.so`) link "successfully"
anyway because shared objects permit undefined symbols by default
(`-Wl,--allow-shlib-undefined` is also in conda's LDFLAGS) — so the failure only
surfaces on the executable, which is misleading.

**Fix — sanitize the environment (no source edits needed):**

```bash
cd thirdparty/tt-npe
env -u LDFLAGS -u CFLAGS -u CXXFLAGS -u CPPFLAGS -u CMAKE_ARGS ./build-npe.sh
```

Wrapped as `./tools/npe_build.sh` in the consuming repo. **Measured:** exit 0,
132/132 targets, `install/` populated.

### "Just install libc++ into conda" does NOT fix this — measured

This is the natural first guess and it is **wrong**. Two independent facts:

1. **libc++ was never missing.** The original failing log contains **414**
   `undefined reference` lines and **0** `cannot find` lines — the linker *found*
   `-lc++` and then **discarded** it. The system has a complete libc++
   (`/usr/lib/llvm-20/include/c++/v1`, `/usr/lib/x86_64-linux-gnu/libc++.so ->
   ../llvm-20/lib/libc++.so`).
2. **Making libc++ findable changes nothing.** Re-linking the real
   `tt_npe_ut` objects, each case actually relinked:

   | case | link flags | result |
   |---|---|---|
   | baseline | (sanitized, no `--as-needed`) | **exit 0** |
   | A | `-Wl,--as-needed -L$CONDA_PREFIX/lib` (upstream ordering) | **exit 1**, 1 undefined ref |
   | B | A **+ explicit libc++ `-L` paths** (= "conda now has libc++") | **exit 1**, 1 undefined ref |
   | C | A + `-stdlib=libc++` **on the link line** | **exit 0** |

   Case **B is the decisive one**: hand the linker every libc++ search path you
   like and it still fails, because `--as-needed` drops `-lc++` for appearing
   *before* the objects. Minimal repro agrees (`clang++-20 -Wl,--as-needed -lc++
   -lc++abi t.o` fails; `clang++-20 -Wl,--as-needed t.o -lc++ -lc++abi` succeeds).

**conda-forge does ship a `libcxx` for `linux-64`** (6.0.1 … 23.1.1, verified via
`conda search --info`), so it *is* installable — it simply does not address the
cause. Worse, it would be actively risky: `-L$CONDA_PREFIX/lib` is searched
**first**, so a conda libc++ (v23) would satisfy `-lc++` ahead of the system
libc++ **v20** that clang-20's headers were compiled against. *Reasoned risk, not
measured* — I did not install it.

### The proper upstream fix (if you ever patch tt-npe)

Put the stdlib on the **link** options so the clang driver inserts libc++ at the
correct position *after* the objects, instead of hand-placing `-lc++ -lc++abi`:

```cmake
add_link_options($<$<LINK_LANG_AND_ID:CXX,Clang>:-stdlib=libc++>)
# or, per-target:
target_link_options(tt_npe_ut PRIVATE -stdlib=libc++)
```

Case C above proves this works even with `--as-needed` present and upstream's
redundant `-lc++ -lc++abi` still on the line. Upstream's commented-out block at
`CMakeLists.txt:68-71` was reaching for exactly this.

> Also note `build-npe.sh` infers `ROOT` via `git rev-parse --show-toplevel`, so you
> **must `cd` into the submodule first**. Invoking it by absolute path from the outer
> repo makes it resolve the *outer* root and cmake fails with
> "does not appear to contain CMakeLists.txt".

`ENV_SETUP` additionally calls `uv pip install -r requirements.txt`; `uv` is not
installed here, so source it and then set the paths manually:

```bash
export PYTHONPATH="$PWD/install/lib:$PWD/install/bin"
export PATH="$PWD/install/bin:$PATH"
```

## Finding 2 — the C++ test binary is CWD-sensitive

`./build/tt_npe/tt_npe_ut` **must be run from `tt_npe/`**, not from `build/` or the
repo root. Tests reference data via relative paths like
`cpp/test/data/multichip-trace-example.json`.

- From `build/`: **2 failures** — `npeAPITest.ValidatesMulticastUtilizationFromTrace`
  and `npeWorkloadTest.CanIngestAndValidateMultichipTraceFile`
  (`Provided input file 'cpp/test/data/multichip-trace-example.json' is not a valid file!`).
- From `tt_npe/`: **132/132 pass**.

`tt_npe/scripts/run_ut.sh` does the right thing (it `cd`s to `$ROOT/tt_npe/` first),
which is why it reports PASS. Don't conclude the binary is broken from a wrong-CWD run.

## Finding 3 — both shipped examples are broken at this commit

**`tt_npe.py` CLI** — crashes *after* a successful simulation
(`tt_npe/py/pycli/tt_npe.py:227`):

```
AttributeError: 'tt_npe_pybind.Stats' object has no attribute 'wallclock_runtime_us'
```

The field exists on `deviceStats` (`bindings.cpp:50`, `npeStats.hpp:57`), but the CLI
reads it off the **`Stats` container**. Fix: read
`result.per_device_stats[-1].wallclock_runtime_us`.
Workaround: `./tools/npe_run.py <workload.json>`.

**`tt_npe/py/examples/programmatic_workload_generation.py`** — crashes with an
uncaught C++ exception before producing results:

```
terminate called after throwing an instance of 'boost::wrapexcept<std::out_of_range>'
  what():  key was not found in unordered_flat_map
```

Two independent bugs: (a) it never calls `setGoldenResultCycles()`, and (b) it has the
same `wallclock_runtime_us` bug as the CLI.

## Finding 4 — `setGoldenResultCycles()` is effectively mandatory for programmatic workloads

`npeStats::insertTimestep` calls `wl.getGoldenResultCycles(device_id)` for **every**
device id **and** `MESH_DEVICE` (`npeStats.cpp:44-46`), and that accessor is
`golden_cycles.at(device_id)` (`npeWorkload.hpp:111`). An empty map throws
`std::out_of_range`, which is **not** an `npeException` and is **not** caught by
`npeAPI::runNPE` — so it terminates the process instead of returning an error.

Always set both the real device id and `-1`:

```python
wl.setGoldenResultCycles({0: (0, 450), -1: (0, 450)})
```

The JSON workload path avoids this via the optional `golden_result.cycles` field.

## Measured test results (evidence)

| suite | command | result |
|---|---|---|
| Python | `cd tt_npe && pytest -q` | **18 passed** (9.25 s) |
| C++ | `cd tt_npe && ../build/tt_npe/tt_npe_ut` | **132 tests / 11 suites, 132 passed** (8.7 s) |

The `E:` lines on stderr during the C++ run are expected output from negative-path
tests (invalid coords, unknown device, bad files) — not failures.

## Golden example baseline

`tt_npe/workload/example_wl.json` (1 phase, 3 transfers, `golden_result.cycles = 450`):

```
estimated cycles: 437    golden cycles: 450    cycle pred error: -2.9%
avg Link util: 4.1%      max Link demand: 99.7%      congestion impact: 0.0%
```

`congestion impact 0.0%` despite `max Link demand 99.7%` is correct model behaviour —
the derate only engages above 100% demand. See `prediction-model.md`.
