# ADR 0009: The Windows Legacy UI is plain Win32 in C++, not MFC

- **Status**: accepted
- **Date**: 2026-11-14 (decision of the owner taken on 2026-11-12, third Clarifications session of the specification)
- **Spike / requirement**: S1 / FR-020, research R6

## Context

The requirements document planned an MFC Legacy UI for Windows. Visual Studio 2005 SP1 is the last MSVC that produces binaries for Windows 2000 (later versions need APIs of XP SP2 and side-by-side manifests), and it has no C++11, while the backend is C++11 (ADR 0001). No MSVC that supports C++11 targets Windows 2000. MFC therefore cannot be combined with the backend on that system.

## Decision

- The Windows Legacy UI (`ui/legacy/windows-win32`) is written in **C++11 on the plain Win32 API**: `CreateWindowEx`, an `EDIT` control and `STATIC` controls, `-mwindows`, no MFC and no other framework.
- **One source** serves Windows 2000 to 11. It is limited to the Win32 calls that exist on Windows 2000 (Unicode `W` functions, `_WIN32_WINNT=0x0500`); anything newer is looked up at run time with `GetProcAddress` and skipped when absent (`ui/legacy/windows-win32/dynfeatures.cpp`, `dynload.h`), each skipped feature declared as a fallback (ADR 0004).
- It is built with the same compiler as the backend, MinGW-w64 GCC (ADR 0012), in two executables: i686 (Windows 2000 to 11) and x86_64 (XP x64 to 11).
- `tools/win2000_imports.json` lists the functions an executable may import, and `tools/check_imports.py` fails the build on any other.

## Rejected alternatives

- **MFC with Visual Studio 2005**: no C++11, a second Windows compiler, and the single toolchain of ADR 0012 would be lost.
- **MFC with a recent Visual Studio**: its binaries do not run on Windows 2000.
- **A toolkit on top of Win32 (wxWidgets, Qt)**: more dependencies and a runtime for a window with two controls; floors too high for Qt.

## Consequences

- The deviation from the requirements document (MFC) is to report in the validation report.
- The UI draws its own dark theme, DPI scaling and fonts (ADR 0004 and `fallbacks.json`: `dark-mode`, `dpi-awareness`, `message-font`, `visual-styles`) since there is no framework to do it.
- The WinUI 3 Modern UI is unchanged (ADR 0008).
