# SC-005 measurement (spec 002, T074)

Program: `loomora_bench_format` (`core/tests/bench_format.cpp`, target `EXCLUDE_FROM_ALL`). It builds the project through
the public C API (`gen NAILS STEPS FILE`: one 500 x 500 mm frame, one layout of NAILS nails on a grid with diameter 0.5 mm,
one image, 3 spools, one result of STEPS steps), then `open FILE N` times, in this order,
`loomora_validate_path`, `loomora_project_open_path` (which validates), then reads every step of the result and every nail
of the layout into memory. `loomora_refgen --scale` (T054) was not used. Release builds, host i7-11800H (8 cores), 32 GB
(WSL: 16 GB), nothing else running.

| Project | File | Host | Validate | Open (incl. validation) | Read all steps + nails | Peak memory |
| --- | --- | --- | --- | --- | --- | --- |
| 5,000 nails, 100,000 steps | 321 KB | Windows 11, clang 23 | 0.009 to 0.010 s | 0.009 to 0.010 s | 0.005 s | 12 MB |
| 5,000 nails, 100,000 steps | 321 KB | WSL Ubuntu 26.04, g++ 15 | 0.008 to 0.010 s | 0.008 to 0.009 s | 0.005 s | 11 MB |
| 500,000 nails, 10,000,000 steps | 41 MB | Windows 11, clang 23 | 1.0 to 1.3 s (one run 2.6 s) | 1.0 to 1.1 s (one run 2.7 s) | 0.5 s (0.5 to 1.1 s) | 426 MB |
| 500,000 nails, 10,000,000 steps | 41 MB | WSL Ubuntu 26.04, g++ 15 | 1.0 s (one run 1.5 s) | 1.0 s (one run 2.2 s) | 0.6 s (one run 1.4 s) | 459 MB (`/usr/bin/time -v`: 470 MB) |

Peak memory: peak working set on Windows, `getrusage` maximum resident set on Linux, measured inside the process that
opens (the generator is a separate process). The first number of each range is the repeatable one; the slow runs are
isolated (host noise, 5 repetitions on Windows, 3 on Linux).

- Required: the 5,000 / 100,000 project validates and opens in under 1 s: **met by a factor of 100** (about 10 ms).
- Required: the 100 times larger project is opened with its warnings and not refused: **met**, both hosts open it with
  0 errors and 2 warnings (counts above the warning thresholds of the data model), in about 1 s.
- Nothing was over budget, so no change was made to `core/src/format`.
