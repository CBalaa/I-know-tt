# Blackhole `add_2_integers_in_riscv`: NoC command and barrier semantics

**Finding (measured and documented).** On Blackhole BRISC, `noc_async_read_barrier`
does not wait for a RISC-V `sw` to a NoC command register. The generated kernel
polls the NIU read-response counter, so its completion event is the read response
being received and written to local L1. `noc_async_write_barrier` similarly waits
for non-posted write acknowledgements. The command-buffer-ready poll and the
barrier completion are separate events.

Observed on tt-metal `4f9fa9e0`, Blackhole p150a (`CHIP_FREQ[MHz]=1350`), using
`example/standalone/add_2_integers_in_riscv/asm_kernel/reader_writer_add_in_riscv_brisc.S`.
The ISA and API semantics below come from the vendored Blackhole documentation
and `tt_metal/hw/inc`.

## Addresses in the generated BRISC kernel

For dedicated NoC0, BRISC uses command buffer 1 (`BRISC_RD_CMD_BUF=1`) for reads,
at `0xFFB2_1000`. The generated negative offsets therefore access:

| assembly address | register | meaning |
|---|---|---|
| `0xFFB2_0840` | `NOC_CMD_CTRL` | command buffer ready / initiate request |
| `0xFFB2_080C` | `NOC_RET_ADDR_LO` | read response destination |
| `0xFFB2_0800` | `NOC_TARG_ADDR_LO` | source address |
| `0xFFB2_0804` | `NOC_TARG_ADDR_MID` | source high address bits |
| `0xFFB2_0808` | `NOC_TARG_ADDR_HI` | source tile coordinates |
| `0xFFB2_0820` | `NOC_AT_LEN_BE` | transfer length |
| `0xFFB2_0040` | write buffer 0 `NOC_CMD_CTRL` | write command buffer ready / issue |

The read barrier polls `0xFFB2_0208`, the NoC0 status counter
`NIU_MST_RD_RESP_RECEIVED`, until it equals the software-issued-read counter.
The write barrier polls `0xFFB2_0204`, `NIU_MST_WR_ACK_RECEIVED`, until it equals
the software non-posted-write-ack counter. These are status-counter addresses,
not command-buffer readiness addresses.

## State-machine events

For a timing model, keep these events distinct:

```text
RISC-V sw to NOC_CMD_CTRL
  -> NIU command accepted / NOC_CMD_CTRL returns to 0
  -> request leaves the initiating NIU
  -> target processes request
  -> read response (or write acknowledgement) arrives
```

`noc_async_read_barrier` waits for the final read-response event. The official
TT-Metalium API describes it as a blocking call that waits for all outstanding
`noc_async_read` operations on the current core to complete. In contrast,
`noc_async_writes_flushed` only waits for writes to depart; it does not wait for
completion. The local API implementation is in
`tt_metal/hw/inc/api/dataflow/dataflow_api.h:1747-1819`.

The Blackhole NoC ISA documentation gives the same request lifecycle and counter
points in `tt-isa-documentation/BlackholeA0/NoC/Counters.md:71-95,119-150` and
the command-buffer layout in `.../NoC/MemoryMap.md:39-57,62-105`.

## Measured service-time anchors

The existing profiled add example measured (net of the 14-cycle marker window)
roughly 520 cycles for the two async reads plus their read barrier and 350 cycles
for the async write plus write barrier. Read-zone samples varied about 478--534
cycles on an otherwise idle device; this is an empirical route/DRAM/NoC service
distribution, not an architectural fixed constant. The whole kernel measured
910 cycles (JIT path) or 854 cycles (hand-written assembly path).

Sources: `test/mca_vs_measured/README.md` and the profiler records described
below.

## What is measured versus what is currently calibrated

The 520-cycle read number and 350-cycle write number are **whole profiler
zones**.  Each zone includes command-buffer setup stores, the RISC-V polling
instructions, the command issue, and the final response/ACK wait.  They do not
isolate either of these intervals:

```text
MMIO request emitted -> NIU command accepted
NIU command accepted -> read response / write ACK received
```

The read-zone samples (about 478--534 cycles) also show that the external
service time is not a fixed ISA latency.

Consequently, a timing model may use parameters such as a 3-cycle command
acceptance delay and 430-cycle read-response or 300-cycle write-ACK delays to
reproduce the current single-core add2 zone.  Those values are **calibration
assumptions**, not independent device measurements and not constants from the
NoC ISA documentation.  They must be replaced or reported as a distribution
when the destination, DRAM state, firmware revision, or NoC traffic changes.
The documented NoC hop costs are lower-bound route components; they do not
determine the barrier-release cycle by themselves.

## Fresh hand-written assembly rerun

On 2026-09-24, the checked-in `build_asm/add_2_integers_in_riscv` binary was
run three times with `TT_METAL_DEVICE_PROFILER=1`.  The run output reported
`Kernels: assembly from asm_kernel/ (USE_ASM_KERNELS=1)` and `Success: Result is
21` each time.  The raw CSV rows were:

```text
BRISC-KERNEL ZONE_START 3733340350762
BRISC-KERNEL ZONE_END   3733340351639   raw=877   net=863

BRISC-KERNEL ZONE_START 3734068093863
BRISC-KERNEL ZONE_END   3734068094738   raw=875   net=861

BRISC-KERNEL ZONE_START 3734791677813
BRISC-KERNEL ZONE_END   3734791678690   raw=877   net=863
```

Here `net = raw - 14`, using the measured profiler marker-pair floor.  These
fresh samples are 861--863 net cycles, while the earlier recorded run was 854
net (868 raw).  Both belong to the hand-written assembly path; the spread is
why the predictor should report a distribution or percentile rather than one
universal cycle number.

The four-cycle branch recovery cost is documented, but the upstream Baby-RISC
notes do not specify the predictor's initial state or replacement policy.  A
software timing model therefore has to choose a policy (the prototype uses a
backward-taken/forward-not-taken rule and learns each branch outcome).  Which
poll-loop iterations incur a misprediction, and the resulting extra cycles,
are **model assumptions** until they are isolated with a branch-only device
probe; they should not be folded into the NoC service-time calibration.
