# ADR 0023: Opaque handles in the C API; additive API, ABI version unchanged

- **Status**: proposed
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

Documents read to write this draft (no external source was fetched):

- `adr/0001-backend-cpp11-c-api.md` (Decision, Consequences)
- `specs/002-loomora-file-format/contracts/c-api-project.md`: introduction, conventions, error codes, library and start of the reading functions (first 60 lines; the rest not read for this draft)
- `specs/002-loomora-file-format/research.md`: R13
- `specs/002-loomora-file-format/plan.md`: Constitution Check, Complexity Tracking
