# ADR 0008: Two thin native UIs per platform, each in the native language, calling only the C API

- **Status**: accepted
- **Date**: 2026-11-14
- **Spike / requirement**: FR-010, FR-020, FR-024 to FR-027, FR-029 / research R1, R12 (records decisions taken on 2026-11-12 and 2026-11-14)

## Context

Loomora must run from very old systems (Windows 2000, macOS 10.6, Android 2.3, iOS 5, November 2011 browsers) up to today. No single toolkit has that range, and the owner wants the most native interface the platform allows on the recent systems. ADR 0001 fixes the single C++11 backend and its C API; this ADR fixes what sits on the other side of that API.

## Decision

- **Two UIs per platform**: a **Legacy UI** that runs on the oldest systems and may disable features the system cannot provide (declared as fallbacks, ADR 0004), and a **Modern UI** that is as native as the platform allows. Below the floor of a Modern UI the Legacy UI is the answer; above it both can be installed.
- **Each UI is written in the native language of its platform** (C++ for Win32, Xlib and gtkmm; Objective-C for macOS and iOS Legacy; Swift for the Apple Modern UIs; Java and Kotlin for Android; C# for WinUI 3; HTML with ES5 or TypeScript for the Web) and holds **no logic**: every greeting, the end-of-life test and the probe record come from the backend (FR-029).
- **The only link is `core/include/loomora.h`**: a UI includes that header and no header of `core/src`; `tools/check_std.py` checks it. Interop per language: header and static library (C, C++, Objective-C), the same header through a module map (Swift), `loomora.dll` through `LibraryImport` (C#), JNI glue in `bridge/jni` (Java, Kotlin), Emscripten exports in `bridge/wasm` (Web).
- **WinUI 3 calls the C API from C# directly. C++/CLI is not used.**
- **State today**: only three UIs exist (Windows Legacy, Linux Legacy, Linux Modern); the others are decided in `specs/001-loomora-cross-platform/research.md` and are not written or verified.

## Rejected alternatives

- **One UI per platform**: it cannot be at once compatible with the 2000-era systems and modern.
- **A cross-platform toolkit (Qt, Flutter, Compose Multiplatform, MAUI, Avalonia)**: the floors are too high for the Legacy side, the result is not native, and some add a runtime.
- **C++/CLI for WinUI 3** (owner decision): a mixed managed layer with no benefit over the P/Invoke of a plain C API. C++/WinRT stays an option if .NET has to be avoided.
- **UI logic written per platform**: it would break the single implementation rule (FR-019) and the sharing measure.

## Consequences

- Twelve UIs in about nine languages; the shared share of each is measured by `tools/measure_sharing.py` (not written yet, T104).
- Each language needs its own toolchain; the Windows, Linux and Android choices are ADR 0009, 0010, 0012 and research R3.
- Floors of the Modern UIs come from their toolkits and are recorded per platform (FR-026).
