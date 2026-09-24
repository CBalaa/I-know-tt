# tt-rpm `--trace-file` is snapshot-resume, not trace replay

Observed on tt-rpm `3094a66` with `ext/whisper` `674f645`.

**Finding:** `--trace-file F` never reads `F`. RPM has no trace-file parser at
all. The argument is used only as a *filename* from which to derive a Whisper
**snapshot directory**; the instruction stream always comes from the embedded
Whisper ISS executing, here resumed from that snapshot via `--loadfrom`.

*Verified by source inspection, not executed.* Written before the model had
been built in this checkout (it is built now; see
[`build-and-run.md`](build-and-run.md)). The evidence is strong enough that this
is not really in doubt, but no run has confirmed it end to end.

## Evidence

1. **The trace filename is stored and never read.**
   `ExecutionDriver::mTraceFileName` is assigned by `setTraceFileName()` and
   referenced nowhere else. A repo-wide grep for `mTraceFileName` returns only
   the setter and the declaration
   (`models/cpu/common/ExecutionDriver.hpp:59,123`).

2. **Nothing in `models/` parses a trace file.** No construction of
   `WhisperUtil::TraceReader`, no `std::ifstream` on a trace, no zstd/zlib
   decompression. The only mentions of the trace file's *content* format are
   `main.cpp`'s `looksLikeTrace()` heuristic, which just inspects the name:

   ```cpp
   if (s.find("-trace") != std::string::npos) return true;
   if (s.find("-snap")  != std::string::npos) return true;
   if (s.size() >= 8 && s.substr(s.size() - 8) == ".csv.zst") return true;
   if (s.size() >= 4 && s.substr(s.size() - 4) == ".csv")     return true;
   ```

   `TraceReader.hpp` *is* included (`models/cpu/common/Instruction.hpp:14`),
   but only for its `WhisperUtil::TraceRecord` **struct** — the in-memory
   record type. RPM populates that struct from the Whisper PerfApi packet in
   `ExecutionDriver::populateTraceRecord()`, in process. It never calls the
   reader's file API.

3. **The name is decoded into a snapshot path.** `ChipSim::initExecutionDriver`
   calls `generateSnapshotFolderName(trace_filename)` and throws if it comes
   back empty:

   > `Trace-driven mode: could not resolve snapshot folder from: <file>`

   `SnapshotUtil.cpp` requires the *basename* to contain two tokens:

   - `-s<N>-` — the simpoint id (`std::regex("-s[0-9]+-")`)
   - `i<N>[M|K]?` — the interval (`std::regex("i[0-9]+(M|K)?")`)

   It computes `snpShotInstr = simpointIndex * intervalSize`, builds
   `<prefix-before -s>snap<snpShotInstr>`, and stats it. If that path does not
   exist it retries a `snap-roi0-` variant; if neither exists, it warns and
   returns `""` — which makes `ChipSim` throw. (A directory passed directly is
   accepted if it contains `memory` and/or `registers`.)

4. **Whisper is then told to resume it.** `buildWhisperArguments()` appends
   `--loadfrom <snapshotFolder>`, and, absent `--target-elf`, runs
   `<...>/bin/fw_jump.elf` — i.e. the default Whisper firmware ELF, not a
   user program. It also looks for `whisper.json` in `<...>/bin/`, two
   directory levels above the snapshot folder (`generateWhisperPath`), with a
   `sparta_assert` that it exists.

So the sequence is: trace *name* → simpoint id + interval → snapshot dir →
`whisper --loadfrom` → PerfApi → pipeline. The `.csv`/`.csv.zst` payload is
never opened.

## Why this matters

- **`--trace-file` cannot be used on its own.** It needs a pre-existing
  snapshot directory laid out with the exact naming convention *and* a
  `bin/` sibling holding `whisper.json` + `fw_jump.elf`.
- **Nothing in this repo produces that snapshot.** `tests/` builds CoreMark and
  Dhrystone ELFs only; no script emits a Whisper snapshot or a simpoint trace.
  Whisper itself can generate snapshots (`--snapshotdir`,
  `--snapshotperiod`, `--compression`; `ext/whisper/Args.hpp`), and there is a
  `--loadfromtrace` flag to "also restore data-lines/instr-lines/branch-trace
  from a snapshot", but the simpoint-selection + trace-generation pipeline that
  produces the `-s<N>-i<M>` filenames is **external to this repository** and
  not vendored.
- **The supported, self-contained path is execution-driven (`--target-elf`).**
  Treat `--trace-file` as an internal hook for a checkpointing workflow that
  ships separately.
- **The "csv.zst" in `main.cpp`'s usage string is a red herring** for anyone
  trying to feed RPM a trace. It describes what the *filename* is expected to
  look like, not something RPM can parse.

## Related

- `ext/whisper/trace-reader/README.md` documents the trace record format (a
  single text file, one comma-separated record per retired instruction, with a
  header line naming the fields: `pc`, `inst`, `modified regs`, `memory`,
  `inst info`, `privilege`, `trap`, `disassembly`, `hartid`, `iptw`, `dptw`,
  `pmp`). That is the format the *name* convention belongs to; RPM does not
  read it.
- `io-contract-and-invocation.md` — the overall input/output contract.
