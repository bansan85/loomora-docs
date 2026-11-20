# ADR 0021: Project file container, entry layout and identifiers

- **Status**: accepted (2026-11-17; the Windows 2000 half of spike S-F3 is still to run, see Results of the spikes)
- **Date**: 2026-11-17
- **Spike / requirement**: S-F2, S-F3 / FR-001 to FR-004, FR-014b, FR-020, FR-026, FR-028 (spec 002)

## Context

Spec 002 defines the `.loomora` project file: frames, nail layouts, images, spools and results in one file, readable without Loomora, never rewritten as a whole for a small edit (FR-002), byte-identical when the same project is written again with the same options on one platform, and the same decompressed content and file structure on any platform, architecture, byte order and date (FR-004; the compressed bytes may differ between platforms, see ADR 0022), and safe to open when the file is hostile (FR-026). The floors are old (Windows 2000, 32-bit devices with 1 to 2 GB of memory). The detail of the format is the contract `contracts/project-format.md` of spec 002; this record keeps the reasons.

## Decision

- **Container**: a ZIP archive (extension `.loomora`, MIME type `application/vnd.loomora+zip`), written and read by project code (about 800 lines, `core/src/format/zip/`). Methods 0 (stored) and 8 (deflate) only; no encryption. ZIP64 is read whenever present and written only when a size, an offset or the entry count needs it. The central directory is authoritative and each local header is checked against it.
- **Deterministic writer**: every header field has a fixed value (version made by and needed, flags, DOS date 1980-01-01 00:00:00, no extra field except ZIP64, no comment, fixed external attributes) and the entry order is fixed by the contract. `mimetype` is first, stored, at offset 0. Deflate uses fixed parameters (level 6, window 15, memLevel 8, default strategy, see ADR 0022). No data descriptor is written (the sink is seekable, sizes are patched).
- **Strict reader** (zip-slip, bombs): relative names with `/`, no `.`, `..` or empty component, no drive letter, no NUL, unique after case folding, at most 512 bytes; symbolic links rejected; entries must not overlap and must lie before the central directory; an unknown method or the encryption flag rejects the file; an expansion ratio above 250 to 1 after 1 MiB is rejected; everything is inflated through a fixed buffer. Nothing is ever extracted to disk.
- **Entry layout**: one folder per item, named by its decimal identifier, grouped by kind: `manifest.json`, `spools.json`, `frames/<fid>/{frame.json, mesh.bin, layouts/<lid>/{layout.json, nails.json}}`, `images/<iid>/...`, `results/<rid>/{result.json, sequence.bin, thumb.png, settings.json}`. An edit touches only the folder of the item and the manifest. Entries unknown to a version are listed, kept and copied on rewrite (FR-020).
- **Identifiers**: unsigned 32-bit integers allocated per kind from counters stored in the manifest (`next_id`), never decremented, so a removed identifier is never reused (FR-014b). The display order is a list per kind in the manifest.
- **Save**: a temporary file in the same directory, flush, then atomic replace: POSIX `rename`; on Windows `MoveFileExW(MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH)` first and, when it fails with `ERROR_ACCESS_DENIED` or `ERROR_SHARING_VIOLATION` because a reader holds the target open (spike S-F3: it does on Windows 11), the rename with POSIX semantics (`SetFileInformationByHandle` with `FileRenameInfoEx`, looked up at run time, Windows 10 1809 and later), then a rename-aside sequence (rename the old file away, move the new one in, delete the old one: every Windows, with a short window in which the target has no file, closed again by renaming the old file back when the second step fails). `ReplaceFile` is never used. The three steps are the fallback `win-replace-open-file`. On failure the temporary file is removed and the previous file is untouched (FR-028). Entries of untouched items are copied as compressed bytes from the old archive.
- **Read** opens with sharing that lets another process replace the file (Win32 `FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE`) and never with write access. The sharing flag is necessary for the rename-aside and POSIX-rename steps above but is not enough for `MoveFileExW` (S-F3).
- **64-bit offsets**: Win32 `SetFilePointer` with the high word (not `SetFilePointerEx`), POSIX `pread` / `pwrite` with `_FILE_OFFSET_BITS=64`, explicit `pread64` / `pwrite64` on Android API 9 to 20 (the `pread64` and `pwrite64` system calls below API 12, `fstat()` for the size below API 21; the code makes no `lseek` call). Each is a compile-time switch in `core/src/internal/config.hpp`, taken only by the platform that needs it and declared in `validation/fallbacks.json` (`win2000-setfilepointer`, `android-api9-lseek64`).

Two points were decided in the draft and gated by spikes; their results are below.

## Results of the spikes

**S-F2, byte identity across architectures (confirmed on the targets that could run, with one fallback).** `loomora_refgen` wrote the six reference files (`basic`, `empty`, `escape40`, `multi`, `multi-light`, `one`) on Linux x86_64 (g++ 15.2), Linux i686 (`linux-gcc-i686.cmake`, `-m32`, a 32-bit process on a 64-bit kernel), Windows x86_64 (MinGW-w64, `loomora_refgen.exe` only), the Windows 2000 MinGW-w64 build (run as 32-bit processes on Windows 11, **not on a Windows 2000 system**) and Windows 11 x64 (clang 22.1.3): the SHA-256 of the six files are identical everywhere, `tools/check_loomora.py --compare` reports `same`.

| File | SHA-256 |
|---|---|
| basic | `fcdf97f0257076ba1c3ecae1867e57ebdc397440286a278f2cbe34eff8ce5f96` |
| empty | `8ca4dc6a2139539feb87f92b11cc0d5685fdff051fa044b692a7be5d984d7913` |
| escape40 | `b17920aabf9cc165a5c3b9290ba15f22e99903592426a8e55c96dc67ae570e2a` |
| multi | `8bcd3ae91876dcf6048c3a5dc689754a8eab90367faed373942b819a7c3f8607` |
| multi-light | `b32c8a390657d4f56ec9bd448e6c291e9781dfbb3b34b8164d049af67a7c4a8d` |
| one | `fc3fb5ebd929689bd7ee32da020b279ca81b3412c6c947eda8c9e7e3ddb846c5` |

The i686 doubles differed at first: 32-bit x86 GCC computes on the x87 unit with 80-bit intermediates, so the **test fixtures** (`cos`, `sin`, `round3`, `0.1 + 0.2`) gave other nails and `multi` (b1f56507...) and `multi-light` (10f8dfc2...) changed; the other four files were identical and `-ffloat-store` alone was not enough. The fix is `-msse2 -mfpmath=sse` on the **test programs only** (`LOOMORA_TEST_FP_FLAGS`, set by the two i686 toolchain files, applied by `loomora_test_fp()`): the files are then byte-identical, with `libloomora` itself still compiled for x87, so **the library needs no switch** and the writer is unaffected. Declared as the fallback `x87-double-fixtures` (`linux-i686`, `windows-2000-x86`); the test programs then need an SSE2 CPU. Not run (`not_tested`): a Windows 2000 system, a big-endian system (QEMU user mode PowerPC: qemu is not installed), the ARM ABIs of Android and the Web builds in a browser (see the next paragraph for what Android and Web did run). The self-test of the Windows x86_64 MinGW build is `blocked_by_antivirus` (its `loomora_refgen.exe` ran).

**S-F2 on Android and Web (run 2026-11-17, recorded 2026-11-18; no new fallback).** After NDK r15c, three Android emulators and Emscripten (7.0.0 and 1.39.20 fastcomp) were installed, `loomora_refgen` and `loomora_format_selftest` ran again and the six files have the same SHA-256 as above on every one:

| Target | Runs on | Six hashes | Not run |
|---|---|---|---|
| Android x86, NDK r15c, built for API 9 | Android 4.1.2 (API 16) x86 emulator | identical | files above 2 GiB, API 9 to 15 |
| Android x86 (API 16 sysroot, PIE) | Android 7.0 (API 24), 32-bit process | identical | |
| Android x86_64 (API 21 sysroot, PIE) | Android 7.0 (API 24) and, as a different system with no platform record, Android 15 (API 35) | identical | |
| Android armeabi, armeabi-v7a, arm64-v8a | not run (no ARM system) | built and ELF-checked only | everything |
| Web `web-wasm` (Emscripten 7.0.0), `web-js` (wasm2js), `web-js-fastcomp` (1.39.20, node 14) | node, not in a browser | identical | any browser run of the module |

No Web-specific or Android-specific switch changes a byte: Clang for x86 uses SSE2, so the `x87-double-fixtures` fallback is not needed there. What the Android runs forced is in the file layer, not in the writer (see the fallback `android-api9-lseek64` and the Evidence below).

**S-F3, file operations (partly run, spike still `open`).** Run on Windows 11 build 26300, NTFS only; the Windows 2000 part and FAT32 were not run.

- (a) `MoveFileExW(MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH)` replaces a file nobody holds, but **fails with `ERROR_ACCESS_DENIED` over a file a reader holds open, even when the reader opened it with `FILE_SHARE_DELETE`**: the assumption of research R10 is refuted on Windows 11. Rename-aside, `ReplaceFileW` and the POSIX-semantics rename all replace it and leave the reader on the complete old file. The Save bullet above is the consequence (fallback `win-replace-open-file`).
- (b) `SetFilePointer` with the high word reaches 5 GiB on a sparse NTFS file (offsets `0x140000000`, `0x1FFFFFFFF`, `0x180000000`); the trap is the low word `0xFFFFFFFF` (equal to `INVALID_SET_FILE_POINTER`), handled by clearing and then reading the last error. The file-layer tests also pass with `LOOMORA_FMT_WIN2000_SEEK=1`.
- (c) `MultiByteToWideChar(CP_UTF8, 0)` gives the right surrogate pairs and the created file carries the exact name; a malformed byte with flags 0 becomes U+FFFD without an error, so the product validates UTF-8 itself first.
- To run on the Windows 2000 VM (NTFS, and FAT32 if available): (a) whether `MoveFileExW` behaves differently, (b) the 5 GiB seek and the expected failure at 4 GiB on FAT32, (c) `CP_UTF8` with flags 0. Until then the Windows 2000 behaviour of every item is unknown and `win2000-setfilepointer` is declared but not measured on that system. `android-api9-lseek64` was run on Android on one x86 emulator at API 16 (the `pread64` system call and `fstat()` paths, with the build made for API 9) and not above 2 GiB; its ARM branch (the padding argument of the system call) never ran.

**Other results tied to this record (2026-11-17).** SC-004: 12.6 million fuzz inputs against the reader (clang++ 23.1.6, ASan and UBSan, leak detection), no crash, leak, undefined behaviour, out-of-bounds access or input over 5 s (slowest 0.26 s). SC-005: 5,000 nails and 100,000 steps validate and open in about 10 ms (required: 1 s); 500,000 nails and 10,000,000 steps (41 MB) open with their warnings in about 1 s, peak 426 to 470 MB. The review (T101) of `core/src/format` found 8 items, 2 fixed (a FIFO opened by the POSIX layer, a data race on the temporary-file counter), 6 deferred for the owner (`validation/bug-reports/format-review-2026-11-17.md`). The Windows 2000 import tables of the format programs pass their allowlist after the condition variable shim (fallback `win2000-condvar-shim`); this is a static check, not a run on that system.

## Rejected alternatives

- **minizip or libzip**: timestamps, extra fields, version-dependent choices, whole entries allocated, API tied to files; none gives control of every byte (FR-004) or the reader strictness above.
- **miniz**: single file and handy, but it bundles its own deflate, whose output differs from zlib's and is less documented across versions.
- **One big `project.json`**: any edit rewrites everything, and listing the results needs a full parse.
- **UUID item identifiers**: longer paths and no gain; the project already has a UUID.
- **A SQLite file**: rejected for the format by decision 5 of the requirements, and not available on the floors.
- **In-place append to the ZIP**: unsafe on a crash, contradicts FR-028.
- **Decode and re-encode every entry on save**: slow and can change bytes; copy-through keeps untouched entries identical by construction.

## Consequences

- A project of any size is validated by streaming (SHA-256 and CRC-32 per entry); memory is bounded by one range or one small item.
- The reference hashes of the writer depend on the pinned zlib version, v1.3.1 as a git submodule (ADR 0022); a zlib upgrade changes them in a visible commit.
- The requirements document, section 9.1, is rewritten from the contract (task of spec 002, main repository).
- Still to verify: S-F3 on the Windows 2000 VM (NTFS and FAT32 if available), S-F2 on a big-endian system and on a real Windows 2000, and the Android `android-api9-lseek64` fallback above 2 GiB, on an ARM system and on Android 2.3 to 4.0; until then the Windows 2000 and Android fallbacks above are declared, not measured on those systems. The spike S-F3 stays `open` in `validation/spikes.json`.
- Saving over a file another process holds open works through the three-step replace, with a short window without a target file on systems that have neither `MoveFileExW` success nor the POSIX rename (the rename-aside step; a leftover `.old-` file is possible if the last delete fails, a deferred finding of the T101 review).
- The `x87-double-fixtures` fallback concerns the test programs of 32-bit x86 targets only; no product code differs.

## Evidence

All under `validation/evidence/adr-0021/`, stored once (ADR 0014; no image):

- `windows-11-x64/`: the six reference files (written by the MinGW-w64 x86_64 `loomora_refgen.exe` on Windows 11), `hashes.json` (the hashes of the Windows MinGW and clang runs), `s-f1-fmt-deps-output.txt` and `s-f3-fileio-probe-output.txt` (raw output, copied from `validation/results/spec-002/spike_results_sf.md`).
- `linux-i686/`: `hashes.json` and the two files written without the SSE2 switch (`x87-float-store-multi.loomora`, `x87-float-store-multi-light.loomora`), the proof that the fallback is needed.
- `linux-ubuntu-26.04-x86_64/` and `windows-2000-x86/`: `hashes.json` only (identical to the reference, not stored again). The first one is the Ubuntu 26.04 distribution that ran the Linux work: no platform record exists for it. The second records a Windows 2000 build run on Windows 11, not on the VM.
- `android-4.1.2-x86/`, `android-7.0-x86_64/`, `web-chrome-157-windows-11-x64/` and `web-firefox-10-windows-2000-x86/`: `hashes.json` only (identical to the reference, not stored again; no APK, no binary, no image). `android-7.0-x86_64/` also holds, as a separate run with its caveat, the Android 15 (API 35) emulator, for which no platform record exists. The Web files were written under node, not in the browsers the platform records name.
- The verdict notes of `windows-11-x64`, `windows-2000-x86`, `linux-i686`, `linux-ubuntu-24.04-x86_64` (Legacy), `android-4.1.2-x86` (Legacy), `android-7.0-x86_64` (Modern), `web-chrome-157-windows-11-x64` (Modern) and `web-firefox-10-windows-2000-x86` (Legacy) carry the results; the machine that ran spec 002 (Intel Core i7-11800H) is not the one described by `validation/platforms/windows-11-x64.json`.

## Sources

Documents read for the draft (2026-11-17, no external source was fetched):

- `specs/002-loomora-file-format/research.md`: R2, R3, R5, R7, R10, R11, R12, R15
- `specs/002-loomora-file-format/plan.md`: Technical Context, Constitution Check, Project Structure
- `specs/002-loomora-file-format/contracts/project-format.md`: sections 1 to 3 (container, entry order, manifest; the rest not read for the draft)
- `specs/002-loomora-file-format/spec.md`: FR-001 to FR-004 (the other requirements are cited from `research.md`)

Documents read for the acceptance (2026-11-17):

- `validation/results/spec-002/cross_arch_results.json` and `cross_arch_results.md`: per target built / ran / hashes, fallbacks needed, import checks, `blocked_by_antivirus`, `not_tested`
- `validation/results/spec-002/spike_results_sf.md`: S-F1 and S-F3 results, including the raw output of the Windows 11 run (the probe source `tools/guest/fileio_probe.c` itself was not re-read for this record)
- `validation/results/spec-002/android_results.md` and `android_results.json`, `validation/results/spec-002/web_results.md` and `web_results.json` (added 2026-11-18, runs made on the branches `feat/fmt-android` and `feat/fmt-web` of the main repository): the Android and Web builds, runs, hashes, code fixes and `not_tested` lines. The reference files of these runs and `file_posix.cpp` were not re-hashed or re-read for this record: the hashes are those the result files state
- `validation/results/spec-002/fuzz_run_sc004.md` (SC-004 fuzz run) and `validation/results/spec-002/perf_sc005.md` (SC-005 numbers)
- `validation/bug-reports/format-review-2026-11-17.md` (T101 review)
- the raw reference files of each target (`$HOME/loomora-build/xtarget-refs/<target>/` in WSL), re-hashed on 2026-11-17 and compared with `python tools/check_loomora.py --compare`
- the logs of the last `tools/verify.py quick` runs (`ui-project-*` logs of the Windows and Linux hosts)
- **Not read** (listed by `research.md`, no external document was fetched or read for this record; the results above rest on measurements, not on these texts): PKWARE APPNOTE 6.3.x, Microsoft `MoveFileEx`, `CreateFile` and `SetFilePointer`, Android NDK notes on `off_t`, `pread64` and `fstat64` (the API levels of those functions come from the compile errors of NDK r15c reported in `android_results.md`, not from the notes). A claim in this record that comes from one of them is the claim of `research.md`, not checked here.
