# ADR 0008: Two thin native UIs per platform, each in the native language, calling only the C API

- **Status**: accepted; **amended on 2026-11-16**: the MAUI evaluation is done for Windows and Android; the decision to adopt MAUI is still open (see "Evaluation of .NET MAUI")
- **Date**: 2026-11-14
- **Spike / requirement**: FR-010, FR-020, FR-024 to FR-027, FR-029 / research R1, R12 (records decisions taken on 2026-11-12 and 2026-11-14)

## Context

Loomora must run from very old systems (Windows 2000, macOS 10.6, Android 2.3, iOS 5, November 2011 browsers) up to today. No single toolkit has that range, and the owner wants the most native interface the platform allows on the recent systems. ADR 0001 fixes the single C++11 backend and its C API; this ADR fixes what sits on the other side of that API.

## Decision

- **Two UIs per platform**: a **Legacy UI** that runs on the oldest systems and may disable features the system cannot provide (declared as fallbacks, ADR 0004), and a **Modern UI** that is as native as the platform allows. Below the floor of a Modern UI the Legacy UI is the answer; above it both can be installed.
- **Each UI is written in the native language of its platform** (C++ for Win32, Xlib and gtkmm; Objective-C for macOS and iOS Legacy; Swift for the Apple Modern UIs; Java and Kotlin for Android; C# for WinUI 3; HTML with ES5 or TypeScript for the Web) and holds **no logic**: every greeting, the end-of-life test and the probe record come from the backend (FR-029).
- **The only link is `core/include/loomora.h`**: a UI includes that header and no header of `core/src`; `tools/check_std.py` checks it. Interop per language: header and static library (C, C++, Objective-C), the same header through a module map (Swift), `loomora.dll` through `LibraryImport` (C#), JNI glue in `bridge/jni` (Java, Kotlin), Emscripten exports in `bridge/wasm` (Web).
- **WinUI 3 calls the C API from C# directly. C++/CLI is not used.**
- **State today**: the twelve UI directories exist (`ui/legacy/*`, `ui/modern/*`); their verification status is recorded per platform in `validation/verdicts/`. The UIs not yet verified are those listed in `specs/001-loomora-cross-platform/research.md`.
- **.NET MAUI is not adopted.** It was evaluated for Windows and Android (below). The two-native-UI decision above stands; no MAUI code replaces a native UI, and a MAUI spike lives only on its own branch.

## Evaluation of .NET MAUI

The owner asked to test MAUI and compare it with the Modern and Legacy UIs of each platform. The measurements below were taken on 2026-11-16 on Windows 11 and on one Android emulator. The adoption decision is left to a new ADR.

### Facts checked

- **Windows**: MAUI runs on WinUI 3 (Windows App SDK). It has the same Windows floor as the WinUI 3 Modern UI and cannot run on Windows 2000 or XP, so the Win32 Legacy UI is not replaced.
- **Android**: MAUI on .NET 11 runs on the emulator once the assemblies are embedded in the APK (`EmbedAssembliesIntoApk`). Without that setting, a Debug APK relies on Fast Deployment and aborts at launch (`No assemblies found`). The earlier `Failed to load System.Private.CoreLib.dll` came from the same cause, not from the CoreCLR runtime.
- **Linux**: Microsoft's GTK4 backend (`dotnet/maui-labs`, `platforms/Linux.Gtk4`) is **experimental, changes between versions, and is not officially supported by Microsoft**. It needs GTK 4, so it cannot serve the GTK 3.10 floor of the gtkmm Modern UI (ADR 0005, ADR 0006). The Avalonia MAUI backend (Linux and browser) is in preview. Neither is a supported production target.
- **Mobile runtime**: since .NET 11 Preview 6, CoreCLR is the only mobile runtime for MAUI; `UseMonoRuntime` is refused on .NET 11 (NETSDK1242).
- **Floors**: MAUI on Android needs API 24 or more (as in the spike); the Legacy floors (Android 2.3, iOS 5, macOS 10.6) are below what MAUI supports, so the Legacy UIs stay.
- **Web**: MAUI Blazor is not the PWA of ADR 0017 and is not evaluated.

### Measurements

Both Android builds are Debug (the only Android build of the spike), so they compare with each other but not with a Release build. Both Windows builds are Release. The MAUI Windows spike targets .NET 11; the WinUI 3 Modern UI targets .NET 10 in the repository, so the Windows comparison mixes two .NET versions.

Android, emulator API 35 x86_64, cold start measured with `am start -W` and memory with `dumpsys meminfo`:

| UI | APK | Cold start | Total PSS |
|---|---|---|---|
| Compose Modern (Kotlin, `ui/modern/android-compose`) | 12.1 MB | 7.3 s (first launch), then 2.6 s and 2.1 s | 68 MB |
| MAUI spike (.NET 11, embedded assemblies) | 134 MB (arm64 and x86_64) | 11.4 s (9.9 to 12.0 s over three runs) | 312 MB |

Windows 11, one window opened, measured 5 s after the window appeared:

| UI | Folder | Window shown | Working set | Private memory |
|---|---|---|---|---|
| WinUI 3 Modern (.NET 10, Release) | 36 MB | 2.2 s | 109 MB | 34 MB |
| MAUI spike (.NET 11, Release) | 160 MB | 3.8 s | 166 MB | 56 MB |

Not measured: iOS, macOS, Linux, web, the Legacy UIs (the MAUI floor excludes them), accessibility, high contrast, printing and the shared drawing surface (criteria 3 and 5 of the list above).

### Reading

- On both platforms, MAUI costs more size, start-up time and memory than the native Modern UI. On Android the gap is 4 to 5 times the memory and about 5 times the start-up time; on Windows about 1.5 times the memory and 1.8 times the start-up time. The Android figures are Debug builds; a Release build would narrow the gap but not remove it.
- MAUI does not replace the Legacy UIs, which target systems MAUI does not run on.
- The spike calls the same three C API functions as the Modern UIs, so it is not a test of the backend.

### Phases

Each phase runs on its own branch and is reported before the next one starts.

- **Phase 1, Windows**: done (above). Branch `spike/maui`.
- **Phase 2, Android**: done for the emulator (above), with the caveats in the measurements. No physical device tested.
- **Phase 3, Linux**: not started. MAUI GTK4 (experimental) against the gtkmm GTK 3.10 Modern UI, built in WSL in `$HOME`.
- **Phase 4, Apple**: not started. MAUI iOS and macOS against SwiftUI and the Legacy UIs. Needs a Mac; the macOS VM of ADR 0019 is not run in this phase.

### Recommendation

Keep the native UIs (this ADR, unchanged). The measurements do not justify adding MAUI on Windows or Android. A MAUI UI adopted for a platform needs a new ADR that supersedes the corresponding part of this one.

## Rejected alternatives

- **One UI per platform**: it cannot be at once compatible with the 2000-era systems and modern.
- **A cross-platform toolkit (Qt, Flutter, Compose Multiplatform, Avalonia)**: the floors are too high for the Legacy side, the result is not native, and some add a runtime. MAUI is not in this list: it is evaluated above.
- **C++/CLI for WinUI 3** (owner decision): a mixed managed layer with no benefit over the P/Invoke of a plain C API. C++/WinRT stays an option if .NET has to be avoided.
- **UI logic written per platform**: it would break the single implementation rule (FR-019) and the sharing measure.

## Consequences

- Twelve UIs in about nine languages; the shared share of each is measured by `tools/measure_sharing.py` (not written yet, T104).
- If MAUI is adopted for a platform, that platform has three UIs during the transition; the count and the languages are then restated here.
- Each language needs its own toolchain; the Windows, Linux and Android choices are ADR 0009, 0010, 0012 and research R3.
- Floors of the Modern UIs come from their toolkits and are recorded per platform (FR-026).
