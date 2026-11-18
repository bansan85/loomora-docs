# Fuzzing run of the reader, SC-004 (spec 002, T073)

Result: **12.6 million inputs, no crash, no leak, no undefined behaviour, no out-of-bounds access, no input
slower than 5 s** (slowest input 0.26 s). Nothing to fix, so no regression file was added.

## Build

- Host: WSL Ubuntu 26.04 on an i7-11800H (8 cores, 16 threads), 16 GB. Compiler: clang++ 23.1.6 (LLVM 23, with libFuzzer).
- `cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_COMPILER=clang++ -DLOOMORA_BUILD_FUZZ=ON`, targets
  `loomora_fuzz_project` (libFuzzer) and `loomora_fuzz_loop` (mutation loop), same entry point
  (`LLVMFuzzerTestOneInput` of `core/fuzz/fuzz_project.cpp`: validate, open and read every item of an archive in memory).
- Flags: Release (`-O3 -DNDEBUG`), backend and drivers instrumented with
  `-fsanitize=address,undefined -fno-sanitize-recover=undefined -fno-omit-frame-pointer`, leak detection on (ASan default).
- libFuzzer: `-timeout=5 -max_len=262144 -runs=N -seed=S`, the structure-aware custom mutator (stored entry edited, CRC-32
  and manifest SHA-256 repaired). Loop: `--iterations N --seed S` (random seed per process, seeds + all-stored copies).

## Runs (15 processes at once, distinct seeds, own corpus directory)

| Run | Processes | Inputs | Seed sets |
| --- | --- | --- | --- |
| 1, libFuzzer | 6 (seeds 1001, 1002, 1004, 1005, 1006, 1008) | 6 x 1,000,000 = 6,000,000 | all corpus + refgen; valid-* + refgen; invalid-*/warning-* + refgen |
| 1, loop | 7 (seeds 1100 to 1106) | 7 x 600,000 = 4,200,000 | valid-* + refgen; hostile-* + refgen; invalid-*/warning-* + refgen; all corpus + refgen |
| 2, libFuzzer | 6 (seeds 5000 to 5005) | 6 x 400,000 = 2,400,000 | hostile-*, large-warn-*, refgen |

- Seeds: the 114 files of `tools/gen_invalid_corpus.py --skip-truncate` (valid, invalid, hostile, warning, large-warn)
  plus `loomora_refgen` output (`basic`, `empty`, `escape40`).
- Wall time: run 1 was 2,746 s (45.8 min, 10.2 million inputs, under the one-hour limit); run 2 was 720 s.
  Throughput with 15 processes on 16 threads: 220 to 330 inputs/s per loop process, about 390 inputs/s per libFuzzer process
  (about 1,500 inputs/s for one process alone).
- libFuzzer final stats: slowest unit 0 s (< 1 s), peak RSS 480 to 490 MB, final coverage 140 to 143 libFuzzer coverage points (plateau
  reached in the first minutes), 92 to 186 units added per process.
- Run 1 had two libFuzzer processes (hostile seed set) that exited at start because the seed argument was a file, not a
  directory; run 2 redoes that seed set with the right argument (they are not counted in the totals above).

## Result

No sanitizer report, no crash artifact (`crash-*`, `timeout-*`, `leak-*`, `oom-*`), no input over 5 s. Opened vs rejected
inputs and the most frequent first errors are printed by each loop process (13 % of the loop mutants still open and pass validation (78,360 of 600,000 for the first process)).
Logs: `$HOME/loomora-build/fuzz-logs/run1`, `run2` (not committed).
