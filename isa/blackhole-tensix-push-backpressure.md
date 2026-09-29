# Blackhole Tensix push buffering and downstream waits

**Finding (source-read and device-measured).** On Blackhole p150a, firmware
19.5.0 and tt-metal `4f9fa9e0`, a TRISC0 stream of Tensix NOP pushes can run
ahead of a blocked Tensix Wait Gate. With default tt-metal configuration,
an MMIO wall-clock load after 29 pushes can return before gate release,
while the one after 30 pushes observes the release time. A separate `mcycle`
CSR reading shows that the RISC push sequence itself still advances across
this boundary. The burst-after-CSR timing instead jumps between 35 and 36
pushes, under the same blocked NOP workload. Neither turn measures the first
FIFO's physical capacity.

## Hardware rule and measured path

The Blackhole ISA says the first FIFO holds at most 32 instructions, normally
stops accepting pushes above occupancy 28, and can reach 32 with a fused
four-`.ttinsn` push. Default tt-metal sets `cfg0.DisTriscCache`, disabling
fusion. Each RISC also has a four-entry store queue, and requests pass
through memory-system queues before a push is processed by the Tensix FIFO.
The frontend contains further MOP, Replay, and Wait Gate stages. Therefore
FIFO acceptance, RISC retirement, and backend completion are separate times.

The Blackhole ISA frontend diagram gives **three instruction FIFOs per Tensix
thread** (T0, T1, and T2 each have their own path), with 32-bit entries:

| Stage | Position | Storage / acceptance limit |
| --- | --- | --- |
| FIFO0 | Before MOP Expander | 32 physical slots; normal acceptance stops above occupancy 28, so single-word pushes can reach 29, while a fused four-word push from 28 can reach 32. |
| FIFO1 | Between MOP and Replay Expanders | 8 instructions. |
| FIFO2 | Between Replay Expander and Wait Gate | 2 instructions. |

The Replay Expander's separate 32-instruction replay buffer stores reusable
instructions; it is not FIFO2. Neither that buffer nor the four-entry
per-RISC store queue should be added to FIFO capacity. The ISA also says
downstream FIFO capacity is only additive to FIFO0 when MOP or Replay
expansion happens, due to Auto TTSync tracking restrictions that apply even
when Auto TTSync is disabled. These sizes are from the ISA diagram and text,
not inferred from the blocked-gate timing measurements below.

The four-entry RISC store queue must not be confused with four-way `.ttinsn`
gathering. Gathering happens in the TRISC instruction cache before EX1: up to
four adjacent `.ttinsn` words become one fused RISC store carrying up to 128
bits. Such a fused group has the ordering behavior of one store request. The
ISA does not specify whether its 128-bit data consumes one or multiple
physical store-queue slots, so the four-entry queue must not be used to derive
an exact fused-push look-ahead. With gathering disabled, four separate
`.ttinsn` stores can occupy up to four store-queue entries, subject to the
queue draining into the memory subsystem. In either case, a store can retire or leave the
store queue before its write request is processed by FIFO0, so a full FIFO0 can
be temporarily hidden by the store queue and intervening memory-system
buffers. This does not make the extra words accepted by FIFO0, nor does it
give a fixed look-ahead of four Tensix instructions.

The Wait Gate is not a timer and does not block merely because FIFO2 is
non-empty. It examines the head instruction when that instruction reaches the
gate. The documented hardware causes include a busy or contended backend unit,
an in-progress scalar operation, a mutex held by another thread, and a source
bank whose `AllowedClient` is not yet correct for the instruction. A latched
`STALLWAIT`, `SEMWAIT`, or `STREAMWAIT` can add a software-selected block mask;
once the first instruction caught by that mask is held, the in-order frontend
holds every instruction behind it. Auto TTSync can create an additional wait
for tracked GPR, TDMA, or backend-configuration dependencies. Passing the gate
means dispatch into the backend, not backend completion.

The [reproducible probe](../../../example/standalone/tensix_fifo_probe/README.md)
uses `SEMWAIT` to stop Tensix T0 at the Wait Gate, has TRISC0 push N NOPs,
then has TRISC1 release the semaphore at a chosen wall-clock time. TRISC2
provides a watchdog; it did not fire in the runs below. `t1-t0` uses MMIO
wall-clock loads around the push burst and can include delay to service the
second load. `c1-c0` brackets the burst with `mcycle` CSR reads, before that
second MMIO load. Each entry is three independent program launches, in cycles:

| Push form and cfg0 | N | Release delay | Wall `t1-t0` | CSR `c1-c0` |
| --- | ---: | ---: | ---: | ---: |
| `.ttinsn`, `0x00060008` | 29 | 5000 | 55/50/50 | 49/45/45 |
| `.ttinsn`, `0x00060008` | 30 | 5000 | 5044/5034/5037 | 50/46/46 |
| `.ttinsn`, `0x00060008` | 35 | 5000 | 5036/5041/5041 | 55/51/52 |
| `.ttinsn`, `0x00060008` | 36 | 5000 | 5044/5044/5044 | 5039/5039/5039 |
| `.ttinsn`, `0x00060008` | 40 | 5000 | 5057/5048/5049 | 5052/5043/5044 |
| `.ttinsn`, `0x00060008` | 40 | 10000 | 10051/10045/10057 | 10046/10040/10052 |
| `sw`, `0x00060008` | 35 | 5000 | 5039/5044/5042 | 59/54/53 |
| `sw`, `0x00060008` | 36 | 5000 | 5057/5036/5050 | 5052/5031/5045 |
| `.ttinsn`, gathering on, `0x00020008` | 32 | 5000 | 33/29/29 | 28/25/24 |
| `.ttinsn`, gathering on, `0x00020008` | 33 | 5000 | 5051/5037/5037 | 29/25/25 |

The default N=29/30 turn occurs in the MMIO observation: `c1-c0` increases
only one cycle while wall `t1-t0` moves to the release time. In the same
experiment, `c1-c0` jumps between N=35 and N=36, and N=40 follows a change
of release delay from 5000 to 10000 cycles. Ordinary `sw` shows the same
CSR turn. This locates when the RISC can reach the following CSR under this
workload, not the exact EX1 entry of the 36th push. Gathering moves the MMIO
observation turn to N=32/33; neither CSR measurement waits at those N.

In an unblocked warm-I-cache stream, the default wall-clock window grows
one cycle per NOP. Enabled gathering reduces short-burst growth to about
one cycle per four NOPs up to N=32; long bursts approach one NOP/cycle
effective downstream progress. An earlier blocked run's Manual TTSync
window waited about 5000 cycles at N=29, confirming the Wait Gate was held.

The debug `RISCV_DEBUG_REG_INSTRN_BUF_STATUS` register reports the FIFO
**before the Replay Expander**, not the first RISC-facing FIFO. While the
gate is blocked, its T0 status changes from non-full to full between N=9
and N=10, well before the MMIO observation's N=29/30 turn. A status bit and
the MMIO timing window therefore cannot be used to infer the first FIFO's
occupancy or the exact cycle in which each write is accepted. The `mcycle`
turn is also downstream of the store queue and memory-system buffering.
MOP/Replay expansion changes how the intermediate queues fill, so the N=30
or N=36 turns must not be installed as universal FIFO capacities in the
predictor.

For modeling, keep the documented first-FIFO acceptance rule and the
per-thread MOP/Replay/Wait Gate state distinct from the RISC store queue
and pending memory requests. Calibrate the as-yet-undocumented inter-stage
delays and buffering with device events; report RISC stall, FIFO acceptance,
and Tensix completion separately.

A minimal first-FIFO transition is `q0_next = q0 + accepted - dequeued`:
without gathering, at most one instruction is accepted per cycle; with
gathering, one fused push may carry up to four. The documented acceptance
gate closes when `q0 > 28` and reopens after it drops to 28. Its downstream
dequeue rate is at most one instruction per thread per cycle, and can be
lower when expansion or later stages backpressure it. The exact ordering of
same-cycle accept/dequeue and the queues between RISC LSU and `q0` remain
calibration parameters. A single queue with capacity 30 or 36 would not
reproduce the distinct MMIO, CSR, and Manual TTSync observations.

Sources: Blackhole [`PushTensixInstruction.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/PushTensixInstruction.md),
[`MemoryOrdering.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/MemoryOrdering.md),
[`CSRs.md`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/CSRs.md),
and [`SEMWAIT.md`](../tt-isa-documentation/BlackholeA0/TensixTile/TensixCoprocessor/SEMWAIT.md).
