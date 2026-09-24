# Building and running an out-of-tree tt-metal program against a build tree

Observed on tt-metal `4f9fa9e0`, arch **blackhole** (p150a), linking the **build tree**
`thirdparty/tt-metal/build_Release/` rather than an install tree. Both gotchas below were
hit in practice and both are easy to misdiagnose.

## 1. `find_package(TT-Metalium)` needs `TT-Metalium_DIR` under cmake 4.x

A build tree is not laid out as a standard `<prefix>/lib/cmake/<name>/`, and
`build_Release/tt-metalium-config.cmake` includes a sibling `Metalium.cmake` that does
**not** sit next to it. Under cmake 4.x — which searches `<prefix>/` itself — the wrong
copy is picked up, its `_IMPORT_PREFIX` resolves three directories too shallow (to the
**repo root**), and configure fails with:

```
     "/wafer/.../tt-loop-scheduler/lib/libtt_stl.so"
  but this file does not exist.
```

Fix: point at the complete package explicitly, and keep both prefix entries so `fmt` and
friends are still found.

```cmake
-DCMAKE_PREFIX_PATH="<tt-metal>/build_Release;<tt-metal>/build_Release/lib/cmake"
-DTT-Metalium_DIR=<tt-metal>/build_Release/lib/cmake/tt-metalium
```

The `lib/cmake/tt-metalium` copy has its own `Metalium.cmake` beside it, so
`_IMPORT_PREFIX` = `build_Release`, which is correct. (Alternative seen in this repo: use
cmake 3.31.x, which does not search `<prefix>/` and works with the documented recipe
alone.)

## 2. Set `TT_METAL_RUNTIME_ROOT` at run time or the program aborts

Without it:

```
TT_FATAL @ tt_metal/llrt/rtoptions.cpp:377: !this->root_dir.empty()
```

Root resolution order: a compiled-in `TT_METAL_INSTALL_ROOT` (nonexistent for a build
tree) → `$TT_METAL_RUNTIME_ROOT` → CWD containing `tt_metal/`. **`TT_METAL_HOME` is not
consulted by that path.** For a build tree **two** roots work, and which one you need
depends on whether the run pulls in the device profiler:

- `build_Release/libexec/tt-metalium/` — the install layout the build tree keeps
  (`tt_metal/hw/inc`, `tt_metal/soc_descriptors`, `tt_metal/core_descriptors`). Fine for a
  plain program; **measured** working for `add_2_integers_in_riscv` (a DPRINT run, no
  profiler).
- the tt-metal **source** tree — also **measured** working for that same program, and the
  one you must use when JIT compiles in the device profiler, whose headers are missing
  from the `libexec` tree (see `runtime-tree-missing-profiler-headers.md`).

So the earlier claim that the source tree is the *only* valid root is too strong: it is
the only valid root **for profiler runs**.

```bash
TT_METAL_RUNTIME_ROOT=<repo>/thirdparty/tt-metal ./your_program
```

The build tree keeps the install layout at `build_Release/libexec/tt-metalium/`, which is
why the runtime root and the CMake package prefix are two different directories — and why
the same build tree can serve as the runtime root either way.

## 3. Kernel paths and kernel-to-kernel includes

A non-absolute kernel path is resolved as: absolute → CWD → `$TT_METAL_KERNEL_PATH` →
system kernel dir → `$TT_METAL_HOME` (`tt_metal/impl/kernels/kernel.cpp:45 resolve_path`).
For an out-of-tree program, bake the program's own directory in at compile time so the
binary works from any CWD:

```cmake
target_compile_definitions(<target> PRIVATE
    "OVERRIDE_KERNEL_PREFIX=\"${CMAKE_CURRENT_SOURCE_DIR}/\"")
```

⚠️ **`OVERRIDE_KERNEL_PREFIX` is not consumed by tt-metal.** It is only a string-literal
macro; nothing in the runtime looks at it. The host program must concatenate it with the
relative kernel path *itself*:

```cpp
#ifndef OVERRIDE_KERNEL_PREFIX
#define OVERRIDE_KERNEL_PREFIX ""
#endif
CreateKernel(program, OVERRIDE_KERNEL_PREFIX "kernels/foo.cpp", core, cfg);
```

Defining it in CMake and then passing a bare `"kernels/foo.cpp"` compiles fine and fails at
run time with `Kernel file kernels/foo.cpp doesn't exist in any of the searched paths!`.

Two related facts that make out-of-tree kernel layout easier:

- `Kernel::process_include_paths` (`tt_metal/impl/kernels/kernel.cpp:374`) puts the kernel
  source's **own directory** on the `-I` path, so a kernel can include a sibling header by
  relative path (`#include "../common/foo.hpp"`).
- Header **contents** participate in the JIT cache key through the per-object `.dephash`
  mechanism (`Kernel::compute_hash`, `kernel.cpp:672-682`, hashes include *paths* only), so
- Host sources lifted out of `tt_metal/programming_examples/` need that directory's
  compile options too. `tt_metal/programming_examples/CMakeLists.txt` adds
  `-Wno-c++11-narrowing` (clang) / `-Wno-narrowing` (gcc) for **everything** under it, and
  an out-of-tree target inherits none of it: `SetRuntimeArgs` takes a `std::vector<uint32_t>`
  while `Buffer::address()` returns a 64-bit `DeviceAddr`, so the braced initializer list
  fails to compile with `error: non-constant-expression cannot be narrowed from type
  'DeviceAddr' (aka 'unsigned long') to 'unsigned int' [-Wc++11-narrowing]` (six times for
  `add_2_integers_in_riscv`). Re-add the option on the standalone target instead of editing
  the vendored source. **Measured** on clang 20.1.8, cmake 4.3.0.
