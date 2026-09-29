# Blackhole tt-metal disables TRISC instruction gathering by default

**Finding (source-read and device-measured).** On tt-metal `4f9fa9e0`,
Blackhole p150a (firmware bundle 19.5.0), the hardware's four-way
`.ttinsn` gathering is available but disabled in a default tt-metal run.
Model the checked-in `matmul_multi_core/asm_kernel` with one Tensix push per
executed RISC instruction. This does not remove FIFO backpressure when a
downstream expander or Wait Gate stops consuming pushes.

## Configuration path

- `tt_metal/hw/inc/internal/firmware_common.h` calls
  `configure_gathering()` at RISC startup. On Blackhole, unless
  `ENABLE_GATHERING` is defined, it sets `cfg0.DisTriscCache` (bit 18), citing
  workaround `tt-metal#16439`.
- `tt_metal/llrt/rtoptions.cpp` leaves gathering disabled by default.
  `TT_METAL_ENABLE_GATHERING=1` makes `tt_metal/jit_build/build.cpp` define
  `ENABLE_GATHERING` for the JIT build.
- When that macro is defined, `tt_metal/tt-llk/tt_llk_blackhole/common/inc/ckernel.h`
  disables gathering around `load_replay_buf()`'s record window and enables
  it afterward. The checked-in matmul `.S` was generated without this macro;
  changing only the runtime option for its assembly-kernel path does not add
  that protection. Regenerate and validate the assembly with matching flags
  before enabling gathering for the matmul.

## Device evidence

The [isolated FIFO probe](../../../example/standalone/tensix_fifo_probe/README.md)
ran one TRISC0 on p150a. Its warm modes execute the same linked burst address
once, wait for Tensix completion, then time a second execution. Linked ELF
disassembly confirms a continuous `ttnop` run and both calls to that address.
`t1-t0` includes clock-read and call overhead, so the table is a comparison
of windows, not an intrinsic instruction latency. These are the first two of
three repeated launches; the third has a common four-cycle lower overhead
and the same differences relative to its zero-length control.

| Tensix NOP count | Gathering off, `t1-t0` | Gathering on, `t1-t0` |
| ---: | ---: | ---: |
| 0 | 11 | 11 |
| 16 | 27 | 15 |
| 32 | 43 | 19 |
| 64 | 75 | 32 |
| 128 | 139 | 96 |

The probe read `cfg0=0x00060008` by default (bit 18 set), versus
`0x00020008` with the option (bit 18 clear). For warm bursts of up to 32,
the enabled window above the zero-length control is `ceil(N/4)` cycles;
default windows grow by `N`. A cold, one-pass enabled burst of 1 through 8
instructions still took `N+1` cycles, so a clear bit does not imply every
dynamic burst fuses. The 64-to-128 enabled window grew by 64 cycles, which is
consistent with an effective downstream rate near one Tensix instruction
per cycle. I-cache behavior, the RISC store queue, and FIFO stages all affect
that window. This probe does **not** locate the documented 28/32 first-FIFO
threshold or isolate the exact cycle of backpressure.

The hardware capacity and fusion rules are in the Blackhole ISA
[`PushTensixInstruction.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/PushTensixInstruction.md).
The controlled Wait-Gate experiment and its separate MMIO/CSR timing
boundaries are in
[`../isa/blackhole-tensix-push-backpressure.md`](../isa/blackhole-tensix-push-backpressure.md).
