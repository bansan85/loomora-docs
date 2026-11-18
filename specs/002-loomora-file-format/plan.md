# Implementation Plan: The `.loomora` File Format

**Branch**: `feat/loomora-format` | **Date**: 2026-11-17 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/002-loomora-file-format/spec.md`

## Summary

Add to the single C++11 backend (`core/`) a **format module** that writes, reads, edits and validates `.loomora` project files, plus a host command `loomora validate`. A project is a ZIP archive with one set of JSON and binary entries per **frame**, **nail layout**, **image** and **result**, one entry for the project's **spools**, and a manifest with a SHA-256 per entry ([contracts/project-format.md](contracts/project-format.md)). It holds no weaving progress.

Approach ([research.md](research.md)):

- **Streaming.** The reader validates every entry while streaming it (no entry is held whole in memory) and serves items on demand, by range for the bulk data (nails, steps, mesh). A 32-bit machine can therefore use a project far larger than its memory, and "no limit on elements" costs only time and a warning.
- **Own container code, pinned codec.** A strict ZIP reader and a deterministic ZIP writer (stored and deflate, ZIP64 when needed) written for the project; **zlib** (vendored) only for deflate and inflate. **RapidJSON** (vendored, header-only) for JSON, whose own number conversion makes the text identical on every platform. SHA-256 is written for the project.
- **Deterministic bytes.** Fixed entry order, fixed ZIP header values, no clock, fixed deflate parameters from the pinned zlib: the same project writes the same bytes on every architecture. Each target's self-test recomputes the SHA-256 of a reference project and compares it to an embedded value, so cross-architecture identity is checked on every platform without moving files.
- **Edit mode** stages changes and saves by writing a new archive next to the old one and replacing it atomically; entries of untouched items are copied unchanged (same checksums, SC-013). Changing a frame, layout, image or spool removes the results that depend on it. **Use mode** opens read-only and never writes.
- **Boundary.** A second C header, `loomora_project.h`, included by `loomora.h`, with opaque handles, caller-supplied buffers, range reads and a validation report ([contracts/c-api-project.md](contracts/c-api-project.md)). Every UI gets it through the existing glue; each UI only gains a probe-style `--validate-project` mode (no logic).

## Technical Context

**Language/Version**: backend **C++11**, `CXX_STANDARD 11` as a target property, checked by `tools/check_std.py` (GCC and Clang `-std=c++11 -pedantic-errors -Werror`); vendored C (zlib) compiled as C89/C99 by the same toolchains; no language standard above C++11 (FR-031). Exceptions are used inside the module and never cross the C API.

**Primary Dependencies**: zlib 1.3.x (git submodule, v1.3.1, built subset: `adler32 crc32 deflate inflate inffast inftrees trees zutil`, zlib licence), RapidJSON (vendored `include/rapidjson`, MIT), Catch2 v2.13.10 (tests only, existing submodule), Python 3.8+ standard library for host tools (ADR 0015), `jsonschema` only for the schema check of the independent checker (already allowed).

**Storage**: the project file itself (a ZIP archive); no database. The weaving progress is application data and is outside this feature.

**Testing**: Catch2 tests on the host; `loomora_format_selftest` (plain C++11, no framework) cross-compiled and run on every target, including the determinism check against an embedded SHA-256; an **independent Python checker** (`tools/check_loomora.py`: `zipfile` + `json` + `hashlib` only, no Loomora code) that validates every reference file and decodes `sequence.bin` and `mesh.bin`; a generator of the invalid and hostile corpus (`tools/gen_invalid_corpus.py`, build-time, nothing committed); a libFuzzer harness on the host; `loomora validate` on the whole corpus.

**Target Platform**: every platform of spec 001, both variants. Floors that shape the code: Windows 2000 (MinGW-w64, `msvcrt`, no `SetFilePointerEx`, no `ReplaceFile`), Android 2.3 (NDK r15c, 32-bit `off_t`), macOS 10.6 and iOS 5 (compiled only, per spec 001), Web (Emscripten, memory buffers, WebAssembly and JavaScript builds), Linux built on the target. The command `loomora validate` is a host program (Windows, Linux, macOS), not built for Android, iOS or Web.

**Project Type**: library (module of `libloomora`) + one host command + test and tooling code.

**Performance Goals**: SC-005: 5,000 nails and 100,000 steps per result validated and opened in under 1 s on a current desktop; a project 100 times larger opens without refusal. Throughput target of the streaming validation: at least 100 MB/s of archive on a current desktop (SHA-256 and inflate dominate).

**Constraints**: offline, no acceleration, no elevated rights (FR-032); no exception or C++ type through the C API (ADR 0001); memory bounded by one loaded range or one small item, never by the archive size; byte-identical output across architectures (FR-004); one implementation of the format (FR-031); every fallback declared in `validation/fallbacks.json`.

**Scale/Scope**: about 25 new source files in `core/src/format/` (about 6,000 lines), one C header (about 60 functions), one host command, about 15 test files, 3 ADRs, one rewrite of §9.1 of the requirements. Units: a typical project has a handful of items and 5,000 to 100,000 nails and steps; the tested maximum is 100 times that.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is still the **unfilled template** (as for spec 001): no principle is defined, so there is no constitutional gate to evaluate. The binding rules are those of [CLAUDE.md](../../CLAUDE.md), checked here instead:

| Rule | Status | How the plan respects it |
|---|---|---|
| Backend always C++11, standard as a target property, never global | pass | `loomora` target keeps `CXX_STANDARD 11`; the new sources join the same target; vendored C is a separate `C` target (`loomora_zlib`). |
| Fallbacks per case, never for everyone; declared in `fallbacks.json` | pass | Candidates in research R10 and R5 (32-bit offsets on Android API 9, file-size calls on Windows 2000, x87 float handling on i686 if measured necessary); each is a compile-time switch in `config.hpp` or a run-time check, and is declared. |
| Linux binaries built on the target; Windows cross-compiled from WSL; WSL builds under `$HOME` | pass | No change to the build model; the new targets use the existing toolchain files. |
| `verify.py quick` is the verification of any branch; `full` only for graphical changes | pass | This feature changes no graphics; every task is verified by `quick` (extended to build and run the new tests, the corpus and the checker). No screenshot. |
| Evidence stored only when it proves something new; screenshots are not evidence by default | pass | New proof: the reference files per architecture and the determinism hashes (R15), stored once under `validation/evidence/adr-0021/<platform-id>/`. Re-runs are mentioned in verdict notes, not stored. No image. |
| Implementation documentation in the docs submodule; ADRs numbered in order | pass | Three ADRs (0021, 0022, 0023) and the fallbacks, written in `docs/implementation` on a branch of that repository; the requirements document §9.1 is rewritten in the main repository. |
| Branches only, merge with `--no-ff` after the owner's approval | pass | Work on `feat/loomora-format` (and agent worktrees outside the repository); nothing merged without approval. |
| Commit messages in English, Conventional Commits, title at most 50 characters | pass | Applies to the implementation commits. |

**Deviations that need an ADR** (Complexity Tracking below): one rule of ADR 0001 (memory only in caller buffers) is extended by opaque handles; two third-party libraries are vendored. `LOOMORA_ABI_VERSION` stays **1** (owner decision 2026-11-17): the new functions are additive and a UI detects the project API with `loomora_format_version`.

**Post-design re-check**: unchanged. The design adds no principle conflict; the three deviations are recorded.

## Project Structure

### Documentation (this feature)

```text
specs/002-loomora-file-format/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── project-format.md        # byte-level format (becomes §9.1 of the requirements)
│   ├── c-api-project.md         # loomora_project.h
│   ├── cli-validate.md          # loomora validate
│   └── schemas/                 # JSON Schemas of the JSON entries
│       ├── manifest.schema.json
│       ├── spools.schema.json
│       ├── frame.schema.json
│       ├── layout.schema.json   # layout.json and nails.json
│       ├── image.schema.json
│       └── result.schema.json
├── checklists/requirements.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
core/
├── include/
│   ├── loomora.h                 # LOOMORA_ABI_VERSION stays 1; includes loomora_project.h
│   └── loomora_project.h         # handwritten C header of the format API (C89 valid)
├── src/
│   ├── c_api_project.cpp         # C binding: handles, exception catch, buffers
│   ├── format/
│   │   ├── limits.hpp            # warning thresholds, ratio guard, tolerances (one place, runtime setters for tests)
│   │   ├── testing.hpp           # fault-injection hooks (null in production)
│   │   ├── endian.hpp            # byte-wise little-endian read/write, varint
│   │   ├── sha256.{hpp,cpp}      # FIPS 180-4, streaming
│   │   ├── io/                   # source.hpp (random-access read), sink.hpp (write),
│   │   │   ├── file_posix.cpp    #   file_win32.cpp, memory_io.cpp; atomic replace,
│   │   │   ├── file_win32.cpp    #   share-delete open, 64-bit offsets
│   │   │   └── memory_io.cpp
│   │   ├── zip/                  # zip_reader.cpp, zip_writer.cpp (ZIP64, deterministic)
│   │   ├── json/                 # json_in.cpp (DOM and SAX with limits), json_out.cpp
│   │   ├── codec/                # sequence_codec, mesh_codec, nails_codec, image_meta
│   │   │                         #   (PNG/JPEG/WebP chunk walk and metadata strip)
│   │   ├── model/                # project.hpp, frame, layout, image, spool, result
│   │   ├── validate/             # validator.cpp, report.hpp (issues, severities)
│   │   ├── project_reader.cpp    # open, validate, item access, range reads
│   │   ├── project_editor.cpp    # staged edits, dependency removal, save (copy-through)
│   │   └── project_export.cpp    # subset and light export, EXIF strip
│   └── (existing files unchanged)
├── fuzz/                         # libFuzzer harness and seed corpus recipe (host, Clang)
├── loomora.exp  loomora.map      # export lists extended with the new functions
└── tests/
    ├── format/                   # Catch2 tests per module; golden hashes
    ├── format_selftest.cpp       # plain C++11 self-test run on every target
    ├── refgen.cpp                # loomora_refgen: writes the reference projects (full, light)
    ├── use_demo.cpp              # loomora_use_demo: read-only use, export, edit demo for the checks
    └── CMakeLists.txt            # extended (option LOOMORA_BUILD_FUZZ)
cli/loomora/main.cpp              # `loomora validate <file>` (host only)
third_party/
├── zlib/                         # submodule (madler/zlib, tag v1.3.1); built by cmake/loomora_zlib.cmake
└── rapidjson/                    # submodule (Tencent/rapidjson, pinned commit 297637bc); headers only
bridge/jni, bridge/wasm           # export lists and glue for the new functions
ui/*/                             # one `--validate-project <path>` probe mode per UI
tools/
├── check_loomora.py              # independent reader/checker (Python, standard library)
├── check_readonly.py             # proves use mode never writes (bytes, mtime, no new file)
├── run_vectors.py                # extended: --validate-project through each UI
├── gen_invalid_corpus.py         # builds the invalid and hostile files at build time
├── gen_reference.py              # expected content of the reference projects (JSON)
├── verify.py                     # extended: format tests, corpus, checker (quick)
└── check_std.py                  # extended: new exports
docs/requirements-specification.md  # §9.1, §9.2, §9.3, §10.2, §16.1 rewritten to match
docs/implementation/ (submodule)    # ADR 0021-0023, fallbacks.json, evidence, verdict notes
```

**Structure Decision**: the format is a module of the existing backend library (`core/src/format/`), not a new library, so that every UI that links `libloomora` has it and the single C boundary is kept (ADR 0001). The host command lives in `cli/loomora/`, the directory §14 of the requirements already reserves for it. Third-party code is vendored under `third_party/` (not a submodule, because the product build must not need a submodule). Spec 001 files are left as they are; spec 001's plan, research and contracts keep describing the probe.

## Delivery phases (input for `/speckit-tasks`)

Each phase ends with `python tools/verify.py quick` green on its branch.

0. **Spikes (gate the rest)**: S-F1 zlib and RapidJSON compile and pass a round trip on the lowest toolchains (MinGW-w64 Windows 2000 i686, NDK r15c, Emscripten fastcomp 1.39.20, Linux GCC and Clang `-std=c++11`); S-F2 identical bytes from the writer on i686 and x86_64 (and big-endian under emulation when available); S-F3 file calls on the Windows 2000 VM (atomic replace over a file opened for reading, `FILE_SHARE_DELETE`, 64-bit seek without `SetFilePointerEx`).
1. **Foundations**: `limits`, `endian`, `sha256`, I/O sources and sinks, ZIP reader and writer, JSON in and out; Catch2 tests and NIST/known-answer vectors.
2. **Model and codecs**: model structs, `sequence.bin`, `mesh.bin`, nails streaming, image metadata strip (PNG, JPEG, WebP).
3. **Reader and validator**: open, streaming validation, report, item and range access, light and version handling.
4. **Writer and editor**: staged edits, dependency removal, deterministic save, copy-through, subset and light export.
5. **C API and CLI**: `loomora_project.h`, `c_api_project.cpp`, export lists, `loomora validate`.
6. **Test corpus and proofs**: reference projects, invalid corpus, large-file corpus, golden determinism hashes in each self-test, independent checker, fuzzing run, cross-architecture reference files.
7. **UI probe modes**: `--validate-project` and `--roundtrip-project` in each UI and in `run_vectors.py`, per spec 001's PoC scope (lowest testable system for Legacy, one system for Modern).
8. **Documentation**: ADRs, fallbacks, requirements rewrite, verdict notes and evidence.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Opaque handles in the C API (extends ADR 0001 rule 3, "memory only in caller-supplied buffers") | Opening a project is stateful: a project of any size cannot be copied through a buffer, and range reads need the open archive. | A stateless API that re-opens and re-validates the archive on every call would hash the whole file for each read. Handles never expose memory; all data still crosses in caller buffers (ADR 0023). |
| Two third-party libraries as pinned git submodules (zlib, RapidJSON) | Deflate and inflate (FR-003) and a JSON parser and writer whose number text is platform-independent (FR-004). | Writing inflate and deflate and a correct double-to-text routine ourselves would be more code and more risk than the two small, proven libraries; the system zlib differs by platform and would break byte identity (ADR 0022). |
