# ADR 0002: Keep KERNEL32!GetThreadId out of the Windows 2000 executable with a link-time shim

- **Status**: accepted
- **Date**: 2026-11-12
- **Spike / requirement**: S1 (T066) / FR-017, research R6

## Context

The i686 Legacy executable (`ui/legacy/windows-win32`, built by `tools/build.py windows legacy`) must load on Windows 2000 SP4 (research R6). Reading its import table function by function showed `KERNEL32!GetThreadId`, which exists only since Windows Vista / Server 2003 (not on Windows 2000, not on XP x86): the loader of those systems refuses an executable whose import table names a function the DLL does not export. The import check of `tools/build.py` compared only DLL names and missed it.

A linker map (`-Wl,-Map`, `-Wl,--trace-symbol=_GetThreadId@4`) of the MinGW-w64 GCC 13 build (win32 thread model, msvcrt) shows the chain:

- `libstdc++.a(eh_alloc.o)`, the emergency pool behind every C++ `throw` (the backend uses exceptions, ADR 0001), references `__gthr_win32_mutex_init_function`;
- that makes the linker take the whole `libgcc.a(gthr-win32.o)`, one object holding all the `__gthr_win32_*` helpers;
- inside it, `__gthr_win32_equal` (the `__gthread_equal` of the win32 thread model) calls `GetThreadId` twice, so the object refers to `__imp__GetThreadId@4` and the linker takes the import from `libkernel32.a`.

Our code uses no `std::thread` or mutex and never calls `__gthread_equal`: the import is dead code at run time, but it is in the import table, and the table is what the loader checks. `libgcc_eh.a(gthr-win32-thread.o)` is not linked. The same chain exists for any 32-bit executable that links libstdc++ statically (the backend's `loomora_selftest` too).

## Decision

Satisfy the reference inside the executable. `ui/legacy/windows-win32/getthreadid_shim.cpp` defines the two symbols the linker would otherwise take from `libkernel32.a`: `_GetThreadId@4` (a function) and `__imp__GetThreadId@4` (the import-table-style pointer that `gthr-win32.o` calls through). The linker finds both already defined and does not pull the `libkernel32.a` member. The function behaves like the real one if it is ever called: it looks `GetThreadId` up in kernel32 when called (found on Vista and later) and otherwise reads the thread id with `NtQueryInformationThread(ThreadBasicInformation)` from ntdll (what `GetThreadId` does; available since NT 3.1), returning 0 on failure as documented. It uses only `GetModuleHandleA` and `GetProcAddress`, which the executable imports already, so the import table gains nothing and no DLL is added. Static libstdc++/libgcc stay as they are.

`ui/legacy/windows-win32/CMakeLists.txt` adds the shim only for 32-bit GNU builds. The x64 executable is unchanged: its floor is XP x64 / Server 2003 x64 (built on Server 2003 SP1, believed to export `GetThreadId`, entry marked `verify` in the allowlist).

To make sure this cannot recur, `tools/build.py` now runs `tools/check_imports.py` on both executables: every imported function must be on `tools/win2000_imports.json` (per DLL, with the minimum OS, who uses it and a confidence of `documented` or `verify`; the x64 list adds the x64-only imports and `GetThreadId`). An import that is not listed fails the build with the exact `dll!function`. `python tools/check_imports.py --self-test` shows a Vista-only import being rejected without any executable.

Checked on the Windows 11 host with the shim built into a test program: `GetThreadId` through the shim equals the real one for a created thread, the current-thread pseudo handle and (returns 0) an invalid handle, both with the native path and with the `NtQueryInformationThread` fallback forced.

## Rejected alternatives

- **A different libstdc++/libgcc configuration** (a build without the emergency pool, `--disable-threads`, the posix thread model): not available without installing or building another toolchain; the posix model would add libwinpthread and more imports instead. Rewriting the object's other symbols to keep `gthr-win32.o` out of the link would mean defining every `__gthr_win32_*` function ourselves: hackier than two symbols.
- **Research R6 fallbacks** (ship `loomora.dll`; an older MinGW-w64/GCC; give Windows 2000 a "KO" verdict): not needed while the shim works; they stay available if spike S1 refutes something else. Changing the compiler for one dead import is a heavier change than this shim, and an older version could bring other problems.
- **Binary patching or editing the import table after linking** (rename the import, strip it): fragile, breaks on every linker change, and hides the cause.
- **Leaving it and relying on Windows XP SP2+ or Vista**: contradicts the Windows 2000 floor.

## Consequences

- The i686 executable no longer imports `GetThreadId`; its function-level import list (88 functions from gdi32, kernel32, msvcrt, user32) is entirely on the Windows 2000 SP4 allowlist. 12 entries are marked `verify` (see below).
- Any new 32-bit MinGW executable for Windows 2000 (the `loomora_selftest` of T066, for one) must link the same shim, and `tools/build.py` must run `check_imports.py` on it.
- A new GCC / MinGW-w64 version can bring other post-2000 imports: the check fails the build on them and the allowlist entry must be justified before it is added.
- Still to verify (not done by this build): the run on a real Windows 2000 SP4 (spike S1 / T066); the export table of that system's `kernel32.dll` for `TryEnterCriticalSection` (documented minimum client is Windows XP, exported since NT 4.0 SP3) and of its `msvcrt.dll` 6.10 for the start-up internals (`__getmainargs`, `__initenv`, `__mb_cur_max`, `__p__acmdln`, `__p__commode`, `__p__fmode`, `__set_app_type`, `__setusermatherr`, `_amsg_exit`, `_cexit`, `_initterm`); on the x64 side `GetThreadId` and the x64 `msvcrt.dll` accessors on XP x64 / Server 2003 x64.
