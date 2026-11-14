# Engine and implementation overview

This page describes how Loomora is built: the engine (the backend), the code layout, the technical rules and where the decisions are recorded. The decisions themselves are the numbered records in this folder; the product description is in the root `README.md`.

## The engine

- One backend, written in strict C++11 (namespace `loomora`), in `core/`. Every interface uses it and none of them computes anything.
- The only way in is a handwritten C API, `core/include/loomora.h`. The backend is the static library `libloomora.a`. Interfaces never include anything from `core/src`.
- The backend is always compiled as C++11 (`CXX_STANDARD 11`, `CXX_STANDARD_REQUIRED ON`, `CXX_EXTENSIONS OFF`), set as a target property and never globally, so the standard of an interface never leaks into it.
- Tests are in `core/tests/`: Catch2, `selftest` and the reference vectors in `vectors.json`.
- At this stage the engine only builds the probe record and the greeting "Bonjour " followed by the input.

## Code layout

| Path | Content |
|---|---|
| `core/` | the backend, its C API and its tests |
| `bridge/` | glue with no logic: `jni/` (Android, built by the Android Legacy UI), `wasm/` (Emscripten, not built yet) |
| `ui/legacy/`, `ui/modern/` | one folder per platform and UI variant, each in the native language, with no logic |
| `cmake/toolchains/` | toolchain files: MinGW-w64 (Windows 2000 i686, Windows x86_64), Linux i686 and Android NDK r15c |
| `tools/` | Python host tools: `verify.py`, `build.py`, `run_vectors.py`, `check_std.py`, `check_imports.py`, `check_probe.py`, `check_ui_logic.py`, `validate_records.py`, `measure_sharing.py`, `prefix_check.py`, `report.py`, `diff_verdicts.py`, and `trials/` (drivers that produced the evidence) |
| `specs/001-loomora-cross-platform/` | specification, plan, research, tasks, quickstart, contracts |
| `docs/implementation/` | this git submodule (`../loomora-docs`): the ADRs and `validation/` (platforms, verdicts, deliverables, fallbacks, evidence, procedures, report) |
| `third_party/catch2/` | git submodule, Catch2 (tests only) |

## Interfaces

Each interface is native and thin. Four exist: Windows Legacy (plain Win32, C++11), Linux Legacy (Xlib with Xft, C++11), Linux Modern (gtkmm, up to C++17) and Android Legacy (Java views and JNI glue, API 9 to 23). The Legacy ones target very old systems (down to Windows 2000), the Modern ones current systems.

## Technical rules

- Legacy interfaces are C++11; Modern ones may use up to C++17 (for example gtkmm 4 needs it).
- Linux interfaces are C++ built with CMake; Meson and Makefiles are not used.
- Linux binaries are compiled on the machine that runs them. Windows binaries are cross-compiled with MinGW-w64 from WSL.
- A fallback applies only to the problematic case (platform, compiler, OS version, architecture), never to everyone, and is declared in `validation/fallbacks.json`.
- The minimum GTK version is 3.10; newer features sit behind compile-time macros.
- Evidence is stored only when it proves something new. No image is added without the owner's authorization.
- `python tools/verify.py quick` verifies any branch; `full` only for a graphical change.

The complete rules are in `CLAUDE.md` at the root of the repository. Building and running are described in `docs/building/README.md`.

## Decision records

| Record | Subject |
|---|---|
| 0001 | backend in C++11 behind a C API |
| 0002 | Windows 2000 `GetThreadId` shim |
| 0003 | Windows 2000 VM and Unicode launcher |
| 0004 | fallback per case |
| 0005 | GTK version policy |
| 0006 | Linux Modern interface in gtkmm with CMake |
| 0007 | Linux builds on the target machine |
| 0008 | native interfaces per platform, Legacy and Modern |
| 0009 | Win32 instead of MFC for Windows Legacy |
| 0010 | Linux interfaces in C++ with CMake |
| 0011 | C++ standards per target |
| 0012 | MinGW-w64 cross toolchain from WSL for Windows |
| 0013 | verification levels, `quick` and `full` |
| 0014 | evidence rules |
| 0015 | Python host tools: Python 3.8+, standard library only |
| 0016 | Android Legacy: NDK r15c, CMake toolchain file and scripted APK build for API 9 |
