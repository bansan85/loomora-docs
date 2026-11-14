# Procedure: Android (Legacy and Modern UIs)

Replayable procedure for the Android Legacy UI (Java views, JNI glue, C++11 backend, parts A to D, run on 2026-11-14) and for the Android Modern UI (Kotlin, Jetpack Compose, the same JNI glue, parts E to H, run on 2026-11-15), as they were really run (FR-016, SC-004, SC-008):

| Platform record | Variant | Verdict | What was done |
|---|---|---|---|
| `android-4.1.2-x86` | legacy | `validation/verdicts/android-4.1.2-x86-legacy.json` | built for API 9, RUN on the API 16 x86 emulator (part C) |
| `android-2.3-arm` | legacy | `validation/verdicts/android-2.3-arm-legacy.json` | `armeabi` and `armeabi-v7a` built only, `not_tested` |
| `android-5.0-arm64` | legacy | `validation/verdicts/android-5.0-arm64-legacy.json` | `arm64-v8a` built only, `not_tested` |
| `android-7.0-x86_64` | modern | `validation/verdicts/android-7.0-x86_64-modern.json` | built for API 24, RUN on the API 24 x86_64 emulator (part G) |
| `android-2.3-arm`, `android-4.1.2-x86`, `android-5.0-arm64` | modern | `validation/verdicts/<id>-modern.json` | below the Modern floor (API 24), `not_tested` |

## Status of this document

- Written on 2026-11-14 from the commands that were run. One machine: the Windows 11 host of `validation/platforms/windows-11-x64.json`, no WSL (the Android Legacy build runs on the Windows host, ADR 0016).
- **The only system that was run is Android 4.1.2 (API 16) on x86, in the emulator.** The UI is built for API 9 (`minSdkVersion 9`), but the emulator cannot start below API 16, so Android 2.3 to 4.0 and every ARM ABI were NOT run. Read the verdicts as "built for API 9, run on API 16".
- Modern UI (Kotlin, API 24): see parts E to H. The `modern` verdicts of the three Legacy platforms are `not_tested` (below its floor, API 24).
- **Package id**: from 2026-11-15 the Java / Kotlin package is `fr.bansan.loomora.probe` (JNI exports `Java_fr_bansan_loomora_probe_Native_*`), Legacy applicationId `fr.bansan.loomora.probe`, Modern `fr.bansan.loomora.probe.modern`. The Legacy run of 2026-11-14 (parts B to D, and the evidence under `validation/evidence/android-legacy/`) used the earlier id `fr.loomora.probe`: that evidence is history and was not edited; the text of this procedure follows the new id. After the rename the Legacy APK was rebuilt (`python tools/build.py android legacy`) and its 15 vectors passed again on the API 24 emulator (not stored).

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

For each ABI (`armeabi`, `armeabi-v7a`, `x86`, `arm64-v8a`) it runs CMake + Ninja with `cmake/toolchains/android-ndk-r15c.cmake` (sysroot `platforms/android-9`, `android-21` for `arm64-v8a`; static backend `libloomora.a` with `-std=c++11`, `libloomora_jni.so`, `loomora_selftest`), checks `-std=c++11` in `compile_commands.json`, the `DT_NEEDED` list, the exported symbols (only `Java_fr_bansan_loomora_probe_Native_*`) and the undefined symbols (the link uses `--no-undefined` against the API 9 / 21 libraries), then runs `ui/legacy/android-views/build_apk.py` (javac, `d8 --min-api 9`, aapt, zipalign, apksigner v1 only) and writes `out/android-legacy/loomora.apk` (228,623 bytes), the self-tests in `out/android-legacy/selftest/<abi>/` and one deliverable record per Android platform record. The output of the last run is in `build-output-tail.txt` of the evidence folder.

## C. Run on the emulator (API 16, x86)

Start the emulator yourself, headless, ONE at a time, and always kill it at the end:

```
PS> $env:ANDROID_HOME = "E:\Program\Android\Sdk"
PS> Start-Process E:\Program\Android\Sdk\emulator\emulator.exe -ArgumentList "-avd","API16","-no-window","-no-audio","-no-snapshot","-no-boot-anim","-gpu","swiftshader_indirect"
PS> adb shell getprop sys.boot_completed            # wait for 1
```

1. **Self-test of spike S3 (x86 ABI)**: `adb push out/android-legacy/selftest/x86/loomora_selftest /data/local/tmp/`, then `adb shell "chmod 755 /data/local/tmp/loomora_selftest; /data/local/tmp/loomora_selftest; echo exit=$?"` prints `all passed` and `exit=0` (`selftest-x86-api16.txt`). In Git Bash set `MSYS_NO_PATHCONV=1`, or the `/data/...` paths are rewritten.
2. **Probe, one input**: `adb install -r out/android-legacy/loomora.apk`, `adb logcat -c`, `adb shell am start -W -n fr.bansan.loomora.probe/.MainActivity --ez probe true` (add `--es probe_input '<text>'` to change the input), wait 5 s, `adb logcat -d -v raw -s LOOMORA_PROBE:I` prints the probe line; check it with `python tools/check_probe.py --input Coucou <file>` (`probe-default.txt`). The UI finishes by itself 2 s after the first display.
3. **The 15 reference vectors**: `python tools/run_vectors.py android-4.1.2-x86 legacy --evidence-dir <dir>` (needs `LOOMORA_ANDROID_SDK` for adb; `--serial` if several devices). The launcher installs the APK, then for each vector force-stops the app, clears logcat, starts the activity and reads the line (a long line is logged in numbered parts `#i/n` and joined). A text with a character outside the BMP, or longer than 1,500 bytes, goes through a file (`--es probe_input_file /data/local/tmp/loomora-probe-input.txt`): the `am` command of Android 4.1 crashes on a 4-byte UTF-8 sequence and adb refuses a 4,096-byte command. Result: 15 vectors, 15 passed, 16 skipped (`run-vectors-...-log.txt`, `-probe-lines.tsv`).
4. **Typing**: start the activity without probe mode, `adb shell input keyevent KEYCODE_MOVE_END`, six times `KEYCODE_DEL`, `adb shell input text Marie`, then `adb shell uiautomator dump /sdcard/ui.xml` and read the `text=` attributes: `Marie` and `Bonjour Marie` (`typing-uiautomator.txt`). The end-of-life banner is on screen.
5. **Kill the emulator**: `adb emu kill`, then `adb kill-server`; check with `Get-Process emulator, qemu*` that nothing is left.

The platform id of the probe line is `android-<ro.build.version.release>-<ro.product.cpu.abi>` (here `android-4.1.2-x86`), the same rule in `Probe.java`.

## D. What was NOT done

- No run on Android 2.3 to 4.0 (the emulator starts from API 16), on API 17 to 23, or on any ARM or ARM64 system or device: `armeabi`, `armeabi-v7a` and `arm64-v8a` are built and statically checked only.
- No screenshot, no person at the keyboard, no real phone, no store install, no dark or high-contrast theme, no rotation.

## E. Modern UI: tools and pinned versions (read on 2026-11-15)

| Tool | Version / place |
|---|---|
| Android Gradle Plugin | **9.5.1** (`ui/modern/android-compose/gradle/libs.versions.toml`; AGP 9 has built-in Kotlin, no `kotlin-android` plugin) |
| Gradle | **9.9.1**, wrapper (`gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar` and `.properties` are committed; the distribution is checked against `distributionSha256Sum d8a18a7a...`; the jar's sha256 is 621008af..., the published wrapper checksum). Made once with `gradle wrapper` of the 9.9.1 distribution downloaded from services.gradle.org (sha256 d8a18a7a841dd5df825d35a685fad6f2d7f12ac0c280e57b67c27b4a520ba228, verified) into `E:\Program\Android\_tools` |
| Kotlin | **2.3.21** (plugin `org.jetbrains.kotlin.plugin.compose` 2.3.21 = the Compose compiler) |
| Compose | BOM **2026.09.00** (`compose-ui`, `compose-foundation`, `compose-material3`; versions from the BOM, for example `animation-core` 1.12.1), `androidx.activity:activity-compose` **1.13.0** |
| Android SDK | `compileSdk` 37 (`platforms/android-37.0`), `targetSdk` 37, `minSdk` 24, `build-tools/36.0.0`, `cmake/3.22.1` |
| NDK | **r30** `30.0.16248370` (Clang 21.0.0), `ANDROID_STL=c++_static`, ABIs `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64` |
| JDK | Temurin **25.0.4** (`C:\Program Files\Eclipse Adoptium\jdk-25.0.4.101-hotspot`): **Gradle 9.9.1 and AGP 9.5.1 ran on it unchanged**, no JDK 21 was needed (`JAVA_HOME` set in the shell only) |
| Emulator | 37.2.13.0, AVD `API24`: system image `android-24` default `x86_64` (revision 8), 1,536 MB, no SD card, 1080x1920 at 420 dpi; the AVD files are in `H:\repos\loomora-android-avd` (`ANDROID_AVD_HOME`: the C: drive had too little room for the userdata partition) |

Nothing global is changed: `JAVA_HOME` and `ANDROID_HOME` / `LOOMORA_ANDROID_SDK` are set in the shell that runs the build. The Gradle cache is `~/.gradle`, never in the repository (`ui/modern/android-compose/.gitignore`: `.gradle/`, `.kotlin/`, `local.properties`, `app/.cxx/`; `build/` is in the root `.gitignore`).

## F. Modern UI: build (Windows host)

```
PS> $env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-25.0.4.101-hotspot"
PS> $env:LOOMORA_ANDROID_SDK = "E:\Program\Android\Sdk"
PS> python tools/build.py android modern
```

It makes once a throwaway signing key with `keytool` in `build/android-modern-throwaway.keystore` (never committed, nothing secret) and runs `gradlew assembleRelease -Ploomora.keystore=...` in `ui/modern/android-compose`. Gradle's `externalNativeBuild` runs CMake 3.22.1 with the NDK r30 on `ui/modern/android-compose/app/CMakeLists.txt`, which adds `core/` (static `libloomora`, target property `CXX_STANDARD 11`, never global) and `bridge/jni/loomora_jni.cpp` (the same file as the Legacy UI) as `libloomora_jni.so` per ABI with `--exclude-libs,ALL`; R8 minifies and Gradle signs the APK (v2). The tool copies `out/android-modern/loomora.apk` (1,396,354 bytes), then checks inside the APK: `-std=c++11` in the `compile_commands.json` of the four ABIs for the backend and the glue; for each `libloomora_jni.so` the `DT_NEEDED` list (only `libc.so`, `libm.so`, `libdl.so`), the exports (only the six `Java_fr_bansan_loomora_probe_Native_*`) and the 16 KB alignment of the LOAD segments; `apksigner verify`, `zipalign -c -P 16`. It writes `validation/deliverables/android-7.0-x86_64-modern.json` (one record per Android platform record of API 24 or later). The same sources gave the same APK sha256 in three builds. `python tools/build.py android modern --min-sdk 21` builds with another minSdk (spike S8) and writes no record: it fails (part H).

The Modern UI sources are `Native.kt` (the `fr.bansan.loomora.probe.Native` class: `@JvmStatic external` functions that the glue exports), `MainActivity.kt` (Compose: `OutlinedTextField` "Coucou" kept to 1,000 code points, `Text` with the greeting from `Native.greet`, banner from `Native.eolWarning`), `Probe.kt` (intent extras, `VmHWM`, logcat tag `LOOMORA_PROBE`, 3,000-character chunks) and `Theme.kt` (system dark theme; high contrast through `UiModeManager.getContrast()` only behind `SDK_INT >= 34`, fallback `android-uimode-contrast`).

## G. Modern UI: run on the emulator (API 24, x86_64)

Create the AVD once (the image `system-images;android-24;default;x86_64` must be installed with `sdkmanager`), then start your own instance, headless, on its own port, so that another emulator (for example the API 16 one on 5554) is untouched:

```
PS> $env:ANDROID_HOME = "E:\Program\Android\Sdk"; $env:ANDROID_AVD_HOME = "H:\repos\loomora-android-avd"
PS> "no" | avdmanager create avd -n API24 -k "system-images;android-24;default;x86_64" -d pixel   # then hw.ramSize=1536 and hw.sdCard=no in config.ini
PS> Start-Process emulator.exe -ArgumentList "-avd","API24","-port","5556","-no-window","-no-audio","-no-snapshot","-no-boot-anim","-gpu","swiftshader_indirect"
PS> adb -s emulator-5556 shell getprop sys.boot_completed        # wait for 1, then: adb -s emulator-5556 shell input keyevent 82
```

Always address it with `adb -s emulator-5556`, and kill it at the end: `adb -s emulator-5556 emu kill`, then check with `Get-Process emulator, qemu*` that no process of it is left.

1. **Probe, one input**: `adb -s emulator-5556 install -r out/android-modern/loomora.apk`, `adb -s emulator-5556 logcat -c`, `adb -s emulator-5556 shell am start -W -n fr.bansan.loomora.probe.modern/fr.bansan.loomora.probe.MainActivity --ez probe true` (the component is written with the full class name: the class lives in the code namespace `fr.bansan.loomora.probe`, not in the applicationId), wait 5 s, `adb -s emulator-5556 logcat -d -v raw -s LOOMORA_PROBE:I`; `python tools/check_probe.py --input Coucou <file>` says OK (`probe-default.json`; 6 runs in `probe-default-runs.txt`: display_ms 243 / median 303.5 / 463, peak_rss_kb 68,616 to 69,060). `disabled_features` is `["uimode-contrast"]` on this API.
2. **The 15 reference vectors**: `python tools/run_vectors.py android-7.0-x86_64 modern --serial emulator-5556 --evidence-dir <dir>`. The Modern launcher (`AndroidModernLauncher` in `tools/run_vectors.py`, `out/android-modern/loomora.apk`, package `fr.bansan.loomora.probe.modern`) uses the same protocol as the Legacy one; the field limit counts code points. Result: 15 vectors, 15 passed, 16 skipped, 2 truncated to 1,000 characters, 58.4 s.
3. **Typing**: start the activity without probe mode, `input keyevent KEYCODE_MOVE_END`, six times `KEYCODE_DEL`, `input text Marie`, `uiautomator dump`: `Marie` and `Bonjour Marie` (`typing-uiautomator.txt`).
4. **Screenshots** (the owner authorized two): `adb shell screencap` returns blank images on this setup, so use `adb -s emulator-5556 emu screenrecord screenshot <dir>`. (1) default start, no probe mode, 4 s after `am start -W`; (2) mixed Unicode through probe mode: push a UTF-8 file with `Zo\u00e9 \u20ac \u0395\u03bb \u65e5\u672c` (the escapes decoded) to `/data/local/tmp/loomora-probe-input.txt`, `am start` (no `-W`) with `--ez probe true --es probe_input_file /data/local/tmp/loomora-probe-input.txt`, and the screenshot command 1.25 s after `am start` returned (shots at 0 to 0.75 s caught the launcher or the window animation, shots at 1.0 to 1.75 s caught the UI; the activity finishes 2 s after its first frame). Both images were looked at: not blank, the glyphs are drawn.

## H. Modern UI: spike S8 (minSdk 21) and what was NOT done

- **S8, Android half**: `python tools/build.py android modern --min-sdk 21` fails in the manifest merger: `androidx.compose.animation:animation-core-android:1.12.1` declares minSdk 23 (`spike-s8-minsdk21.txt`). The floor stays 24.
- No run on any API level but 24, and on no ARM or 32-bit x86 system: the `armeabi-v7a`, `arm64-v8a` and `x86` libraries are built and statically checked only.
- No dark theme, high contrast, rotation or TalkBack run; the API 34+ contrast path was not executed; no person at the keyboard, no real phone, no store install.
