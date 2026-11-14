# ADR 0010: Both Linux UIs are C++ built with CMake: Xlib for Legacy, gtkmm for Modern

- **Status**: accepted (the Modern half is detailed by ADR 0005 and ADR 0006)
- **Date**: 2026-11-14
- **Spike / requirement**: owner decision of 2026-11-14 / FR-020, research R7, R9

## Context

The first Linux UIs were written in C: the Legacy one with Xlib and a Makefile, the Modern one with GTK 3 and Meson. The backend is C++11 and the Windows UI is C++ (ADR 0009); two other build systems (Makefile, Meson) and a second language for the Linux UIs only made the tree harder to build and to read. Linux binaries are built on the machine that runs them (ADR 0007), so each UI has to configure and build with what that machine has.

## Decision

- **Legacy UI** (`ui/legacy/linux-xlib`): C++11 on Xlib, no toolkit, with RAII classes for the Xlib and Xft handles. It links libX11, and libXft with fontconfig when CMake finds them (the core X11 font is the fallback, `xlib-core-font-glyphs`, ADR 0007).
- **Modern UI** (`ui/modern/linux-gtk3`): C++ with gtkmm on the system packages, plain executable, no Flatpak, floor GTK 3.10 (ADR 0005 and ADR 0006).
- **One build system for the whole repository: CMake**, with the backend as the static library `libloomora` that both UIs link. Meson and Makefiles are not used. Both UIs are targets of the top-level `CMakeLists.txt` (the Xlib one is added on every Linux build and skipped with a message when libX11 is missing, unless `LOOMORA_XLIB_REQUIRED=ON`; the gtkmm one only with `LOOMORA_BUILD_LINUX_MODERN=ON`); `tools/build.py linux legacy|modern` drives them and records the deliverable.
- The i686 Xlib build is native too, through `cmake/toolchains/linux-gcc-i686.cmake` (`gcc -m32`).
- Both UIs reach the backend only through the C API (ADR 0008), whatever their language standard (ADR 0011).

## Rejected alternatives

- **Keep the C UIs**: a second language and two more build systems for two small programs.
- **Qt, or GTK without gtkmm in C++**: Qt is not a system toolkit of every distribution and has no GTK 3.10 floor; plain GTK from C++ gives up RAII and the typed signals for nothing.
- **A Makefile or Meson kept for the Linux UIs**: the owner wants one tool, and CMake already builds the backend and the Windows toolchains.

## Consequences

- The Meson options of the first GTK UI became CMake options: `LOOMORA_GTK_API` (3 or 4) and `LOOMORA_GTK_FLOOR` (ADR 0005, ADR 0006).
- The target machine needs CMake, a C++11 compiler and the development files (libX11; libXft and fontconfig for Xft; gtkmm for the Modern UI): `sudo apt install libgtkmm-3.0-dev` on Debian and Ubuntu.
- No Linux UI is verified on a distribution other than Ubuntu 24.04 (WSL2); a real old distribution waits for T092.
