# ADR 0012: Windows binaries are cross-compiled with MinGW-w64 GCC, from WSL

- **Status**: accepted
- **Date**: 2026-11-14 (the choice of MinGW-w64 was made on 2026-11-12 and confirmed by spike S1, T066; the WSL rule was written on 2026-11-14)
- **Spike / requirement**: S1 / FR-017, FR-022, research R6

## Context

Windows 2000 is the floor of the Windows Legacy UI. Visual Studio no longer targets it, and the last MSVC that did (2005 SP1) has no C++11 (ADR 0009). The backend is C++11 and the build machine of the project is a Windows host with WSL2 (Ubuntu 24.04).

## Decision

- **One toolchain builds the backend and the Windows Legacy UI: MinGW-w64 GCC**, `-std=c++11`, the `msvcrt.dll` runtime (not UCRT, which Windows 2000 does not have), the win32 thread model preferred, `-static` with libstdc++ and libgcc linked statically (ADR 0001 rule 4). Two toolchain files: `cmake/toolchains/mingw-w64-win2000-i686.cmake` (`_WIN32_WINNT=0x0500`, Windows 2000 to 11) and `cmake/toolchains/mingw-w64-win-x86_64.cmake` (`_WIN32_WINNT=0x0502`, XP x64 to 11).
- **The compilers run in WSL** (Ubuntu: `g++-mingw-w64-i686`, `g++-mingw-w64-x86-64`, the `-win32` variants selected with `update-alternatives`) and the build goes under `$HOME/loomora-build`, never in `/tmp` or in the Windows tree. `tools/build.py windows legacy` is started in WSL: `wsl -d Ubuntu-24.04 -- python3 tools/build.py windows legacy`.
- Problems only Windows 2000 has are solved by switches of the Windows 2000 toolchain, not for everyone (ADR 0004): `LOOMORA_TARGET_WIN2000` (ADR 0002) and `LOOMORA_AVOID_IOSTREAM`.
- Every function imported by an executable is checked against `tools/win2000_imports.json` by `tools/check_imports.py`, which `tools/build.py` runs; DLL names alone are not enough (ADR 0002).
- The **Modern** Windows UI (WinUI 3) is outside this decision: it is built by current MSVC with `/std:c++14` (the source stays C++11).
- The host build of `core/` and the Win32 UI with clang and Ninja (`CMakePresets.json`, `verify.py quick` on Windows) is a check of the sources, not a deliverable.

## Rejected alternatives

- **Visual Studio 2005 SP1**: reaches Windows 2000 but has no C++11 and is hard to obtain.
- **A recent Visual Studio or clang-cl with the MSVC runtime**: their binaries need APIs and a C runtime that Windows 2000 does not have.
- **Compile on the Windows host with WinLibs**: `tools/build.py` accepts it (the MinGW-w64 compilers found on `PATH`), but it is not the documented route; WSL keeps the toolchain reproducible with `apt`.
- **Ship the backend as a `loomora.dll`**: kept as the first fallback if a spike refutes the static library, not needed so far (research R6).

## Consequences

- The Windows executables were checked on Windows 2000 build 2195 and Windows 11 only (verdicts of `windows-2000-x86` and `windows-11-x64`); Windows XP to 8.1 were not tried (spike S9).
- A different MinGW-w64 version or thread model may reintroduce Vista-only imports: the import check catches it.
- No macOS, iOS, Android or Web toolchain is covered here (research R3 to R5, R8).
