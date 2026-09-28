# Analysing Tensix kernel `.S` with `llvm-mca`

**Finding.** Running `llvm-mca` over a Tensix kernel needs exactly one thing a stock
LLVM does not have: a scheduling model for the baby RISCV. As of
[`tt-bh-model-accuracy.md`](tt-bh-model-accuracy.md) that model exists in this
repository (`-mcpu=tt-bh`, installed by `tools/llvm-mca-tensix/apply.sh`), and with
it all five `matmul_multi_core` kernels analyse with **zero instructions silently
dropped**.

The one thing that still does not work is `.ttinsn`, and that is a *parse* gap in
stock LLVM, unrelated to any model.

Observed on tt-metal `4f9fa9e0` (Blackhole p150a), `llvm-project` `34152bb7d2d`
(LLVM 24.0.0git), over `example/standalone/matmul_multi_core/asm_kernel/`.
**Measured** unless marked otherwise.

## Building the binary (RISC-V only)

538 compilation units / 255 MB, if the build is restricted to the RISC-V backend. X86,
clang and MLIR are *not* needed to analyse Tensix kernels.

```bash
cmake -G Ninja -S thirdparty/llvm-project/llvm -B build/llvm-mca \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=/usr/bin/clang-20 \
  -DCMAKE_CXX_COMPILER=/usr/bin/clang++-20 \
  -DLLVM_USE_LINKER=/usr/bin/ld.lld-20 \
  -DLLVM_TARGETS_TO_BUILD=RISCV \
  -DLLVM_INCLUDE_TESTS=OFF -DLLVM_INCLUDE_BENCHMARKS=OFF \
  -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_BUILD_UTILS=OFF \
  -DLLVM_ENABLE_ASSERTIONS=OFF \
  -DLLVM_ENABLE_ZLIB=OFF -DLLVM_ENABLE_ZSTD=OFF

cmake --build build/llvm-mca --target llvm-mca -j 12
```

`build/` is already covered by the consuming repo's `.gitignore`. `ld.lld` is worth
passing explicitly — linking `llvm-mca` with GNU `ld` is far slower. The result is
`build/llvm-mca/bin/llvm-mca`, registering `riscv32`, `riscv32be`, `riscv64`,
`riscv64be`. Because there is **no host target**, `--version` prints an empty
`Default target:` and `llvm-mca` cannot infer a triple from the input: every
invocation needs an explicit `-mtriple=riscv32`.

**Rebuilding after installing the model takes ~4 minutes, not 538 units.** Touching
`RISCV.td` / `RISCVProcessors.td` only re-runs the RISCV tablegen and recompiles the
files that include its output: 39 ninja edges.

## 1. `.ttinsn` and `TTREPLAY` are not stock RISC-V

| Construct | Files | Count |
| --- | --- | --- |
| `.ttinsn <imm>` | `mm_trisc{0,1,2}_*.S` | 106 |
| `TTREPLAY` | `mm_trisc0_unpack.S`, `mm_trisc1_math.S` | 2 |

Neither appears in `brisc`/`ncrisc` kernels. Both are legitimate baby RISCV — `.ttinsn`
pushes a static instruction word to the Tensix coprocessor, and is listed in the
upstream ISA docs at
[`BabyRISCV/InstructionSet.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/InstructionSet.md).

Stock `llvm-mc`/`llvm-mca` reject them (`error: unknown directive`,
`error: unrecognized instruction mnemonic`). `-skip-unsupported-instructions=any`
tolerates **both** failure modes in one run — a parse failure *and* a missing
scheduling model — dropping the offending lines, still printing the diagnostics, and
finishing with exit 0. (`=parse-failure` and `=lack-sched` each cover only one of the
two.) Filtering the lines out up front is therefore optional, but it keeps the log
readable:

```bash
grep -vE '^\s*\.ttinsn|^\s*TTREPLAY' kernel.S > kernel.clean.S
```

## 2. The feature set, and why the CPU name is `tt-bh`

The baby RISCV ISA is not a guess: the tt-metal toolchain's own GCC names this core.

```
$ riscv-tt-elf-g++ -mcpu=tt-bh -Q --help=target | grep -E '^-march|^-mabi'
  -mabi=                     	ilp32
  -march=                    	rv32im_zmmul_zaamo_zba_zbb
```

`rv32i + m + zmmul + zaamo + zba + zbb`, soft float. Note what is **absent**, because
guessing here costs a parse failure:

- **no `C`** — the compressed extension is not in the arch string;
- **no `A`** — only `Zaamo`; there is no `lr.w`/`sc.w`;
- **no `F`** — `fadd.s`/`fmul.s`/`fmadd.s` exist in silicon but the ABI is soft-float,
  and `fdiv.s`/`fsqrt.s` do not exist at all.

So the recipe carries no `-mattr` at all. That is the practical payoff of a real
model: `-mcpu=sifive-e31` needed `-mattr=+m,+a,+c,+zba,+zbb` bolted on to make the
assembler accept the input, and even then dropped instructions.

## Recipe

```bash
# brisc / ncrisc kernels -- nothing else needed
llvm-mca -mtriple=riscv32 -mcpu=tt-bh -iterations=100 kernel.S

# trisc kernels, which contain .ttinsn / TTREPLAY
llvm-mca -mtriple=riscv32 -mcpu=tt-bh -iterations=100 \
         -skip-unsupported-instructions=any mm_trisc1_math.S
```

If *every* line is dropped, llvm-mca reports `error: no assembly instructions found.`
and exits non-zero — that is an empty input, not the flag failing.

## What changed from the Rocket workaround

Before the model, the only RV32 option was `-mcpu=sifive-e31`, which uses LLVM's
`RocketModel`, and it had to be run with `-skip-unsupported-instructions=lack-sched`
because **no RV32 model in LLVM 24 has scheduling information for `sh1add`/`sh2add` or
for `fence`** (`fence` is declared `Sched<[]>` in `RISCVInstrInfo.td` — it carries no
scheduling information in *any* RISC-V model, RV32 or RV64). That silently dropped 3
instructions in `brisc` and 8 in `ncrisc`; every dropped instruction is a free
optimistic bias.

| input | Rocket (`sifive-e31`) | `tt-bh` |
| --- | --- | --- |
| `writer_unary_interleaved_start_id_brisc.S` | 3 dropped | **0** |
| `reader_mm_output_tiles_partitioned_ncrisc.S` | 8 dropped | **0** |
| `mm_trisc{0,1,2}_*.S` | needs `=any` | needs `=any` (parse gap only) |

## What this does *not* give you

- **The Tensix coprocessor ops are invisible.** `llvm-mca` sees only the scalar RISC-V
  around them; the `.ttinsn` words — the actual math — are gone. The numbers describe
  address/control overhead, *not* Tensix throughput.
- **The model covers the core, not the memory system.** Load latency is keyed on the
  opcode, so an MMIO `lw` is charged the L1/local-RAM 2 cycles. See
  [`tt-bh-model-accuracy.md`](tt-bh-model-accuracy.md) for the model's limits
  and for `# LLVM-MCA-LATENCY` as the workaround.
- **Skipping `fence` is no longer necessary, and the model charges it 9 cycles** —
  the measured cost on p150a; see
  [`../isa/l0-data-cache-and-fence.md`](../isa/l0-data-cache-and-fence.md).
