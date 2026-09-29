# `TTI_MOP` timing boundaries on Blackhole

**Finding (source-read; related p150a FIFO measurements are cited separately).** `TTI_MOP`
is an asynchronous push into the Tensix frontend. It is not a blocking call
that waits for MOP expansion, Replay expansion, or backend execution.

The Blackhole `PushTensixInstruction` documentation says that `.ttinsn` and
the equivalent `sw` push a 32-bit Tensix instruction into a frontend FIFO. A
full FIFO stalls the issuing baby RISC-V, but a write is considered processed
once the instruction has been pushed into that FIFO; that event is earlier than
Tensix completion. The MOP Expander and Replay Expander then consume and
expand instructions asynchronously, at up to one output per thread per cycle
when downstream stages do not apply backpressure.

For the `matmul_multi_core` math kernel, `.ttinsn 25165824` is
`TTI_MOP(1, 0, 0)`. It pushes one template-1 MOP. With the programmed
`MopCfg`, that MOP emits four `REPLAY(16,16,0,0)` instructions and one
`SETRWC`; each Replay emits the 16 recorded instructions. The one RISC push
therefore must not be equated with the 64 resulting `MVMUL`s.

Keep these observations separate:

* `mcycle` around `.ttinsn 25165824` measures RISC issue/store/FIFO progress.
  It can be similar for different Replay payloads while the payloads are still
  queued downstream.
* `PC_BUF_BASE + 8` (`MOPExpanderDoneCheck`) waits until the MOP input FIFO is
  empty and the MOP Expander is idle. It does not say that the expanded Replay
  instructions have started or finished executing.
* `PC_BUF_BASE + 4` (`CoprocessorDoneCheck`) waits for all in-flight work from
  that Tensix thread, and is the relevant completion point for the expanded
  backend sequence.

Changing the Replay buffer contents normally does not change the cost of
accepting the single MOP word when the frontend has space. It can change the
Replay/backend resource usage, Wait Gate behavior, and completion time. If
those stages apply backpressure, that pressure can propagate to the frontend
and make the measured RISC push window longer. Thus there is no universal
payload-independent `TTI_MOP` completion latency.

The related FIFO/backpressure probe was run on tt-metal `4f9fa9e0`, Blackhole
p150a, firmware bundle 19.5.0; it did not directly compare MOP payloads. See
the upstream [`PushTensixInstruction`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/PushTensixInstruction.md),
[`ManualTTSync`](../tt-isa-documentation/BlackholeA0/TensixTile/BabyRISCV/ManualTTSync.md),
[`MOPExpander`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/MOPExpander.md),
and [`REPLAY`](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/REPLAY.md)
pages. The Blackhole MOP page notes that its functionality is similar to, but
not identical with, the Wormhole page.
