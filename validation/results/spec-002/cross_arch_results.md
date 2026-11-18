# Cross-architecture results (spec 002, T059, T060, T009 old toolchains, T064)

Machine-readable version: `cross_arch_results.json`. Raw reference files are kept outside the repository
(`$HOME/loomora-build/xtarget-refs/<target>/` in WSL). Run on one Windows 11 x64 machine with WSL Ubuntu 26.04.

## Reference files

`basic`, `empty`, `escape40`, `multi`, `multi-light`, `one` written by `loomora_refgen`: the SHA-256 are identical on
every target that ran it (`python tools/check_loomora.py --compare` reports `same` for the six files).

| File | SHA-256 |
|---|---|
| basic | `fcdf97f0257076ba1c3ecae1867e57ebdc397440286a278f2cbe34eff8ce5f96` |
| empty | `8ca4dc6a2139539feb87f92b11cc0d5685fdff051fa044b692a7be5d984d7913` |
| escape40 | `b17920aabf9cc165a5c3b9290ba15f22e99903592426a8e55c96dc67ae570e2a` |
| multi | `8bcd3ae91876dcf6048c3a5dc689754a8eab90367faed373942b819a7c3f8607` |
| multi-light | `b32c8a390657d4f56ec9bd448e6c291e9781dfbb3b34b8164d049af67a7c4a8d` |
| one | `fc3fb5ebd929689bd7ee32da020b279ca81b3412c6c947eda8c9e7e3ddb846c5` |

## What ran where

| Target | Built | Ran | Where | format_selftest | Hashes |
|---|---|---|---|---|---|
| Linux x86_64, g++ 15.2 | yes | yes | WSL | pass (ctest 3/3) | reference |
| Linux i686 (`linux-gcc-i686.cmake`) | yes | yes | WSL, 32-bit process | pass (ctest 3/3, Catch2 included) | identical, with the SSE2 switch |
| Windows x86_64 MinGW-w64 | yes | `loomora_refgen.exe` only | Windows 11 | blocked_by_antivirus | identical |
| Windows 2000 i686 MinGW-w64 | yes | yes | Windows 11 (32-bit process, not a Windows 2000 system) | pass | identical, with the SSE2 switch |
| Windows 11 x64, clang 22.1.3 | yes | yes | Windows 11 | pass (`verify.py quick`) | identical |

`tools/verify.py quick` (T064): Windows 11 host 11 passed, 1 failed, 2 skipped; WSL 14 passed, 1 failed. The only failure on both is
`records`, expected: `fallbacks.json` is on the unmerged docs branch (it reports LOOMORA_FMT_LSEEK64, LOOMORA_FMT_NO_LARGE_FILES,
LOOMORA_FMT_WIN2000_SEEK and the new LOOMORA_TEST_FP_FLAGS as undeclared, and S-F1..S-F3 as not recorded).

## blocked_by_antivirus

On Windows 11, for the target Windows x86_64 MinGW-w64: `loomora_format_selftest.exe`, `loomora_selftest.exe` and
`loomora_spike_fmt_deps.exe` could not be run (`loomora_refgen.exe` of the same build runs). Errors: Git Bash `Permission denied`
(126); PowerShell "file not found" (the file had been removed after the launch attempt); `Start-Process`: access denied; through
WSL interop: `Invalid argument`. The same sources pass on Linux x86_64 and in the Windows clang x64 build. No workaround was
attempted; one rename probe of `loomora_format_selftest.exe` (`xt_fmt_check.exe`) was made before the no-evasion instruction
reached the run and was blocked the same way. This target's selftest remains unverified by a run.

## Fallbacks needed (to declare in `fallbacks.json`, T063)

- `x87-double-fixtures`, platforms `linux-legacy-i686` and `windows-2000-x86`, macro `LOOMORA_TEST_FP_FLAGS` (`-msse2 -mfpmath=sse`),
  set by the two i686 toolchain files and applied by `loomora_test_fp()` to the test programs only. Reason: 32-bit x86 GCC uses the
  x87 unit with 80-bit intermediates, the fixtures (`cos`, `sin`, `round3`, `0.1 + 0.2`) then differ and the bytes of `multi` and
  `multi-light` change (the other four files are identical without it). `libloomora` is not affected (library on x87 plus fixtures on
  SSE2 gives the reference bytes); `-ffloat-store` alone was not enough. The test programs need an SSE2 CPU.
- Condition variable shim, platforms `windows-2000-x86` and `windows-xp-x64`: libgcc `gthr-win32-cond.o` (pulled by
  `libstdc++.a(guard.o)`) imported `InitializeConditionVariable`, `SleepConditionVariableCS`, `WakeConditionVariable` and
  `WakeAllConditionVariable` (Windows Vista and later). `ui/legacy/windows-win32/getthreadid_shim.cpp` now defines their import entries
  (real function when kernel32 has it, polling wait otherwise). To confirm on the Windows 2000 VM.

## Windows 2000 import table

The check had been silently passing: with binutils 2.39 and later `tools/check_imports.py` parsed "0 imports" (fixed, self-test
extended). The real tables then showed file functions of the format module missing from `tools/win2000_imports.json` (added as
documented) and, on the toolchain of Ubuntu 26.04 (GCC 13.2 mingw-w64), the msvcrt names `__p___initenv`, `__p___mb_cur_max`,
`__p__iob`, `_lock`, `_unlock` (added as `verify`: not confirmed to exist in the msvcrt.dll of Windows 2000; confirm with
`check_imports.py --verify-dlls` on the VM). With these, the four programs of the x86 build and of the x64 build pass their allowlists.

## S-F1 (T009, MinGW old toolchain)

`loomora_spike_fmt_deps.exe` (win2000 toolchain) run on Windows 11: `S-F1: ok`, the zlib and RapidJSON hashes equal those of
`spike_results_sf.md` (section updated).

## not_tested

- Windows 2000 system (VM): no VM used. The Windows 11 runs prove bytes and logic, not the Windows 2000 loader or msvcrt exports.
- Android (NDK r15c, emulator or device): not installed.
- Web (emscripten fastcomp and wasm, Node): emscripten not installed.
- Big-endian (QEMU user mode PowerPC): qemu not installed.
- `loomora_format_selftest.exe` and `loomora_selftest.exe` of the Windows x86_64 MinGW build: blocked_by_antivirus (above).
