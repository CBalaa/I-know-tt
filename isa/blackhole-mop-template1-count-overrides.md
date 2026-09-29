# Blackhole template-1 MOP count overrides

**Finding (source-read, not device-measured).** On Blackhole p150a with
tt-metal `4f9fa9e0`, the second and third arguments of `TTI_MOP(1, a, b)`
do not separately mean outer and inner loop counts. They encode a 32-bit
MOP instruction whose outer override spans both arguments. The official ISA
model marks nonzero overrides as unvalidated; the checked-in matmul uses
`TTI_MOP(1, 0, 0)`, which relies on the programmed `MopCfg` counts.

For template 1, tt-metal encodes `a` in instruction bits [22:16] and `b`
in [15:0]. The Blackhole ISA model interprets bits [19:10] as the optional
outer count override and [9:0] as the optional inner count override:

```text
outer_override = ((a & 0xf) << 6) | ((b >> 10) & 0x3f)
inner_override = b & 0x3ff
```

A nonzero outer override replaces `MopCfg[0]`; a nonzero inner override
replaces `MopCfg[1]`. A zero field leaves its corresponding `MopCfg` count
in effect. Bits [22:20], sourced from `a[6:4]`, are outside these two
documented Blackhole override fields. Do not infer Blackhole template-1
semantics from the macro argument names `loop_count` and
`zmask_lo16_or_loop_count`, which also serve template 0.

In `matmul_multi_core/mm_trisc1_math.S:241`, `0x01800000` is
`TTI_MOP(1, 0, 0)`. Both overrides are zero, so the template uses
`MopCfg[0] = 1` and `MopCfg[1] = 4` as programmed at lines 190 and 192.

Sources: [Blackhole template-1 functional model](../tt-isa-documentation/WormholeB0/TensixTile/TensixCoprocessor/MOPExpander.md),
[tt-metal MOP encoding macro](../../tt-metal/tt_metal/tt-llk/tt_llk_blackhole/common/inc/ckernel_ops.h),
and [tt-metal Blackhole assembly description](../../tt-metal/tt_metal/tt-llk/tt_llk_blackhole/instructions/assembly.yaml).
