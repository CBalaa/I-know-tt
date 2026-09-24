# Building and running RPM on this box

**Finding.** `bash scripts/build_scripts/build_all.sh` -- the Quick Start in
RPM's README -- does **not** work here. It fails twice, both times for
conda-environment reasons rather than RPM defects, and both times the fix is a
flag RPM already has. The wrapper that does the whole thing is
`tools/rpm_build.sh` in the consuming repo.

Observed on tt-rpm `3094a66` ("Initial public release"), whisper `674f645d`,
Sparta `map_v2` `4145671`, host gcc 14.4 (conda-forge), conda 26.1.

## Blocker 1: the dependency probe, and which env vars it honours

`setup_env.sh` reports three missing libraries on a machine that has two of them:

```text
[MISSING] yaml-cpp        [MISSING] RapidJSON        [MISSING] Boost
```

- yaml-cpp and hdf5 genuinely are not installed system-wide, and there is no
  sudo.
- Boost 1.74 *is* in `/usr/include`, and RapidJSON too. They are reported
  missing because the probe runs a bare `g++ -x c++ -E -`, and **conda's g++
  does not search `/usr/include`** -- it searches its own sysroot:

  ```text
  /home/.../envs/<env>/bin/../x86_64-conda-linux-gnu/sysroot/usr/include
  ```

  conda's activation exports `CPPFLAGS`/`CXXFLAGS` with
  `-isystem $CONDA_PREFIX/include`, but a bare `g++ -E` does not read those
  variables. It does read **`CPATH`**.

So the workable setup is one dedicated conda env plus three exports:

```bash
conda create -y -n rpm -c conda-forge gxx_linux-64=14 cmake ninja make \
    libboost-devel yaml-cpp rapidjson hdf5 sqlite zlib
conda activate rpm
export CPATH=$CONDA_PREFIX/include
export PKG_CONFIG_PATH=$CONDA_PREFIX/lib/pkgconfig
export LIBRARY_PATH=$CONDA_PREFIX/lib
```

`build_all.sh` runs `setup_env.sh` under `set -e`, so a failing probe aborts the
whole build before it starts.

## Blocker 2: whisper links boost statically

`ext/whisper/GNUmakefile` defaults to `STATIC_LINK := 1`, which expands to

```make
LINK_LIBS := $(addprefix -l:lib, $(addsuffix .a, $(BOOST_LIBS))) $(EXTRA_LIBS)
```

i.e. `-l:libboost_program_options.a`. conda-forge's `libboost-devel` ships only
shared objects, so the link stops with:

```text
ld: cannot find -l:libboost_program_options.a
```

The makefile has the switch for this -- `make STATIC_LINK=0` links the shared
library and adds an rpath -- but `build_deps.sh` calls a plain `make` and never
passes it. Build whisper yourself first:

```bash
make -C thirdparty/tt-rpm/ext/whisper -j12 STATIC_LINK=0 BOOST_ROOT="$CONDA_PREFIX"
```

`BOOST_ROOT` matters: it is what puts `-L$CONDA_PREFIX/lib` into `LINK_DIRS`.
After that `build_deps.sh` sees `build-Linux/whisper` up to date and moves on to
the models.

## Blocker 3: Dhrystone dies on a misaligned store

`make -C tests run_dhrystone` deadlocks out of the box:

```text
=== DEADLOCK DETECTED @ cycle 4704 (no retirement for 1000 cycles) ===
false: ROB timeout: ROB is empty — frontend failed to deliver instructions
```

Whisper alone shows what really happened, and it is not the core model:

```text
$ whisper --configfile tests/build/whisper.json --target tests/build/dhrystone.bare.elf ...
Error: Failed stop: Hart 0: 16 consecutive illegal instructions
```

The trace ends on a **misaligned 8-byte store**:

```text
#5195 ... 8000126e e218 c.sd x14, 0x0(x12) [0x80044354] (exception)
```

`0x80044354 % 8 == 4`. `enable_misaligned_data: true` is not enough, because
`Hart.cpp` consults the **PMA attribute** for misaligned stores
(`pma.misalOnMisal()`, `Hart.cpp:13539`). Whisper's default PMA allows
misaligned access everywhere (`PmaManager.hpp:568`), but an explicit `attribs`
list *replaces* that default, and `tests/whisper.json` lists its attributes
explicitly -- without `misal_ok`:

```json
"attribs": ["read", "write", "exec", "amo", "rsrv", "idempotent"]
```

Adding `"misal_ok"` to both PMA entries fixes it. **This is applied in the
submodule**; to see or revert it:

```bash
git -C thirdparty/tt-rpm diff tests/whisper.json
git -C thirdparty/tt-rpm checkout -- tests/whisper.json
```

CoreMark never trips this (it does not misalign its accesses), which is why the
problem only shows up in the Dhrystone half of the test suite.

## Result

| workload | result |
| --- | --- |
| CoreMark, 20 iterations | `Correct operation validated.` -- 6,607,744 instructions, 4,111,373 cycles, IPC 1.61 |
| Dhrystone, 1000 runs | 679 us/run, 1470 Dhrystones/s -- 177,794 instructions, 86,726 cycles, IPC 2.05 |

Reproduce with `./tools/rpm_build.sh` (build + both tests) in the consuming repo.
