# TRISC Unpack Context Ping-Pong and Cfg-State Branching

**Observed on:** tt-metal `4f9fa9e0`, Blackhole A0 (p150a), LLK unpack kernels (`llk_unpack_AB_matmul.h`).

## Concrete Finding

In Tenstorrent Blackhole TRISC0 unpack and TRISC2 pack assembly kernels:
1. **Unpack Double Buffering (`unp_cfg_context`)**: The kernel ping-pongs between two hardware configuration contexts (`Context 0` and `Context 1`) on every tile iteration to overlap tile unpack dispatch with configuration register updates without stalling the datapath.
2. **Cfg-State ID (`cfg_state_id`)**: A global LLK configuration state ID (`_ZN7ckernel12cfg_state_idE`) is evaluated in the prologue via conditional branches to select tile configuration offset addresses.

## Evidence from Disassembly (`mm_trisc0_unpack.S`)

### 1. `unp_cfg_context` Ping-Pong FSM

In prologue setup:
```assembly
    lui t5, %hi(unp_cfg_context)
    sw  zero, %lo(unp_cfg_context)(t5)   # State initialized to 0
```

Inside the innermost tile unpack loop:
```assembly
    lw  a5, %lo(unp_cfg_context)(t5)
    bne a5, zero, .Lcontext_1           # Branch on context: 0 -> fallthrough, 1 -> taken
```

- **Context 0 Arm (Fallthrough)**: Configures unpacker registers for context 0 (e.g. registers at offsets 304, 496) and transitions state to 1:
  ```assembly
    sw  t0, 304(t3)
    sw  t6, 496(t3)
    sw  t2, %lo(unp_cfg_context)(t5)   # t2 == 1 -> state toggles 0 -> 1
  ```
- **Context 1 Arm (Taken)**: Configures unpacker registers for context 1 (offsets 308, 500) and transitions state to 0:
  ```assembly
    sw  t0, 308(t3)
    sw  t6, 500(t3)
    sub t6, t2, a5                     # 1 - 1 = 0
    sw  t6, %lo(unp_cfg_context)(t5)   # state toggles 1 -> 0
  ```

Because this is a closed 2-state toggle machine, an $N$-tile loop executes Context 0 exactly $\lceil N/2 \rceil$ times and Context 1 exactly $\lfloor N/2 \rfloor$ times.

### 2. `cfg_state_id` Branches in Prologue

Prologue branches conditionally select configuration table base pointers:
```assembly
    lw  a4, %lo(_ZN7ckernel12cfg_state_idE)(t3)
    addi t3, a5, 896
    bne a4, zero, .Lcfg_state_nonzero
    mv  t3, a5
.Lcfg_state_nonzero:
```
Because this occurs only once during kernel setup outside of loops, its impact on total kernel latency is negligible (1-2 cycles out of ~400,000 cycles). When `cfg_state_id` is unknown, an expected cost model $(C_{\text{then}} + C_{\text{else}})/2$ accurately represents the average configuration launch overhead.
