# Spike results S-F1 and S-F3 (spec 002), host phase

Raw material for whoever writes `spikes.json` (S-F1, S-F3) and the ADR 0022 draft. Only the
Windows 11 host and WSL Ubuntu were available; the old toolchains and the Windows 2000 VM were not.

## S-F1: zlib and RapidJSON (T009, `core/tests/spike_fmt_deps.cpp`)

The program deflates (raw, levels 6 and 1) and inflates a 1 MiB buffer through the vendored zlib 1.3.1,
writes and parses a JSON document with 2000 doubles through RapidJSON (`kParseFullPrecisionFlag |
kParseIterativeFlag`, every double compared bit for bit, the re-written text compared), and prints the
SHA-256 of the deflated streams and of the JSON text. Target `loomora_spike_fmt_deps` (EXCLUDE_FROM_ALL).
sha256 of the source: `89c5c9b4a8dbe4ca14591476a0c7f14d84ba13d18acf1548a47c10a1e32e5bdd`.

| Toolchain | Run | Result |
|---|---|---|
| Windows 11 x64, clang (Visual Studio 18 LLVM) + Ninja, CMake target | yes | `S-F1: ok` |
| WSL Ubuntu 26.04, g++ 15.2 `-std=c++11 -pedantic-errors -Wall -Wextra -O2` (zlib sources with gcc `-w`) | yes | `S-F1: ok` |
| MinGW-w64 `mingw-w64-win2000-i686.cmake`, GCC 13-win32 (Ubuntu 26.04 cross build in WSL), `.exe` run on Windows 11 x64 (not on a Windows 2000 VM) | yes | `S-F1: ok`, the three lines identical to the two above except `sizeof(void*)=4` |
| same toolchain on the Windows 2000 VM | not run | |
| `android-ndk-r15c.cmake` (emulator or device) | not run | |
| `emscripten-js-fastcomp.cmake` (Node) | not run | |

The two runs print the same three lines (outputs identical apart from `sizeof(long)` 4 on Windows and 8 on Linux), stored in
`validation/evidence/adr-0021/windows-11-x64/s-f1-fmt-deps-output.txt` (`zlib 1.3.1 level 6 ... sha256(deflated)=0f01cbcd...`, `level 1 ...
2d7f429d...`, `rapidjson: 2000 doubles ... sha256(json)=9dff6554...`).

A toolchain that prints these three lines produces the same deflate bytes and the same JSON text.
No fallback is needed on the two hosts. Not measured: the Android and Emscripten toolchains, nor the Windows 2000 VM.

MinGW-w64 win2000 (T059, cross-architecture phase): `loomora_spike_fmt_deps.exe` built from WSL with
`cmake/toolchains/mingw-w64-win2000-i686.cmake` (`_WIN32_WINNT=0x0500`, msvcrt, static libgcc and libstdc++, the
GetThreadId and condition variable shim linked in), run on Windows 11 x64 (32-bit process): the `zlib` level 6 and
level 1 lines and the `rapidjson` line are byte for byte the three lines above (same `sha256(deflated)` `0f01cbcd...`,
`2d7f429d...` and `sha256(json)` `9dff6554...`), so zlib and RapidJSON need no fallback with the MinGW-w64 toolchain.
Its import table passes `tools/check_imports.py --arch x86` (69 imports, 5 on `verify` entries of
`tools/win2000_imports.json`: `msvcrt.dll!__p___initenv`, `__p___mb_cur_max`, `__p__iob`, `_lock`, `_unlock`).

## S-F3: file operations (T010, `tools/guest/fileio_probe.c`)

Written in C89, kernel32 and the C runtime only; newer functions are found with `GetProcAddress`.
It compiles clean with the Visual Studio clang and (compile only) with `i686-w64-mingw32-gcc -std=c89
-D_WIN32_WINNT=0x0500`. sha256 of the source: `71a40f26f7150c1cd6d6806b32c1eadc8add468300c76c7385b84bba10820ccc`
(`tools/guest/.gitattributes` marks it `-text`).

Run: Windows 11 (RtlGetVersion 10.0 build 26300), NTFS. Not run: Windows 2000 (NTFS, FAT32).

(a) Replacing a file a reader holds open (reader opened with `GENERIC_READ`):

| Method | Reader share | Result |
|---|---|---|
| `MoveFileExW(REPLACE_EXISTING\|WRITE_THROUGH)`, no reader open | | replaced |
| `MoveFileExW(REPLACE_EXISTING\|WRITE_THROUGH)` | READ\|WRITE\|DELETE | **fails, ERROR_ACCESS_DENIED (5)**; target and reader untouched |
| same | none, or READ\|WRITE | fails, error 5 |
| rename old aside, move new in, delete old (all Windows) | READ\|WRITE\|DELETE | replaced; reader keeps reading the complete old file |
| `ReplaceFileW` (XP and later) | READ\|WRITE\|DELETE | replaced; reader keeps the old file |
| `SetFileInformationByHandle(FileRenameInfoEx, REPLACE_IF_EXISTS\|POSIX_SEMANTICS)` (Windows 10 1809 and later) | READ\|WRITE\|DELETE | replaced atomically; reader keeps the old file |

Finding: contrary to the assumption of research.md R10, `MoveFileExW` replace does **not** work over a
file that is open, even with `FILE_SHARE_DELETE`, on Windows 11 NTFS. The sharing flag is necessary
for the rename-aside and POSIX-rename methods but not sufficient for `MoveFileExW`. Whether Windows 2000
behaves differently is unknown until the VM run.

Decision implemented in `file_win32.cpp` (`WinSink::replace_target`): try `MoveFileExW` (atomic, the
common case, no reader open); when it fails with ACCESS_DENIED or SHARING_VIOLATION and the target is a
regular file that is not read-only, try the POSIX-semantics rename (found at run time, atomic), then the
rename-aside sequence (every system, with a short window in which the target has no file, closed again
by renaming the old file back when the second step fails). `ReplaceFile` is not used. Needed in
`fallbacks.json`: a run-time fallback "windows-replace-open-target" (platforms: every Windows where a
reader may be open; mechanism: GetProcAddress + rename-aside).

(b) 5 GiB offset with `SetFilePointer` and its high word on a sparse NTFS file: offsets `0x140000000`
(5 GiB), `0x1FFFFFFFF` (low word 0xFFFFFFFF, equal to INVALID_SET_FILE_POINTER) and `0x180000000` (low
word negative as a LONG) all write and read back correctly; `GetFileSize` high word right. The only
trap is the 0xFFFFFFFF low word, so the code calls `SetLastError(NO_ERROR)` before and checks
`GetLastError()` after (`seek64` in `file_win32.cpp`). `LOOMORA_FMT_WIN2000_SEEK=1` was also compiled and
the whole test file run on Windows 11 with it: the 5 GiB read test passes. Not run on FAT32 (4 GiB limit
expected: the write must fail with a disk-full or file-too-large error, which the probe reports).

(c) UTF-8 path with non-BMP characters (U+1F600, U+1D11E) and accents: `MultiByteToWideChar(CP_UTF8, 0)`
gives the right surrogate pairs (16 units for the test string), a file created through the converted path
carries exactly that name when read back from the directory. `MB_ERR_INVALID_CHARS` is accepted on
Windows 11 (Vista and later only); a malformed byte with flags 0 becomes U+FFFD with no error, so
`file_win32.cpp` validates the UTF-8 itself first (strict, no overlong form, no surrogate) and uses
flags 0. To check on Windows 2000: that CP_UTF8 with flags 0 gives surrogate pairs (not run).

Raw output of the Windows 11 run: `validation/evidence/adr-0021/windows-11-x64/s-f3-fileio-probe-output.txt` (not repeated here).

## Verification of the I/O layer (T016 to T018)

`core/tests/format/test_file_io.cpp`, 15 test cases, passes on Windows 11 (clang, default path, plus two
ad hoc builds with `LOOMORA_FMT_WIN2000_SEEK=1`, one of them with the POSIX-rename step disabled to
exercise the rename-aside step) and on Linux (g++ in WSL, ext4). `file_posix.cpp` also compiles with
`LOOMORA_FMT_LSEEK64=1`, `LOOMORA_FMT_NO_LARGE_FILES=1` and `-m32`; `file_win32.cpp` compiles with
MinGW-w64 i686 `-D_WIN32_WINNT=0x0500 -std=c++11 -pedantic-errors` and x86_64 (compile only, not linked
or run). New kernel32 imports for `tools/win2000_imports.json`, when the Legacy executable links the
library: CreateFileW, ReadFile, FlushFileBuffers, MoveFileExW, DeleteFileW, SetFilePointer,
GetFileInformationByHandle, GetFileType, GetFileAttributesW, GetFullPathNameW, MultiByteToWideChar, Sleep, InterlockedIncrement, QueryPerformanceCounter, GetTickCount (all
Windows 2000); SetFilePointerEx only without the switch; SetFileInformationByHandle only by GetProcAddress.
