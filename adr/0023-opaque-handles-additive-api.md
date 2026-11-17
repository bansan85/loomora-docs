# ADR 0023: Opaque handles in the C API; additive API, ABI version unchanged

- **Status**: accepted (2026-11-17)
- **Date**: 2026-11-17
- **Spike / requirement**: none / FR-017, FR-019 (spec 001), FR-026 (spec 002)

## Context

ADR 0001 sets four rules for the C API; rule 3 says memory crosses the boundary only in caller-supplied buffers, with no "free this" function. Opening a `.loomora` project is stateful: a project of any size cannot be copied through a buffer, range reads need the open archive, and an editor keeps staged changes. A stateless API that opens and validates the archive at every call would hash the whole file each time. The language bridges of the UIs (Java through JNI, C# `LibraryImport`, Objective-C, JavaScript on Emscripten) can all hold an opaque pointer and pass byte buffers (research R13).

The project owner decided on 2026-11-17 that `LOOMORA_ABI_VERSION` stays 1 for this feature.

## Decision

- **Rule 3 is extended, not dropped**: the library creates and destroys **opaque handles** (`loomora_project`, `loomora_editor`, `loomora_report`) through explicit open, create and close functions. A handle exposes no memory; every byte of data still crosses in a buffer owned by the caller. The other three rules stay as they are: no C++ type in the header, every exported function catches all exceptions and returns a negative `LOOMORA_ERR_*` code, the runtime is linked as ADR 0001 and ADR 0007 say.
- **Data conventions**: size query (`cap = 0` returns the length needed, then a second call), range reads into caller arrays for bulk data, metadata items as UTF-8 JSON documents whose schema is the contract, a validation report as a handle with accessors rather than a text blob, a progress and cancel callback. A handle is used from one thread at a time; read-only project handles may be shared for reads.
- **Additive API**: the new functions live in `core/include/loomora_project.h`, included by `loomora.h`. No existing function, constant or structure changes, so `LOOMORA_ABI_VERSION` is **not edited and stays 1**. A caller that needs the project API checks `loomora_format_version`, and links only against a library that has it. The export lists (`loomora.exp`, `loomora.map`, the JNI and Emscripten lists) are extended with the new functions; `tools/check_std.py` keeps checking that only `loomora_*` C symbols are exported.
- If a later change breaks an existing function, the ABI version is raised then, by its own decision.

## Verification (2026-11-17)

No spike was needed. What was run on the opaque-handle API:

- **SC-010, the project modes of the UIs** (`tools/verify.py` step `ui project`; `tools/run_vectors.py --validate-project` and `--roundtrip-project` against `loomora validate --quiet`, byte for byte and by exit status): 37 of 37 runs on each of the Windows Legacy UI x64 (the default host build, clang), the Linux Legacy UI (Xlib) and the Linux Modern UI (gtkmm on GTK 3), 6 reference files and 9 valid corpus files validated and round-tripped, one invalid file, one missing path, 3 usage rows. Not run: the Windows 2000 VM, macOS and iOS, the WinUI 3 UI (not built by the quick verification), the i686 Linux UI. The bridges of the Apple and WinUI 3 UIs (`LibraryImport`, Apple) were written but not run, so the claim that every bridge language can hold an opaque pointer (research R13) is verified for the three UIs above and, from the later runs below, for the JNI and Emscripten bridges.
- **SC-010 on Android and Web (run 2026-11-17, recorded 2026-11-18, `core/tests/android_results.md` and `web_results.md` of the main repository)**: against `loomora validate --quiet`, the Android Legacy UI (Java, JNI) matched in 37 of 37 runs on the API 16 and API 24 emulators and the Modern UI (Kotlin, Compose) in 37 of 37 on API 24 and, as a system with no platform record, API 35; the whole corpus (113 files, validate only) matched in 116 of 116 with the Legacy UI on API 16 and the Modern UI on API 24. On Web, the Legacy page matched in 242 of 242 runs under node with the WebAssembly module and 242 of 242 with the fastcomp `loomora.js` alone (DOM stub, no browser), the Modern page in 242 of 242 under node and 51 of 51 in real Chrome 157.0.8103.39 (headless, 6 references and 18 corpus files). The T091 and T093 code needed no fix. Not run: the Legacy page in Firefox 10 on Windows 2000, any ARM Android system, Android 2.3 to 4.0, a browser other than Chrome, and macOS, iOS and the Windows 2000 VM as above.
- **SC-004**: 12.6 million fuzz inputs through the reader entry point of `core/fuzz/fuzz_project.cpp` (validate, open and read every item of an archive in memory) under ASan, UBSan and leak detection: no crash, leak or undefined behaviour; whether that entry point goes through the C handle functions or the C++ layer behind them was not checked for this record.
- **T101 review** of `core/src/c_api_project*.cpp` and `core/src/format/`: the exception boundary of every C API function was read and found correct; the thread-safety rule of this record (a handle used from one thread at a time, read-only handles shareable) is what the finding about the shared counter of the temporary file name (fixed) rests on; the review also lists six deferred items (`validation/bug-reports/format-review-2026-11-17.md`).
- `LOOMORA_ABI_VERSION` was not edited and stays 1.

## Rejected alternatives

- **A stateless API (re-open and re-validate on every call)**: a full hash of the archive for each read.
- **One function per field**: hundreds of functions.
- **Library-allocated JSON that the caller must free**: breaks rule 3 more widely than a handle, and adds an ownership rule per call.
- **A callback-based visitor**: awkward from Java and JavaScript.
- **Raising `LOOMORA_ABI_VERSION` to 2**: not needed while nothing existing changes (owner decision of 2026-11-17).

## Consequences

- Each handle has one owner and one destroy function; the leak risk moves to the bridge code, which the probe modes of spec 002 exercise.
- A UI built against the new header and run against an older library fails at link or load time, not through a version check; hence the `loomora_format_version` rule above.
- ADR 0001 is not amended in its text; this record is the extension of its rule 3.
- This is the deviation "opaque handles" of the Complexity Tracking of spec 002.

## Sources

Documents read for the draft (2026-11-17, no external source was fetched):

- `adr/0001-backend-cpp11-c-api.md` (Decision, Consequences)
- `specs/002-loomora-file-format/contracts/c-api-project.md`: introduction, conventions, error codes, library and start of the reading functions (first 60 lines; the rest was not read for the draft and was not re-read for the acceptance)
- `specs/002-loomora-file-format/research.md`: R13
- `specs/002-loomora-file-format/plan.md`: Constitution Check, Complexity Tracking

Documents read for the acceptance (2026-11-17):

- `validation/bug-reports/format-review-2026-11-17.md` (T101 review)
- `core/tests/fuzz_run_sc004.md` of the main repository (SC-004)
- the logs of the last `tools/verify.py quick` runs (`ui-project-*` of the Windows and Linux hosts) and the project-mode section of `tools/run_vectors.py`
- No external document is cited by this record.
