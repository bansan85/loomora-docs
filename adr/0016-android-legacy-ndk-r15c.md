# ADR 0016: The Android Legacy UI is built with NDK r15c, a CMake toolchain file and a scripted APK build, for API 9

- **Status**: accepted
- **Date**: 2026-11-14
- **Spike / requirement**: S3 / FR-003, FR-005, FR-022, research R3 (corrected)

## Context

The decided floor of the Android Legacy UI is Android 2.3 (API 9, spec Clarifications). Research R3 and tasks T029 and T081 planned NDK r16b with `APP_PLATFORM := android-9` and `ndk-build`. The tools installed on the build machine (Windows 11 host, no WSL for this target) were measured on 2026-11-14:

- **r16b** (`ndk/16.1.4479499`) has `platforms/android-14` and above only (14 directories, from android-14): there is no `android-9` in it. The planned NDK cannot target API 9.
- **r15c** (`android-ndk-r15c`, 15.2.4203891) still has `platforms/android-9` with `arch-arm`, `arch-mips` and `arch-x86` (and android-12 to 26 for the rest), the unified headers (`sysroot/usr/include`), Clang 5.0.300080, GCC 4.9, and `armeabi`. But its `ndk-build` raises every `APP_PLATFORM` below 14 to 14: `build/core/init.mk` line 491, `NDK_MIN_PLATFORM_LEVEL := 14`. `Android.mk` / `Application.mk` therefore cannot produce an API 9 library with r15c either.
- The Android emulator of this machine cannot run below API 16 (owner decision): the x86 system image `android-16` is the oldest that can be executed. No ARM system is run.

## Decision

1. **NDK r15c is used for the Legacy UI**, **without ndk-build and without the CMake support of the NDK**. `cmake/toolchains/android-ndk-r15c.cmake` (replaces the planned `android-ndk-r16b.cmake`, T029) drives the r15c Clang directly: `-target <triple>`, `--sysroot=platforms/android-9/arch-<arch>` (`android-21` for `arm64-v8a`, the first level with that ABI), `-isystem sysroot/usr/include` and the per-arch directory of the unified headers, `-D__ANDROID_API__=9`, `-gcc-toolchain` of the NDK's GCC 4.9 for the linker and libgcc. The ABIs are the parameter `ANDROID_ABI`: `armeabi` (`armv5te-none-linux-androideabi`, `-march=armv5te -mthumb`), `armeabi-v7a` (`armv7`, `-mfloat-abi=softfp -mfpu=vfpv3-d16`), `x86` (`i686-none-linux-android`, `-mstackrealign`), `arm64-v8a` (`aarch64-none-linux-android`). The backend `libloomora.a` stays `CXX_STANDARD 11` through its own target property; the JNI glue and the UI libraries are `-std=c++11` too.
2. **C++ runtime**: the backend uses `std::string` and exceptions, so a runtime is linked, **statically**: `libgnustl_static.a` and `libsupc++.a` (GNU libstdc++ 4.9 of the NDK) into `libloomora_jni.so`; no shared STL is shipped. Linking `libgnustl` instead of libc++ was chosen because it needs only the API 9 libc (`DT_NEEDED` measured: `libc.so`, plus `libdl.so` on x86 and arm64-v8a; nothing else, in particular no `libstdc++.so` and no `libgnustl_shared.so`). libc++ was not tried. Every library links with `-Wl,--no-undefined` against the API 9 libraries (21 for arm64), so a symbol missing from them stops the build; `tools/build.py` also fails on any `DT_NEEDED` outside `libc libm libdl liblog libstdc++` and on any exported symbol that is not `Java_fr_loomora_probe_Native_*`.
3. **One switch for one case** (ADR 0004): the `armeabi` (ARMv5TE) library is linked with `-latomic` because ARMv5TE has no `ldrex`/`strex`, so Clang turns the atomic reference counts of gnustl into calls to `__atomic_fetch_add_4`, which fail to link otherwise (measured). It is declared as fallback `android-armeabi-libatomic` of the platform `android-2.3-arm` in `validation/fallbacks.json`; the other ABIs do not take it.
4. **Java side**, scripted, no Gradle (`ui/legacy/android-views/build_apk.py`): `javac -source 8 -target 8` against `platforms/android-9/android.jar` (a JDK 25 has no Java 7 output; the source uses no Java 7+ feature, and only API 9 classes compile because the compile class path is the API 9 jar); `d8 --min-api 9` of build-tools 36.0.0 (R8 8.10.9) converts to `classes.dex`: **it accepts `--min-api 9`** (and `--min-api 1`), dex header `dex\n035`, so `dx` and build-tools 30.0.3 are not needed (build-tools 36.0.0 has no `dx`); `aapt` packages the manifest (`minSdkVersion 9`, `targetSdkVersion 23`, no resources); the stripped libraries and the dex are added; `zipalign -p 4`; `apksigner --v1-signing-enabled true --v2-signing-enabled false --v3-signing-enabled false --min-sdk-version 9` (SHA-1 digests, the only scheme Android 2.3 reads) with a debug key made once by `keytool`. One APK holds the four ABIs (228,623 bytes).
5. `tools/build.py android legacy` replaces `build.sh` (T081/T082): it runs on the Windows host (PowerShell / Python), builds each ABI in `build/android-legacy-<abi>`, the self-test per ABI, then the APK, and writes one deliverable record per Android platform record. `tools/run_vectors.py` has an Android launcher (adb, logcat tag `LOOMORA_PROBE`).
6. **MIPS**: the toolchain file also builds `mips` (`-mips32`; measured: it links, 1,076,576 bytes), but the data model has no `mips` architecture and the spec leaves MIPS out of scope, so it is not in the APK.

## Rejected alternatives

- **NDK r16b**: no `android-9` platform (measured), so it cannot target the floor.
- **ndk-build with r15c**: clamps `APP_PLATFORM` to 14 (measured, `init.mk`), so the floor is not reached.
- **Raising the floor to API 14** (what ndk-build allows): it would drop Android 2.3 to 4.0 for no technical reason, against the owner's decision.
- **Android Gradle Plugin / CMake support of a recent NDK**: refuse API levels this low (research R3).
- **libc++ static / a shared STL**: not needed once gnustl static linked with only the API 9 libc; a shared STL would have to be installed next to the library.

## Consequences

- **What was run** (spike S3): the `x86` self-test (`loomora_selftest`, built for API 9) and the APK ran on the emulator `API16` (Android 4.1.2, API 16, x86): `all passed`, and 15 of 15 reference vectors through the UI. So the x86 build is **built for API 9 and run on API 16**, never validated on Android 2.3. `armeabi`, `armeabi-v7a` and `arm64-v8a` are built, linked and size/symbol checked only; their verdicts are `not_tested` (no ARM system, the emulator cannot run below API 16). `armeabi` shows `Tag_FP_arch: VFPv2` in `readelf -A` although it is built soft-float ARMv5TE (probably an object of the prebuilt gnustl / libgcc): it was not investigated and not run, so an illegal instruction on a CPU without VFP is not excluded.
- The sizes measured: libraries unstripped 936,008 (armeabi), 879,740 (armeabi-v7a), 938,812 (x86), 1,214,528 (arm64-v8a) bytes; in the APK, stripped, 95,716 / 87,532 / 136,708 / 141,152 bytes; `classes.dex` 12,484 bytes; APK 228,623 bytes.
- Research R3, T029, T081, T082, the plan, the quickstart and the UI contract table say r15c, not r16b; `validation/sources.json` keeps the old claim of `ndk-revisions` with a note. ADR 0011 still names r16b in passing: this record supersedes that detail.
- `core/tests/selftest.cpp` no longer uses `std::to_string`: the gnustl of the NDK (`_GLIBCXX_USE_C99` undefined) lacks it (it also includes `<cstdio>` unconditionally, it used `std::fputs` through `<iostream>` only).
- The Modern UI (T055, NDK r30, API 24) shares `bridge/jni/loomora_jni.cpp` and is unaffected; r16b is not used by anything.
- Not covered: the ARM libraries on a real or emulated ARM CPU, Android 2.3 to 4.0 on any CPU, the `x86` build on a system older than API 16, API 17 to 23 and 24+ on x86 (no other image was started).
