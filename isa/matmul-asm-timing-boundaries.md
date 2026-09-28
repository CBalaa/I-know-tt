# Timing boundaries in `matmul_multi_core` assembly

**Finding (source-read, not device-measured).** For the Blackhole p150a assembly
generated with tt-metal `4f9fa9e0`, instruction syntax alone does not identify
the timing event. Classify each executed instruction by its address or decoded
Tensix effect, then track issue, acceptance, completion, and observation
separately. The five `.S` files under
`example/standalone/matmul_multi_core/asm_kernel/` contain only `kernel_main`;
linked firmware and called helpers are outside them.

## Cases present in this kernel

| Assembly site | Timing effect |
| --- | --- |
| `reader_mm_output_tiles_partitioned_ncrisc.S:67-72` | `divu`/`remu` use runtime operands, so the executed path and divider cost matter. |
| `reader_mm_output_tiles_partitioned_ncrisc.S:112-159`, `writer_unary_interleaved_start_id_brisc.S:81-137` | RISC loads/stores access NoC command/status MMIO. Command-register store, NIU acceptance, response/ACK, and the next successful RISC poll are different events. |
| `mm_trisc0_unpack.S:257-263`, `mm_trisc2_pack.S:261-265` | RISC `lw` polls CB credit in NoC-overlay stream registers. Related updates may be RISC stores or Tensix `STOREREG`; see [`../tt-metal/cb-credit-counters-in-noc-stream-regs.md`](../tt-metal/cb-credit-counters-in-noc-stream-regs.md). |
| `mm_trisc1_math.S:19-24`, `mm_trisc0_unpack.S:190-198` | `sw; lw; and` at `PC_BUF_BASE+4/+8` uses Manual TTSync. The load response can wait inside the memory system; it is not a RISC polling loop. `+4` waits for the Tensix thread, whereas `+8` waits only for MOP expansion. |
| `mm_trisc1_math.S:238`, `mm_trisc0_unpack.S:283`, `mm_trisc2_pack.S:272` | `sw` to `INSTRN_BUF_BASE` pushes a Tensix instruction, just like `.ttinsn`. The RISC store may retire before the FIFO accepts the push. |
| `mm_trisc1_math.S:127-175,241` | `TTREPLAY 16,16,0,1` records the following 16 instructions without executing them; later MOP/REPLAY expansion creates the backend work. One source `.ttinsn` is not necessarily one executed backend operation. |
| `mm_trisc1_math.S:74,219` | Calls target a helper outside the extracted `kernel_main` assembly; use linked ELF disassembly when that path executes. |

The instruction FIFO can stall its issuing RISC when full. Adjacent `.ttinsn`
instructions on TRISC can fuse up to four pushes in one RISC cycle, while the
Tensix frontend dequeues at most one per thread per cycle. `STALLWAIT` and
`SEMWAIT` hold a Tensix thread at its Wait Gate; Auto TTSync can also stall RISC
MMIO accesses. Backend work has resource and data dependencies, including the
documented four-cycle no-read window after a `Dst` block write. A constant
per-push latency therefore cannot establish a completion timestamp.

`llvm-mca -mcpu=tt-bh` can provide a static scalar scheduling baseline. It has
no runtime addresses, branch outcomes, MMIO effects, or Tensix queue state.
TT-NPE's public `npeAPI` runs a batch workload with predetermined NoC start
offsets; it does not model CB barriers or provide a live completion callback
for the existing TT-loop scheduler. Its internal `ResponseScheduler` can step
some read responses, but is not an incremental interface to the full NoC
model. The measured TT-NPE accuracy on this kernel used *device-recorded* NoC
issue timestamps, not issue times predicted from these `.S` files.

References: Blackhole Baby-RISC
[`PushTensixInstruction.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/PushTensixInstruction.md),
[`ManualTTSync.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/ManualTTSync.md),
[`AutoTTSync.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/AutoTTSync.md),
[`Dst.md`](../tt-isa-documentation/BlackholeA0/TensixTile/TensixCoprocessor/Dst.md),
[`REPLAY.md`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/REPLAY.md),
and [`../tt-npe/prediction-model.md`](../tt-npe/prediction-model.md).
