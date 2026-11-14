# Procedure: Android (Legacy UI)

Replayable procedure for the Android Legacy UI (Java views, JNI glue, C++11 backend), as it was really run on 2026-11-14 (FR-016, SC-004, SC-008):

| Platform record | Variant | Verdict | What was done |
|---|---|---|---|
| `android-4.1.2-x86` | legacy | `validation/verdicts/android-4.1.2-x86-legacy.json` | built for API 9, RUN on the API 16 x86 emulator (part C) |
| `android-2.3-arm` | legacy | `validation/verdicts/android-2.3-arm-legacy.json` | `armeabi` and `armeabi-v7a` built only, `not_tested` |
| `android-5.0-arm64` | legacy | `validation/verdicts/android-5.0-arm64-legacy.json` | `arm64-v8a` built only, `not_tested` |

## Status of this document

- Written on 2026-11-14 from the commands that were run. One machine: the Windows 11 host of `validation/platforms/windows-11-x64.json`, no WSL (the Android Legacy build runs on the Windows host, ADR 0016).
- **The only system that was run is Android 4.1.2 (API 16) on x86, in the emulator.** The UI is built for API 9 (`minSdkVersion 9`), but the emulator cannot start below API 16, so Android 2.3 to 4.0 and every ARM ABI were NOT run. Read the verdicts as "built for API 9, run on API 16".
- Modern UI (Kotlin, API 24): not written (T055), the `modern` verdicts are `not_tested` (below its floor).

## A. Tools (versions read on 2026-11-14)

| Tool | Version / place |
|---|---|
| Android NDK | **r15c** (15.2.4203891), `E:\Program\Android\android-ndk-r15c` (Clang 5.0.300080, GCC 4.9, `platforms/android-9`). r16b (`Sdk\ndk\16.1.4479499`) has no android-9; r30 is for the Modern UI |
| Android SDK | `E:\Program\Android\Sdk`: `platforms/android-9`, `build-tools/36.0.0` (`aapt`, `d8`, `zipalign`, `apksigner`; no `dx`), `platform-tools` (adb), `emulator` 37.2.13.0, system image `android-16` default x86 (revision 7) |
| JDK | Temurin 25.0.4 (`JAVA_HOME`); `javac -source 8 -target 8` |
| Python, CMake, Ninja | 3.13.17, 4.4.0, 1.12.1 (Python 3.8+ is enough) |
| AVD | `API16`: x86, 1,024 MB, 1080x1920 at 480 dpi |

No global environment variable is changed: the paths are given in the shell that runs the build.

## B. Build (Windows host)

```
PS> $env:LOOMORA_ANDROID_NDK = "E:\Program\Android\android-ndk-r15c"
PS> $env:LOOMORA_ANDROID_SDK = "E:\Program\Android\Sdk"
PS> python tools/build.py android legacy
```

For each ABI (`armeabi`, `armeabi-v7a`, `x86`, `arm64-v8a`) it runs CMake + Ninja with `cmake/toolchains/android-ndk-r15c.cmake` (sysroot `platforms/android-9`, `android-21` for `arm64-v8a`; static backend `libloomora.a` with `-std=c++11`, `libloomora_jni.so`, `loomora_selftest`), checks `-std=c++11` in `compile_commands.json`, the `DT_NEEDED` list, the exported symbols (only `Java_fr_loomora_probe_Native_*`) and the undefined symbols (the link uses `--no-undefined` against the API 9 / 21 libraries), then runs `ui/legacy/android-views/build_apk.py` (javac, `d8 --min-api 9`, aapt, zipalign, apksigner v1 only) and writes `out/android-legacy/loomora.apk` (228,623 bytes), the self-tests in `out/android-legacy/selftest/<abi>/` and one deliverable record per Android platform record. The output of the last run is in `build-output-tail.txt` of the evidence folder.

## C. Run on the emulator (API 16, x86)

Start the emulator yourself, headless, ONE at a time, and always kill it at the end:

```
PS> $env:ANDROID_HOME = "E:\Program\Android\Sdk"
PS> Start-Process E:\Program\Android\Sdk\emulator\emulator.exe -ArgumentList "-avd","API16","-no-window","-no-audio","-no-snapshot","-no-boot-anim","-gpu","swiftshader_indirect"
PS> adb shell getprop sys.boot_completed            # wait for 1
```

1. **Self-test of spike S3 (x86 ABI)**: `adb push out/android-legacy/selftest/x86/loomora_selftest /data/local/tmp/`, then `adb shell "chmod 755 /data/local/tmp/loomora_selftest; /data/local/tmp/loomora_selftest; echo exit=$?"` prints `all passed` and `exit=0` (`selftest-x86-api16.txt`). In Git Bash set `MSYS_NO_PATHCONV=1`, or the `/data/...` paths are rewritten.
2. **Probe, one input**: `adb install -r out/android-legacy/loomora.apk`, `adb logcat -c`, `adb shell am start -W -n fr.loomora.probe/.MainActivity --ez probe true` (add `--es probe_input '<text>'` to change the input), wait 5 s, `adb logcat -d -v raw -s LOOMORA_PROBE:I` prints the probe line; check it with `python tools/check_probe.py --input Coucou <file>` (`probe-default.txt`). The UI finishes by itself 2 s after the first display.
3. **The 15 reference vectors**: `python tools/run_vectors.py android-4.1.2-x86 legacy --evidence-dir <dir>` (needs `LOOMORA_ANDROID_SDK` for adb; `--serial` if several devices). The launcher installs the APK, then for each vector force-stops the app, clears logcat, starts the activity and reads the line (a long line is logged in numbered parts `#i/n` and joined). A text with a character outside the BMP, or longer than 1,500 bytes, goes through a file (`--es probe_input_file /data/local/tmp/loomora-probe-input.txt`): the `am` command of Android 4.1 crashes on a 4-byte UTF-8 sequence and adb refuses a 4,096-byte command. Result: 15 vectors, 15 passed, 16 skipped (`run-vectors-...-log.txt`, `-probe-lines.tsv`).
4. **Typing**: start the activity without probe mode, `adb shell input keyevent KEYCODE_MOVE_END`, six times `KEYCODE_DEL`, `adb shell input text Marie`, then `adb shell uiautomator dump /sdcard/ui.xml` and read the `text=` attributes: `Marie` and `Bonjour Marie` (`typing-uiautomator.txt`). The end-of-life banner is on screen.
5. **Kill the emulator**: `adb emu kill`, then `adb kill-server`; check with `Get-Process emulator, qemu*` that nothing is left.

The platform id of the probe line is `android-<ro.build.version.release>-<ro.product.cpu.abi>` (here `android-4.1.2-x86`), the same rule in `Probe.java`.

## D. What was NOT done

- No run on Android 2.3 to 4.0 (the emulator starts from API 16), on API 17 to 23, or on any ARM or ARM64 system or device: `armeabi`, `armeabi-v7a` and `arm64-v8a` are built and statically checked only.
- No screenshot, no person at the keyboard, no real phone, no store install, no dark or high-contrast theme, no rotation.
- The Modern UI (T055) is not written.
