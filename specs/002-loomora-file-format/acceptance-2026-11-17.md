# Acceptance of spec 002 (`.loomora` format), task T102, 2026-11-17

Branch `feat/fmt-quickstart` (from `feat/loomora-format`, worktree `loomora-fmt-quickstart`). Machine: Windows 11 x64 (clang 22, Ninja) and WSL Ubuntu 26.04 (g++ 15.2, clang++ 23 with libFuzzer, MinGW-w64 13 i686 and x86_64, Python 3.14). Neither shell was elevated (Windows: not administrator; WSL: uid 1000). WSL builds are under `$HOME/loomora-build/qs-*`. Docs repository used for `records`: `docs/format-records` (worktree `loomora-docs-fmt`).

## Per step

| Step | Status | What was run | Result |
|---|---|---|---|
| 1 | ran | `python tools/verify.py quick` on Windows; the same in WSL; `python tools/validate_records.py --root <docs worktree> --code-root .` | Windows: 11 passed, 1 failed (`records`), 2 skipped (`run_vectors windows modern`: not built; `std`: no g++ on the host). WSL: 14 passed, 1 failed (`records`); `std` PASS (472 s). `records` fails only because the docs pointer is not moved (it reports `LOOMORA_FMT_LSEEK64`, `LOOMORA_FMT_NO_LARGE_FILES`, `LOOMORA_FMT_WIN2000_SEEK`, `LOOMORA_TEST_FP_FLAGS` undeclared and S-F1..S-F3 unrecorded, all declared on the docs branch). `validate_records.py` against the docs branch: 0 errors, 0 warnings. |
| 2 | ran | `loomora_format_selftest` (and `ctest`) on Windows x64 clang, Linux x86_64 g++, Linux i686, Windows 2000 i686 MinGW (run on Windows 11) | `format selftest: ok` everywhere, exit 0. Goldens (basic, empty, escape40, multi, multi-light, one) are checked inside. Cross-platform comparison: step 10. |
| 3 | ran | `loomora_refgen <dir>`, `check_loomora.py`, `loomora-cli validate` on `multi` and `multi-light`; Catch2 `SC-013`, `[editor]` (38 cases, 3,733 assertions) | 0 errors, 0 warnings on `multi`; `multi-light` has the one `warn.light_project`. |
| 4 | ran | `python tools/check_readonly.py --build-dir <build>` | `check_readonly: ok`. Also PASS in both `verify.py quick` runs; Catch2 `use mode*` (2 cases) passes. |
| 5 | ran | `gen_invalid_corpus.py --out`, `check_cli_validate.py --corpus` (and a first loop over the bare command line) | 3,348 files, 0 failed, 35 option rows, 0 failed. All 3,235 truncations exit 1. |
| 6 | ran | `loomora_use_demo export multi.loomora one-r3.loomora light 3` (and result 4 for the 3D frame), `check_loomora.py`, `--compare-mesh`; Catch2 `[export]` (9 cases) and `[light]` (3 cases) | Result 3 export: 1 frame, 1 layout, 1 image, 2 spools, 4,204 bytes vs 21,258 (full). Mesh geometry identical, entry `sha256` differs (UVs dropped, per contract). Edit refused (Catch2). 50% test: full 89,827 bytes, light 12,147. |
| 7 | ran | Catch2 `version skew*` | 5 cases, 635 assertions, all pass. |
| 8 | ran, short | clang++ 23 libFuzzer + ASan + UBSan, `-max_total_time=120 -timeout=5 -max_len=262144`, seeds from `gen_invalid_corpus.py --skip-truncate` | 149,089 inputs in 121 s, exit 0, no artifact (no crash, timeout, leak, sanitizer report). Not the 10 million or one hour: `docs/implementation/validation/results/spec-002/fuzz_run_sc004.md` (12.6 M inputs) is the full proof. |
| 9 | ran | `run_vectors.py --validate-project` and `--roundtrip-project` on `multi.loomora` (Windows Legacy); `verify.py quick` `ui project` rows | Windows Legacy: 4 of 4 per mode; `ui project` 37/37 for Windows Legacy, Linux Legacy and Linux Modern (WSL). |
| 10 | ran | `check_loomora.py --compare` between refgen output of Windows x64 clang, Linux x86_64 g++, Linux i686 (`linux-gcc-i686.cmake`), Windows 2000 i686 (`mingw-w64-win2000-i686.cmake`, run on Windows 11) | `same` for the six files (sha256 `fcdf97f0...`, `8ca4dc6a...`, `b17920aa...`, `b32c8a39...`, `8bcd3ae9...`, `fc3fb5eb...`). Also `check_imports.py --arch x86` on the Windows 2000 build: refgen, format_selftest, selftest, `loomora.exe` pass the allowlist (3 to 5 imports rest on `verify` entries); `loomora_tests.exe` (a development program, not delivered) has 72 imports outside it. |

## not_tested (and why)

- Windows 2000 system (VM): no VM here. The Windows 2000 build only ran as a 32-bit process on Windows 11, so the Windows 2000 loader and the msvcrt exports marked `verify` are unconfirmed.
- Android (NDK r15c, device or emulator), Web (Emscripten fastcomp and wasm, Node or browser), Apple (macOS 10.6 osxcross, Xcode 5, iOS): toolchains not installed.
- Big-endian system or PowerPC emulation: no qemu.
- Windows Modern UI (WinUI 3): not built on this host. Linux GTK floor and GTK 4 variants, i686 Xlib, Xlib core font: not requested.
- Linux CentOS 5.11 / 6.10 container floors (gcc 4.8.2 devtoolset): the Docker daemon is not running; not started (explicit separate command).
- The full hour or 10 million inputs of step 8 (short run only); SC-005 performance (see `docs/implementation/validation/results/spec-002/perf_sc005.md`); the T101 review (docs repo `validation/bug-reports/format-review-2026-11-17.md`).
- `loomora_format_selftest.exe` of the Windows x86_64 MinGW build: `blocked_by_antivirus` (earlier result in `docs/implementation/validation/results/spec-002/cross_arch_results.md`), not retried.
- No evidence was stored in `docs/implementation` and no image was taken.

## FR-031 (the oldest toolchain of each floor built)

**Not checked.** Only part of it held: the backend builds as strict C++11 (`check_std` PASS with g++ 15.2 in WSL), and it builds and its tests pass with the Linux i686 toolchain file and with the MinGW-w64 Windows 2000 toolchain file (GCC 13-win32, the toolchain the project chose for that floor). The oldest toolchains of the other floors were not built: CentOS 5.11 (gcc 4.8.2), Android NDK r15c, Emscripten fastcomp, osxcross 10.6 and Xcode 5 (not installed or Docker not running). `fallbacks.json` consistency is checked by `validate_records` (0 errors against the docs branch).

## FR-032 (offline, without acceleration, without administrator rights)

**Checked, with this scope.** Administrator rights: no (Windows token not elevated, WSL uid 1000) for every run above. Offline: in WSL the self-test, `loomora_refgen`, `check_loomora.py --compare`, `loomora validate` and the Catch2 format tests ran in an empty network namespace (`unshare --net --user`, `ping` unreachable) with the same results; on the Windows host the network was not cut, but neither the format code nor the tools used here contain any network call (read of `core/src/format`, `cli/`, `tools/check_*.py`, `gen_*.py`). Acceleration: the format module and its tools do not touch graphics; the UI project modes run headless (Windows) or on the private Xvfb display (software X server); no GPU acceleration was configured or needed.

## Deviations and defects found

No defect in the code. Differences between the quickstart text and what the tools do (the quickstart steps 2 to 6 were then aligned with the tools, on 2026-11-18, and re-checked in WSL: commands, expected text and outcomes now match what the tools do):

1. Step 2: the self-test prints only `format selftest: ok`, not `sha256 = <hex>`, `0 differences` or `rejected N of N`.
2. Step 3: `loomora_refgen` takes the directory as a positional argument (no `--out`), writes `multi.loomora` (not `multi-full.loomora`), and the Windows command is `loomora-cli`. `validate` prints `OK ...: 0 errors, 0 warnings`, not the word `valid` or the manifest counts.
3. Step 3: the Catch2 test is named `SC-013: after adding one result every other entry is byte for byte the same` (not `editor_add_result_keeps_others`); version skew tests are `version skew: ...`.
4. Step 4: `check_readonly.py` takes `--build-dir`, not a file; it builds its own `basic` file.
5. Step 5: with `--quiet` the command prints only the summary, not the error code; the `large-*` files print `warn.count` only with the `--set-limit` of their `corpus.json` (default thresholds: 0 warnings). `check_cli_validate.py` is the check that applies both rules.
6. Step 6: `loomora_use_demo export FILE OUT full|light [ID...]` (no `--profile`/`--results`). The light mesh keeps identical geometry but not the same entry `sha256` when the full mesh has UVs (UVs dropped, `contracts/project-format.md`).
7. Observation (test environment, not a product defect): the Catch2 test `open_file_source on a named pipe fails at once...` creates its scratch directory relative to the working directory, so `loomora_tests` run from a Windows drive in WSL (`/mnt/c/...`, DrvFs without FIFO support) fails at `mkfifo` (`-1 == 0`). Run from a Linux directory (as `ctest` does) it passes: 6 of 6 runs.
8. Observation: the invalid corpus is not byte-identical between hosts (`valid-basic.loomora` differs, `reference_size` 3,235 on Windows, 3,228 in WSL, so 3,348 vs 3,341 files), because Python's deflate output differs between the two Python/zlib builds; each corpus validates correctly with its own tools.
