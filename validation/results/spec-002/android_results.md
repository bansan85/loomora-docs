# Android results (spec 002, T009 old toolchain, T060, T091)

Machine-readable version: `android_results.json`. Run on 2026-11-17 on the Windows 11 x64 host (Windows hypervisor on, emulator
37.2.13.0 on WHPX, headless), branch `feat/fmt-android`.

## Installed

- **NDK r15c** (15.2.4203891, Clang 5.0.300080): `sdkmanager` no longer has it (oldest is 16.1), so
  `https://dl.google.com/android/repository/android-ndk-r15c-windows-x86_64.zip` (849,733,996 bytes, SHA-1
  `970bb2496de0eada74674bb1b06d79165f725696`, SHA-256 `6f3785cb...bcd8d`). No published checksum was found (the docs give none, the Google
  repository XML does not list r15c): the hashes are the measured ones; the archive extracted cleanly. It is used where it was
  extracted (`...\Android\ndk-r15c-extract\android-ndk-r15c`): moving the folder was refused (Permission denied), not retried.
- **AVDs** (default location `~/.android/avd`, `pixel` profile), images from `sdkmanager` (it still lists the API 16 x86 image: no
  manual download was needed): `API16` = android-16 default x86 (rev 7, 1 GB), `API24` = android-24 default x86_64 (rev 8, 1.5 GB),
  `API35` = android-35 default x86_64 (rev 2, 2 GB; plain AOSP, 64-bit only). All three boot with WHPX.

## Reference files (`loomora_refgen` run on the emulator, pulled, hashed)

The six SHA-256 are the ones of `cross_arch_results.md` on **every** run below (basic `fcdf97f0...`, empty `8ca4dc6a...`, escape40
`b17920aa...`, multi `8bcd3ae9...`, multi-light `b32c8a39...`, one `fc3fb5eb...`): no determinism difference, no switch needed (Clang
x86 uses SSE2).

| Build | Runs on | selftest | format_selftest | S-F1 (T009) | 6 hashes |
|---|---|---|---|---|---|
| x86, built for API 9, not PIE | API 16 (Android 4.1.2) | all passed | ok | ok, same 3 hashes as the host | identical |
| x86, API 16 sysroot, PIE | API 24 (7.0, 32-bit process) | all passed | ok | ok | identical |
| x86_64, API 21 sysroot, PIE | API 24 and API 35 (15) | all passed | ok | ok | identical |
| armeabi, armeabi-v7a, arm64-v8a | not run | built, ELF checked | built | built | not run |

## UI (T091) and regression

APKs: Legacy 1,472,418 bytes (4 ABIs, NDK r15c, API 9), Modern 3,388,002 bytes (4 ABIs, NDK r30, Gradle). The 15 reference vectors
pass on Legacy API 16 and API 24 (`--ui-abi x86`) and on Modern API 24 and API 35. `validate_project` and `roundtrip_project`, compared
with `loomora validate --quiet` (WSL g++ build of this branch): 37 of 37 runs on each of the same four UI/API pairs (6 references,
9 valid corpus files, an invalid file, a missing path, both modes, 3 usage rows), and the whole corpus (113 files, validate) on Legacy
API 16 and Modern API 24: 116 of 116 each. The T091 Java, Kotlin and JNI code needed no fix.

## Fixes

1. `file_posix.cpp`, LOOMORA_FMT_LSEEK64 (API < 21) did not compile with r15c: `pread64`/`pwrite64` exist from API 12 and `fstat64` from
   API 21, and the libc of API 9 to 11 has no `pread64`. Now: API < 12 uses the `pread64`/`pwrite64` system calls (ARM EABI padding
   argument), API < 21 uses `fstat`/`struct stat` (64-bit `st_size`), API 12 to 20 uses `pread64`/`pwrite64`.
2. gnustl has no `std::to_string` and no `std::snprintf`: local `dec()` helper in `image_meta.cpp`, global `snprintf` in two test files.
3. `loomora_refgen` is built for Android; `tools/build.py` copies it and `loomora_format_selftest` to `out/android-legacy/selftest/<abi>/`.
4. Toolchain `android-ndk-r15c.cmake`: `LOOMORA_ANDROID_PIE` (Android 5.0+ refuses non-PIE executables) and ABI `x86_64`, both only used
   for test programs of API >= 21 devices; default build unchanged.
5. The Modern Gradle build did not find RapidJSON and zlib (source directory is not the repository root): variables set in its
   `CMakeLists.txt`, honoured by `core/CMakeLists.txt` and `cmake/loomora_zlib.cmake`.
6. `tools/run_vectors.py`: Android project modes (`run_project`), without `am start -W` (it never returns for an activity that finishes at
   once), status read from the line, self-tested.

## Not tested

- Any ARM ABI run (no ARM system). The ARM EABI padding argument of the API < 12 `pread64` call is therefore never executed.
- Android 2.3 to 4.0 (the emulator starts from API 16): the x86 build is built for API 9, run on API 16 and 24 only.
- Files above 2 GiB on Android: the fallback `android-api9-lseek64` is declared, not run (5 GiB tests are host only).
- Legacy UI on API 35 (outside its range; the image has no 32-bit ABI), Modern below API 24, API 17 to 23 and 25 to 34.
- Records in the docs repository (fallback text of `android-api9-lseek64`, verdicts, S-F1): a later task.
- `blocked_by_antivirus`: none for adb, emulator, NDK or test binaries; only the NDK folder rename above.
