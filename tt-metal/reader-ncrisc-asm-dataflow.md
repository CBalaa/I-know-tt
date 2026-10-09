# Generated NCRISC reader assembly: CB/NoC mapping

**Finding.** In the original runtime-bounds `matmul_single_core` Blackhole
assembly generated from tt-metal `4f9fa9e0`, the NCRISC reader uses NOC 1 and
the following fixed addresses:

- `0xffb40000` and `0xffb41000` are overlay stream 0 and stream 1.  For a
  local CB, stream `cb_id` maps directly to CB `cb_id`; offset `+40` is the
  stream's `STREAM_REMOTE_DEST_BUF_SIZE` scratch register used as
  `tiles_received`, and `+32` is `STREAM_REMOTE_DEST_BUF_START` used as
  `tiles_acked`.
- `0xffb30000` is NOC 1's register base.  `0xffb31000` is used as a convenient
  base for NCRISC read command-buffer 1; negative offsets from it address
  `NOC_RET_ADDR_LO` (`-2036`), `NOC_TARG_ADDR_LO` (`-2048`), the split high
  address registers (`-2044`, `-2040`), `NOC_AT_LEN_BE` (`-2016`), and
  `NOC_CMD_CTRL` (`-1984`).
- `0xffb30208` is NOC 1 status `NIU_MST_RD_RESP_RECEIVED`, emitted as
  `520(0xffb30000)`.  The read barrier polls it until it equals
  `noc_reads_num_issued[1]`, then executes the API's `fence` to invalidate the
  Blackhole baby-RISC-V L0 data cache. The MMIO counters themselves are not
  cached; this fence is distinct from the NoC completion poll.

For an interleaved DRAM TensorAccessor with eight banks and 2048-byte pages,
the generated address calculation is `bank = page_id & 7`,
`bank_offset_index = page_id >> 3`,
`dram_addr = base + bank_to_dram_offset[bank] +
bank_offset_index * 2048`, and the packed bank coordinate is loaded from the
NOC-1 row of `dram_bank_to_noc_xy`.  A page larger than the 16 KiB maximum NoC
burst is split into 16 KiB requests, with an unsigned carry propagated into the
packed high address; the final request uses the remainder.

Register meanings depend on the basic block. At `.Lreader_ncrisc_46`, `a0` is
`mt`, `s0` is `Kt`, `t6` is `Mt`, and `a2` is the compiler-maintained A-row
base `mt * Kt`. This `a2` value is **not** the C++ `itileB`: that source local
is initialized to zero and never used. The compiler replaces repeated
`mt * Kt` multiplication with an induction variable advanced by `Kt`.
Within the K loop, `s0` is the current A page index, `s1` is the current B page
index, and `s6 = (mt + 1) * Kt` is the exclusive A-page end. Each K iteration
increments `s0` by one and `s1` by `Nt`; other registers are reused as transfer
temporaries after the outer-loop state has been spilled to the stack.

Assembly lines 47-49 are the common M-loop entry: reset `nt` (`a4`) and branch
to line 73 if `Kt != 0`. Lines 50-56 are only the `Kt == 0` path: count through
the empty N loop, increment `mt`, and repeat. This is an early test of the
source's innermost `kt < Kt` condition, not a separate source-level `if` or a
CB/NoC wait. Earlier guards ensure `Mt` and `Nt` are nonzero on this path.
Lines 57-72 are the shared function epilogue; the nonzero-K path reaches it
via line 314 after its loop tail at lines 301-313. The function epilogue's
placement before the transfer blocks in the file does not mean it executes
before them.

The two epilogue entry labels match two register-save paths (source-derived
from the assembly). `.Lreader_ncrisc_74` restores `s1` through `s10`, then
falls through to `.Lreader_ncrisc_25`, which restores `s0`/`s11`, releases the
112-byte stack frame, and returns. The `Mt == 0` / `Nt == 0` guards at lines
24-25 jump directly to `_25`: `s0`/`s11` have already been saved, but the
other saved registers are only stored after those guards. Full completion
uses `_74`, either by fallthrough from line 56 or by the jump at line 314.
The `Kt` spill at `12(sp)` is separate from the caller's saved `s0` at
`108(sp)`; the former restores loop state, the latter preserves the ABI.

Observed on the generated assembly in this repository (`metadata.txt`,
Blackhole p150a configuration, `NUM_DRAM_BANKS=8`, `NOC_INDEX=1`).

The contrast with `mm_trisc0_unpack.S` is structural rather than a difference
in loop count. The compute kernel obtains `Mt/Kt/Nt` through
`get_compile_time_arg_val`, so `20,20,20` are compile-time constants. Its loop
body has fixed CB wait/pop and `.ttinsn` operations; the generated control
section is three induction loops with branches at the ends. The reader obtains
`Mt/Kt/Nt` through runtime arguments and inlines CB reservation, NoC completion
polls, TensorAccessor bank/address generation, burst splitting, and CB pointer
wrapping. Those operations introduce many inner branches and register spills;
they are not extra source-level matrix loops.

The repository's generator also compiles the dataflow reader with `-O2` but
the compute/TRISC kernel with `-O3` (`generate_asm_kernel.sh`,
`compile_dataflow` versus `compile_compute`). This can preserve more generic
guards and spills in the reader, but it is secondary to the API and
runtime-versus-compile-time differences above.

## Compile-time bounds regeneration

On 2026-10-09, regenerated the modified reader with the same generator's
`-O2`, `-mcpu=tt-bh`, and `-fno-lto`, using reader compile-time arguments
`2,2048,2,2048,20,20,20`. This is an inspected compiler-output result, not a
hardware execution measurement; the preceding line numbers refer to the
original runtime-bounds snapshot. The source and host now place `Mt/Kt/Nt`
after both TensorAccessor argument lists, with only A/B addresses at runtime.

The generated reader has M/N/K entries `_26` (line 50), `_44` (54), and `_41`
(59), with tail `bne` backedges at lines 277, 274, and 270 respectively.
Runtime dimension loads, dimension-zero guards, and the empty-K path are
gone; the function epilogue follows the loop tails. K now counts with the B
page index: `s1 = nt + kt * 20`, compared against `s7 = nt + 400`. M counts
with `a2 = mt * 20`, advancing by 20 until 400. CB/NoC polls, transfer
branches, and an `nt` spill remain. This configuration produces the regular
three-loop skeleton, but identical block layouts or unrolling are not
guaranteed for other dimensions, compiler versions, or flags.

In this 292-line output, each K iteration serially reads A into CB0 (lines
60-162) and B into CB1 (164-266). The symmetric stages are CB reservation
(60-72 / 164-176), address generation (73-89 / 177-193), optional large-page
burst handling (90-128 / 194-232), final or only request (129-142 / 233-246),
response-counter polling plus fence (143-150 / 247-254), and CB push with
write-pointer wrap (151-162 / 255-266). With the configured 2048-byte pages,
`bleu` at lines 90/194 jumps directly to `_31`/`_38`, skipping lines 91-128
and 195-232. The retained large-page paths are distinct from the M/N/K loops.
`_33` and `_40` merge the CB write-pointer wrap paths, not extra loops.

Register roles also changed: `s6` is the A page index, `s1` the B page index,
and `s7` the exclusive B-page end. `a4` holds `nt` at the N entry/tail and is
saved at `12(sp)` before reuse as a transfer temporary. `s0` is now the CB1
MMIO base (`0xffb41000`), and `t6` the NOC-1 register base (`0xffb30000`),
not the old runtime dimension registers. There are 8000 K iterations, each
issuing one A and one B request in this page-size configuration.

### Two backedges to the large-read label

In the compile-time snapshot, `_29` has two distinct backedges. Line 99
retries the command-buffer readiness poll at lines 98-99; line 117 repeats
the whole 16 KiB request after advancing source and destination addresses.
This is the expansion of Blackhole `ncrisc_noc_fast_read_any_len` in
`noc_nonblocking_api.h:861-877`. Command-buffer readiness permits another
request; response completion is separately polled at `_32` (lines 143-145).

For an original length `L > B`, `B = 16384`, lines 91-94 precompute the
destination stop pointer as `D + floor((L - 1) / B) * B`. Thus `_29` sends
`q = floor((L - 1) / B)` full chunks and `_31` sends the final `L - q*B`
bytes, in the range 1 through B. For an exact multiple of B, the final
request still has B bytes, not zero. Lines 112-116 implement a 64-bit source
increment with `sltu` carry detection, plus the 32-bit destination increment.
Lines 118-128 prepare the final request from the original address/length;
the low/high source halves change registers before entry to `_31`.
