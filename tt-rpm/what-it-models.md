# What RPM models, and what it does not

**Finding.** RPM is a **core** model, not a device model. It has no NoC, no DRAM
and no Tensix L1, so a tt-metal kernel cannot be run on it as-is: the NoC calls
and the poll loops that wait on them are simply outside its vocabulary. What is
left -- the integer work -- it predicts optimistically for the measured BRISC
sequence, because its memory model can overlap loads more aggressively than the
device sequence allows. The word "serialize" here is a property of that
sequence's effective throughput, not a claim that every baby-RISCV load is
strictly serialized.

Observed on tt-rpm `3094a66`, Blackhole p150a, BRISC, tt-metal `4f9fa9e0`. The
harness, the raw runs and the full analysis are in the consuming repo at
`test/rpm_vs_measured/`.

## Coverage

| | RPM |
| --- | --- |
| pipeline | fetch / decode / rename / issue / execute / writeback / LSQ, ROB 128, PRF 128 |
| width | `fetch_width 16`, `dispatch_width 4`, OoO by default; an in-order preset is `config_inorder.yaml` |
| caches | L1 I$ and D$, probabilistic (`hit_rate 0.9`, `hit_latency 1`, `miss_latency 10`) or structural (256 KB, 64 B lines, 4-way); optional unified L2 |
| execution | Whisper ISS supplies functional results from a real ELF |
| **NoC** | **absent** -- `grep -rniE '\bnoc\b|noc_async' thirdparty/tt-rpm/{models,configs,scripts}` returns 0 |
| device L1 / DRAM | absent; `whisper.json` declares a flat PMA and the model has no memory-side timing |
| MMIO | modelled as an ordinary load; the wall-clock registers at `0xFFB1_21F0` have no special latency |

**Measured, not assumed:** the two `0x1F0`/`0x1F8` reads in the probe below are
cross-clock-domain MMIO on the device (~7 cycles, see
`../tt-metal/measuring-device-instrumentation-cost.md`) and ordinary cached
loads to RPM.

## RPM only loads 64-bit ELFs

```text
Error: Loading non-64-bit ELF file in 64-bit mode.
```

The `xlen` tag in `whisper.json` is deprecated ("xlen is obtained from the isa
tag") and ignored, and whisper's own default is 64. `--xlen 32` can be passed
through the execution driver's `whisper_flags` parameter
(`top.core0.edriver.params.whisper_flags`), a `vector<string>` (the CLI form
`-p <path> --xlen -p <path> 32` does not produce the two-element vector the
driver expects). However, RPM's `ExecutionDriver` itself is compiled with
`PerfApi<uint64_t>`, `Hart<uint64_t>`, and `Session<uint64_t>`; the resulting ELF
loader checks for a 64-bit ELF. Thus a Tensix baby-RISC-V (RV32IM) binary cannot
be simulated directly by this model even if Whisper is given `--xlen 32`.
Transcribe the instruction sequence to RV64 instead, remembering that `lui`
sign-extends in RV64 where it zero-extends in RV32.

## Prediction vs measurement, on `add_2_integers_in_riscv`

The kernel's runtime-argument prologue (six `get_arg_val` loads) is the only
part of that example RPM can be pointed at. Device numbers are profiler zones
(`DeviceZoneScopedN`, 8 samples x 2 runs, latch-order patch applied); RPM
numbers are per-iteration slopes with the loop overhead subtracted, on the
**exact instruction sequence the profiler bracketed**, transcribed from the
JIT'd `brisc.elf`.

| what | RPM stock OoO | RPM stock in-order | RPM calibrated | measured |
| --- | --- | --- | --- | --- |
| empty zone window body (8 insns) | 2.73 | 8.71 | 12.99 | **13** |
| args zone window body (17 insns) | 4.30 | 11.70 | 21.01 | **38** |
| the six `get_arg_val` loads | 1.57 | 2.99 | 8.02 | **25** |

"calibrated" = in-order preset plus `lat_load 4`, `dispatch_width 1`,
`fetch_width 4`.

**Measured on this core:** the six back-to-back runtime-argument loads add about
25 cycles in this exact bracket, or roughly 4 cycles per load as an *effective
sequence cost*. That number is not a universal per-load latency: the baby
RISCV LSU can have multiple requests in flight, and the result depends on L0
state, address region, queue occupancy, and the surrounding marker stores and
MMIO reads. The separate `test/mca_vs_measured` fence probe shows that an L0
miss can be substantially slower (see `../isa/l0-data-cache-and-fence.md`).

RPM can issue independent loads every cycle and, with `lat_load 4`, pays only
the last latency (about 8 cycles for the six loads). That is why it predicts 8
cycles for a sequence measured at 25. A scalar latency knob cannot encode the
device's address-dependent cache and LSU throughput; doing so needs an L0 model
and an explicit model of outstanding-load limits and ordering.

`lat_load` is easy to misread here. It is the decode/execute uop latency for
address generation (`DecodeStructuresParams` calls it “Load execution latency;
cache latency is separate”); D-cache `hit_latency`/`miss_latency` are independent
parameters. Setting `lat_load: 4` therefore does not model a 4-cycle L0/L1
access. In the calibrated probe it only stretches the execute-side latency and
changes how much of the marker/load sequence can overlap. The apparent agreement
(`empty` 12.99 vs 13 cycles) is a fit to one measured window, not evidence that
the cache timing is represented.

## What this does not say

- It says nothing about RPM's accuracy on long straight-line integer code.
  CoreMark's IPC 1.61 (4.1 M cycles for 6.6 M instructions) is RPM running its
  own test workload; this workspace has no matching device measurement, so it
  is not evidence of Baby-RISC accuracy.
- One block, one core, one board. The roughly 4-cycle effective cost for this
  load sequence is a BRISC observation, not a claim about every load or RPM's
  other configurations.

## Probe slope versus a one-shot prologue

The `test/rpm_vs_measured/rpm/run.sh` harness loops each short instruction block
for 16/32/64/128 iterations and fits a cycle-per-iteration slope. That removes
front-end fill and loop setup, but it also measures the loop's steady-state
cache state: in the RPM probe, the first access to `args_array` misses once and
later iterations reuse the generic D-cache line. A real BRISC kernel is entered
after `wait_for_go_message()`; on Blackhole that wait calls
`invalidate_l1_cache()` (`tt-metal/tt_metal/hw/inc/internal/firmware_common.h`),
which is a `fence` and leaves the baby-RISCV L0 data cache cold before the
runtime-argument loads. Therefore the slope is the right quantity for a
steady-state instruction comparison, but it is not by itself a one-shot cold
prologue prediction. A one-shot estimate needs a cache-flush/line-miss state
that RPM does not model for this core (or a separate cold-start calibration).
