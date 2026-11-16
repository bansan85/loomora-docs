# ADR 0008: Two thin native UIs per platform, each in the native language, calling only the C API

- **Status**: accepted; **amended on 2026-11-16**: the MAUI exclusion is replaced by an evaluation (see "Evaluation of .NET MAUI")
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
- **.NET MAUI is not adopted and not excluded.** It is a candidate under evaluation (below). Until the evaluation concludes, the two-native-UI decision above stands unchanged, and no MAUI code replaces a native UI.

## Evaluation of .NET MAUI

The owner asked to test MAUI and compare it with the Modern and Legacy UIs of each platform. The comparison is recorded here, and the decision is taken afterwards by a new ADR.

Facts checked on 2026-11-16 (sources: Microsoft Learn, the Avalonia blog, the NuGet page of the package):

- **Windows**: MAUI runs on WinUI 3 (Windows App SDK). It has the same Windows floor as the WinUI 3 Modern UI and cannot run on Windows 2000 or XP, so the Win32 Legacy UI is not replaced.
- **Android, iOS, macOS**: MAUI draws with the platform controls. Its floors (OS and API levels) are not yet measured; the Legacy floors (Android 2.3, iOS 5, macOS 10.6) are below what MAUI supports, so the Legacy UIs stay.
- **Linux**: Microsoft's GTK4 backend (`dotnet/maui-labs`, `platforms/Linux.Gtk4`) is **experimental, changes between versions, and is not officially supported by Microsoft**. It needs GTK 4, so it cannot serve the GTK 3.10 floor of the gtkmm Modern UI (ADR 0005, ADR 0006). The Avalonia MAUI backend (Linux and browser) is in preview. Neither is a supported production target.
- **Web**: MAUI Blazor is not the PWA of ADR 0017 and is not evaluated in the first phase.

Comparison criteria, measured on each platform where a MAUI build exists, against the Modern UI and, for the floor, against the Legacy UI:

1. Build reproducibility and toolchain size (SDK, workloads, emulators).
2. Lowest OS version the app runs on (the floor).
3. Native behaviour: text input, accessibility, high contrast, file dialogs, printing, shared drawing surface (FR-027).
4. Logic kept out of the UI: only the C API is called, no logic written in C#.
5. Sharing measured by `tools/measure_sharing.py` (T104).
6. Package size and memory at start.

Phases, each on its own branch and reported before the next one starts:

- **Phase 1, Windows**: MAUI on Windows against the WinUI 3 Modern UI. Runs on the machine where the build is done (Windows 11).
- **Phase 2, Android**: MAUI Android against the Compose Modern UI and the Views Legacy UI (floor only). Needs an Android SDK and, for emulation, a device or an emulator image.
- **Phase 3, Linux**: MAUI GTK4 (experimental) against the gtkmm GTK 3.10 Modern UI, built in WSL in `$HOME`.
- **Phase 4, Apple**: MAUI iOS and macOS against SwiftUI and the Legacy UIs. Needs a Mac; the macOS VM of ADR 0019 is not run in this phase.

A MAUI UI adopted for a platform needs a new ADR that supersedes the corresponding part of this one. Until then no MAUI UI is shipped.

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
