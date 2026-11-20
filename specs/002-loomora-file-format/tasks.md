---

description: "Task list for the .loomora file format (C++11 backend module, C API, host command)"
---

# Tasks: The `.loomora` File Format

**Input**: Design documents from `/specs/002-loomora-file-format/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ (project-format.md, c-api-project.md, cli-validate.md, schemas/), quickstart.md

**Tests**: requested by the spec (FR-033, FR-034, SC-001 to SC-014): Catch2 tests on the host, a plain C++11 self-test on every target, an independent Python checker, a generated invalid and hostile corpus, fuzzing. In every story the tests are written before the code they cover and must fail first.

**Organization**: tasks are grouped by user story. Paths are relative to the repository root (`docs/implementation/` is the docs submodule, `../loomora-docs` repository).

## Rules that apply to every task (CLAUDE.md)

- Work on branch `feat/loomora-format` (or an agent worktree outside the repository on its own branch); commits in English, Conventional Commits, title at most 50 characters, imperative. Nothing is merged, and the docs submodule pointer is not moved, without the owner's explicit approval; docs changes go on a branch of `../loomora-docs`.
- Backend C++11 as a target property (`CXX_STANDARD 11`, `REQUIRED ON`, `EXTENSIONS OFF`); never a global standard. Fallbacks only for the failing platform, declared in `docs/implementation/validation/fallbacks.json`.
- **`LOOMORA_ABI_VERSION` stays 1** (owner decision 2026-11-17): never edit it; the project API is additive.
- Every WSL build goes under `$HOME` (for example `~/loomora-build/<target>`), never `/tmp` or the Windows tree.
- Verify a task with `python tools/verify.py quick`; `full` is not used (nothing graphical changes) and no screenshot is taken. Evidence is stored only when it proves something new (T062); a re-run is noted in the verdict, not stored. No image is committed.
- A task that needs an old system (Windows 2000 VM, Android emulator) ends with a `not_tested` record and its reason if the system cannot be obtained, never a skipped step.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: US1, US2, US3 to US7 (spec.md, "User Story 2" is the use-mode story); none for Setup, Foundational and Polish

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: skeleton, third-party submodules, build wiring, records

- [X] T001 Create the directory skeleton of plan.md with `.gitkeep` files: `core/src/format/{io,zip,json,codec,model,validate}`, `core/tests/format`, `core/fuzz`, `cli/loomora`, `third_party/{zlib,rapidjson}`
- [X] T002 [P] Add zlib 1.3.1 as the git submodule `third_party/zlib` (tag v1.3.1, tree untouched; the build uses `adler32.c crc32.c deflate.c inflate.c inffast.c inftrees.c trees.c zutil.c`) and `cmake/loomora_zlib.cmake` defining the C static library `loomora_zlib` (warnings off, position-independent, no `-Wall` inheritance)
- [X] T003 [P] Add RapidJSON as the git submodule `third_party/rapidjson` (pinned commit 297637bc); the include directory is marked SYSTEM wherever used so `-pedantic-errors -Werror` never applies to it
- [X] T004 Wire `core/CMakeLists.txt`: add the format sources to `LOOMORA_SOURCES` (keep the list in one `LOOMORA_FORMAT_SOURCES` variable), link `loomora_zlib` into `loomora` and `loomora_shared`, add the RapidJSON SYSTEM include, keep `CXX_STANDARD 11 CXX_STANDARD_REQUIRED ON CXX_EXTENSIONS OFF` on both targets, add option `LOOMORA_BUILD_FUZZ` (OFF); in `CMakeLists.txt` add `add_subdirectory(third_party/zlib)` before `core`
- [X] T005 [P] Create `core/src/format/limits.hpp` holding every constant of the data-model "Constants" table with its exact value (warn nails per layout 100,000; steps per result 5,000,000; triangles per mesh 5,000,000; decompressed total 256 MiB; items per kind 1,000; bomb ratio 250:1 after 1 MiB; thumbnail 256 px and 64 KiB; nail-in-frame tolerance 0.01 mm; nail overlap tolerance 1e-6 mm; JSON depth 64; entry name 512 bytes; progress callback every 4 MiB), held in one mutable internal `Limits` object (`limits()`; its setters are used only by the tests and by the hidden CLI option `--set-limit`, never exported from the shared library, never read from the environment), and create `core/src/format/testing.hpp`, an internal table of fault-injection hooks (function pointers, null in production: failed write, flush or rename, failed allocation) that the tests set
- [X] T006 [P] Extend `core/src/internal/config.hpp` with the format fallback switches, each default-off and enabled per target by toolchain file or platform macro: `LOOMORA_FMT_WIN2000_SEEK` (SetFilePointer high word), `LOOMORA_FMT_LSEEK64` (Android API 9 to 20), `LOOMORA_FMT_NO_LARGE_FILES` (reject files over 2 GiB with `LOOMORA_ERR_TOO_LARGE_FOR_PLATFORM`); comments name the `fallbacks.json` id each one declares
- [X] T007 [P] Add spikes S-F1 (zlib and RapidJSON build and round trip on the lowest toolchains), S-F2 (byte-identical writer output on i686 and x86_64, big-endian if available) and S-F3 (Windows 2000 file calls) to `docs/implementation/validation/spikes.json` with `status` = `open`, with the same fields as the spikes already in the file
- [X] T008 [P] Draft ADR 0021 (project file container, entry layout, identifiers; Context, Decision, Rejected alternatives, Consequences), ADR 0022 (vendored zlib and RapidJSON; why not the system zlib, minizip, miniz, nlohmann) and ADR 0023 (opaque handles extend rule 3 of ADR 0001; additive API with `LOOMORA_ABI_VERSION` unchanged) in `docs/implementation/adr/`, status `proposed`, from research R2 to R4, R7, R13; a `README.md` index line for each

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: primitives every story uses. **No user story starts before this phase is complete.**

**CRITICAL**: T009 and T010 gate the code that depends on them.

- [X] T009 Spike S-F1: add `core/tests/spike_fmt_deps.cpp` (deflate then inflate a 1 MiB buffer through the zlib submodule, write and parse a JSON document with a double table through RapidJSON with `kParseFullPrecisionFlag` and `kParseIterativeFlag`, print the SHA of the output) and build and run it with GCC and Clang `-std=c++11 -pedantic-errors`, MinGW-w64 `mingw-w64-win2000-i686.cmake` (run on the Windows 2000 VM), `android-ndk-r15c.cmake` (emulator or device), `emscripten-js-fastcomp.cmake` (Node); record each outcome in `spikes.json` (S-F1) and, for a toolchain that cannot build the C sources or the headers, write the per-platform fallback decision in the ADR 0022 draft
- [X] T010 Spike S-F3: write `tools/guest/fileio_probe.c` (C, runs on the system under test) that on one machine (a) opens a file with `FILE_SHARE_READ|FILE_SHARE_WRITE|FILE_SHARE_DELETE` then replaces it with `MoveFileExW(MOVEFILE_REPLACE_EXISTING|MOVEFILE_WRITE_THROUGH)` from a second handle, (b) seeks to 5 GiB with `SetFilePointer` high word on a sparse file where the file system allows it, (c) converts a UTF-8 path with non-BMP characters to UTF-16 and creates a file; run it on the Windows 2000 VM (NTFS and FAT32 if available) and on Windows 11; record in `spikes.json` (S-F3) and fix `LOOMORA_FMT_WIN2000_SEEK` decisions in T006 accordingly
- [X] T011 [P] Write `core/tests/format/test_endian.cpp` then `core/src/format/endian.hpp`: byte-wise little-endian read and write of u16, u32, u64 and IEEE 754 binary32 (never a pointer cast), LEB128 varint for u32 with "at most 5 bytes; a longer or non-minimal encoding is rejected"; test vectors independent of host byte order
- [X] T012 [P] Write `core/tests/format/test_sha256.cpp` (NIST FIPS 180-4 vectors: empty, `abc`, the 448-bit message, one million `a`, split updates at every offset) then `core/src/format/sha256.hpp` and `core/src/format/sha256.cpp` (streaming, byte-wise loads, no alignment assumption)
- [X] T013 [P] Create `core/include/loomora_project.h` (C89 valid): handle typedefs, `LOOMORA_KIND_*`, the new `LOOMORA_ERR_*` codes (-10 to -22), `loomora_progress_fn`, `loomora_sink_fn`, `loomora_format_version`, `loomora_format_limit`, `loomora_uuid4_from_random`; `#include` it at the end of `core/include/loomora.h` **without touching `LOOMORA_ABI_VERSION`**; create `core/src/c_api_project.cpp` with those three functions (`try`/`catch (...)` on each); add a test in `core/tests/format/test_c_header.cpp` that compiles the header as C89 and C++98 and checks `loomora_abi_version() == 1`
- [X] T014 [P] Create `core/src/format/report.hpp` and `core/src/format/report.cpp`: `Issue {severity (warning, error), code string, entry, item_kind (none, frame, layout, image, spool, result), item_id, message}`, `Report` (count, error count, add, merge), the `FormatError` exception carrying a `LOOMORA_ERR_*` code and optional issue, and the mapping from `std::bad_alloc` to `LOOMORA_ERR_NO_MEMORY` (the `LOOMORA_ERR_*` codes come from `loomora_project.h`, created by T013, which therefore comes first); test in `core/tests/format/test_report.cpp`
- [X] T015 [P] Create `core/src/format/io/source.hpp` (random-access read: `size`, `read_at(offset, buf, len)`), `core/src/format/io/sink.hpp` (sequential write plus `patch_at` for headers, `commit` and `abort`), and `core/src/format/io/memory_io.cpp` (memory source over a caller buffer without copy, memory sink with a growth callback and a callback sink for `loomora_sink_fn`); test in `core/tests/format/test_memory_io.cpp`
- [X] T016 [P] Create `core/src/format/io/file_posix.cpp`: read-only source (plain `open`, `pread`/`lseek` with 64-bit offsets, `LOOMORA_FMT_LSEEK64` switch for Android API 9 to 20, `LOOMORA_FMT_NO_LARGE_FILES` check), file sink writing `<name>.tmp-<random>` in the **same directory**, `fsync` (`F_FULLFSYNC` on macOS), `rename` replace, temporary file removed on `abort` or failure, the fault-injection hooks of `testing.hpp`; `#if` guarded so it compiles only on POSIX
- [X] T017 [P] Create `core/src/format/io/file_win32.cpp` (after T010): UTF-8 path to UTF-16 (`MultiByteToWideChar(CP_UTF8)`), read-only source via `CreateFileW(GENERIC_READ, FILE_SHARE_READ|FILE_SHARE_WRITE|FILE_SHARE_DELETE)`, seek with `SetFilePointer` high word under `LOOMORA_FMT_WIN2000_SEEK` (never `SetFilePointerEx`), sink to a temporary file in the same directory, `FlushFileBuffers`, `MoveFileExW(MOVEFILE_REPLACE_EXISTING|MOVEFILE_WRITE_THROUGH)` (never `ReplaceFile`), temporary file removed on failure; compiles for `_WIN32_WINNT=0x0500`
- [X] T018 Write `core/tests/format/test_file_io.cpp`: save to a temporary file then atomic replace; failure injected at every write, flush and rename step leaves the previous file byte-identical and no temporary file (SC-008); a reader opened before the replace still reads the complete old file; a read-only file opens; non-ASCII path (accents, emoji) on every host
- [X] T019 Write `core/tests/format/test_zip_reader.cpp` before the reader: a ZIP builder helper in the test builds archives byte by byte so each rule has a file: valid stored and deflate entries, ZIP64 (forced by lowering the ZIP64 thresholds through the `limits()` setters), data descriptor accepted, central directory authoritative, name rules ("relative, `/` separator, no empty, `.` or `..` component, no leading `/`, no `\`, no `:` in the first component, no NUL, no control character, at most 512 bytes, case-insensitive duplicates rejected"), symbolic link mode, overlapping entries, entry past the central directory, unknown method, encryption flag, comment present, local header mismatch, `mimetype` not first or not stored or wrong content
- [X] T020 Implement `core/src/format/zip/zip_reader.hpp` and `zip_reader.cpp` to pass T019: methods 0 and 8 only, ZIP64, central directory authoritative with local header cross-check, every name and layout rule of `contracts/project-format.md` section 1, streaming `open_entry` through a fixed buffer (inflate via zlib), raw compressed-bytes access (`read_raw`) for copy-through, every allocation sized from values already checked against the archive size
- [X] T021 Write `core/tests/format/test_zip_writer.cpp` before the writer: fixed header values (version made by `0x0314`, DOS time `0x0000` date `0x0021`, flags 0, external attributes `0x81A40000`), `mimetype` stored first at offset 0 with no extra field, ZIP64 only when needed (forced through the `limits()` setters), stored and deflate round trip with the T020 reader, deterministic golden SHA-256 of a small archive written twice
- [X] T022 Implement `core/src/format/zip/zip_writer.hpp` and `zip_writer.cpp` to pass T021: sink with `patch_at` (no data descriptors), `deflateInit2` level 6, window 15, memLevel 8, default strategy, fixed output buffer so chunking cannot change the stream, `add_stored`, `add_deflate`, `add_raw` (copy-through), canonical entry order supplied by the caller
- [X] T023 [P] Write `core/tests/format/test_json.cpp` then `core/src/format/json/json_in.{hpp,cpp}` and `json_out.{hpp,cpp}`: DOM parse with `kParseFullPrecisionFlag | kParseIterativeFlag`, depth limit 64, duplicate-key detection, no NaN or Infinity, SAX adapter for large arrays; compact writer with keys in caller-given order, `-0` written as `0`; tests: round trip of 100,000 random doubles including denormals, `1e-7`, `1e22`, integers above 2^53 rejected for identifiers, a depth-65 document rejected, a duplicate key rejected, `nan` rejected
- [X] T024 [P] Create the model structs in `core/src/format/model/` (`project.hpp`, `frame.hpp`, `layout.hpp`, `image.hpp`, `spool.hpp`, `result.hpp`) with exactly the fields and rules of data-model.md: title up to 1,024 bytes; item names up to 256 bytes; `kind` `2d` or `3d`; frame `dimensions` finite and `> 0`; outline `polyline` of at least 3 points; spool `opacity` in (0, 1], `diameter_mm > 0`; nail `diameter > 0`, `axis` unit vector within 1e-6, `face.u, v >= -1e-6` and `u + v <= 1 + 1e-6`; image `width_px, height_px >= 1`; result `spools` selection non-empty, no repeat, `length_mm > 0`; identifiers `u32`; plus `next_id` counters and `order` lists
- [X] T025 Extend `core/tests/CMakeLists.txt`: add the `core/tests/format/*.cpp` tests to `loomora_tests`; define `loomora_format_selftest` (plain C++11, no framework) with the same special cases as `loomora_selftest` (`LOOMORA_TARGET_WIN2000` GetThreadId shim and `LOOMORA_AVOID_IOSTREAM`, Emscripten exported `main`), registered with `add_test`; the tests need no compile definition (they use the `limits()` setters and the hooks of `testing.hpp`)
- [X] T026 Add the new `loomora_*` symbols to the export lists as each story adds functions: list the three functions of T013 in `core/loomora.exp`, `core/loomora.map`, `bridge/wasm/exports.txt` and `bridge/wasm/exports-selftest.txt` (each later task that adds a function adds its name there; `bridge/jni/loomora_jni.cpp` only gets what a JNI method needs), and extend `tools/check_std.py` so it fails when the shared build exports a `loomora_*` function missing from the header or any non-`loomora_*` symbol

**Checkpoint**: primitives pass their tests on the host (`ctest`); `python tools/verify.py quick` is green.

---

## Phase 3: User Story 1 - Save a project and open it again without loss (Priority: P1) MVP

**Goal**: write a 2D project (frame, layout, image, spools, result) to a deterministic `.loomora` file, atomically, and read it back unchanged.

**Independent Test**: build the reference `basic` project in memory, write it, read it back, compare every field and every step, write it twice and compare the bytes (quickstart steps 2).

### Tests for User Story 1

- [X] T027 [P] [US1] Write `core/tests/format/test_roundtrip.cpp`: reference project `basic` (one 2D frame, one layout of 200 nails, one image with a PNG, 3 spools, one result of 5,000 steps), full round trip with a field-by-field comparison, `empty` project (no item), a result selecting 40 spools whose steps use indexes 31 to 39 (escape path), the same project written twice gives identical bytes (FR-004), modification time comes from the caller (two saves with the same time are identical)
- [X] T028 [P] [US1] Write `core/tests/format/test_sequence_codec.cpp`: the record layout of `contracts/project-format.md` 5.1 (`LMSQ`, version, header flags, varint count, record flags: bit 0 rotation, bits 1 to 2 type, bits 3 to 7 spool index, 31 = escape then varint, level byte only with header bit 0), CRC-32 of all preceding bytes, rejection of type 3, of a non-minimal varint, of a count different from the record count, of a CRC mismatch, and the lossless rebuild of the machine step list of §8.3 (`from_nail, from_side, to_nail, to_side, spool_id, length`) on the §8.3 example, and the winding convention of FR-015 (rotation bit 0 is CW, 1 is CCW, seen from the nail tip in the front view; nothing is mirrored implicitly); about 3 bytes per step measured on 5,000 steps (under 20 KB, SC-006)
- [X] T029 [P] [US1] Write `core/tests/format/test_image_meta.cpp`: PNG with `eXIf`, `tEXt`, `iTXt` (XMP, GPS) chunks, JPEG with APP1 Exif GPS and APP13, WebP with `EXIF` and `XMP ` chunks, all built in the test; after stripping the pixel chunks are byte-identical, no GPS or camera tag remains (SC-009), a thumbnail wider than 256 px or larger than 64 KiB is rejected on write

### Implementation for User Story 1

- [X] T030 [P] [US1] Implement `core/src/format/codec/sequence_codec.{hpp,cpp}` to pass T028, including `rebuild_machine_steps` for the export feature
- [X] T031 [P] [US1] Implement `core/src/format/codec/nails_codec.{hpp,cpp}`: SAX parse of `nails.json` into compact arrays with an offsets index (one entry per 1,024 nails) for range reads, writer of the array with one nail per element and the key order of the schema (`id, x, y, z, diameter, kind, label, axis, length, side, face`), 2D and 3D fields, `nail_count` check; test in `core/tests/format/test_nails_codec.cpp` including 500,000 nails
- [X] T032 [P] [US1] Implement `core/src/format/codec/image_meta.{hpp,cpp}` to pass T029: PNG chunk walk, JPEG marker walk, WebP RIFF walk, container-level metadata strip (pixel data copied byte for byte), signature and structure checks, PNG `IHDR` size read for thumbnails
- [X] T033 [US1] Implement `core/src/format/project_editor.hpp` and `project_editor.cpp` (creation part): `create(project_id, title, created_at)`, `add_spool`, `add_frame` (2D), `add_layout`, `add_image`, `add_result`, identifier allocation from `next_id` counters that never decrease, `order` lists, `required_features` derivation, `generator` fields, canonical entry order of `contracts/project-format.md` section 2, JSON entries written with the key order of the schemas
- [X] T034 [US1] Implement `core/src/format/project_writer.cpp`: save to a path (atomic via T016/T017) and to a stream; default compression (deflate for `.json`, stored for `*.bin`, `*.png`, `*.jpg`, `*.jpeg`, `*.webp` and `mimetype`, option `all_stored`), manifest generated last with `size` and `sha256` of every entry (uncompressed), `modified_at` from the caller, progress callback every 4 MiB, abort removes the temporary file
- [X] T035 [US1] Implement `core/src/format/project_reader.hpp` and `project_reader.cpp` (reading part): open from path and memory without copying, streaming pass computing SHA-256 and CRC-32 of every entry and comparing with the manifest and the ZIP, manifest parse, item access (lists in `order`, item JSON, entry read by range), nails range, steps range through the sequence codec; read-only access only
- [X] T036 [US1] Complete `core/src/c_api_project.cpp` for this story: `loomora_editor_create`, `_add_spool`, `_add_frame`, `_add_layout`, `_layout_set_label`, `_add_image`, `_add_result`, `_set_title`, `_save_path`, `_save_stream`, `_close`; `loomora_project_open_path`, `_open_memory`, `_close`, `_manifest`, `_count`, `_ids`, `_item_json`, `_entry_size`, `_read_entry`, `loomora_layout_nail_count`, `_read_nails`, `_nail_label`, `loomora_result_step_count`, `_read_steps`; the report accessors `loomora_report_*`; every function catches all exceptions; list them in the export files of T026; struct layouts exactly as `contracts/c-api-project.md` (`loomora_nail` 88 bytes, `loomora_step` 12 bytes, static asserts in the tests)
- [X] T037 [P] [US1] Write `tools/check_loomora.py`, an independent checker using only `zipfile`, `json`, `hashlib` and `struct` (plus `jsonschema` for the schema check): verifies `mimetype` first and stored, manifest consistency (counts, `next_id`, `order`, `entries` sizes and SHA-256), validates each JSON entry against `specs/002-loomora-file-format/contracts/schemas/`, decodes `sequence.bin` (CRC, count, spool indexes) and `mesh.bin` (block CRCs, index range, flags against `has_normals` and `has_uvs`, UV section length), checks the references of data-model.md; options `--compare DIR_A DIR_B` (byte-identical files) and `--json`; exit status 0, 1, 2 as ADR 0015
- [X] T038 [US1] Write `core/tests/format/test_atomic_save.cpp` using the T016 fault hook: failure at every write, flush and rename step of a real save leaves the previous file intact (SC-008); a cancelled progress callback leaves it intact and returns `LOOMORA_ERR_CANCELLED`
- [X] T039 [US1] Write `core/tests/refgen.cpp` (`loomora_refgen`, host program) producing the `basic` reference project (full) and `core/tests/format_selftest.cpp` (plain C++11) that builds the same project in memory, writes it, compares the digest of the decompressed content with a constant (placeholder replaced in T060; research R15), reads it back, compares every field, and sweeps truncations of the small file; run `tools/check_loomora.py` on the output
- [X] T040 [US1] Run `python tools/verify.py quick` and fix until green; commit the story

**Checkpoint**: User Story 1 works alone: a 2D project is written deterministically, read back identical, and an independent Python reader agrees (MVP).

---

## Phase 4: User Story 2 - Use a project in use mode without changing it (Priority: P1)

**Goal**: opening, selecting and reading a project never writes, never changes the modification time, never creates a file, and works on a read-only location.

**Independent Test**: quickstart step 4.

- [X] T041 [P] [US2] Write `core/tests/format/test_readonly_open.cpp`: the file marked read-only opens; after opening, listing, reading a result's steps and a thumbnail, the file bytes and modification time are unchanged and no new file exists in its directory (SC-012); the reader opened before an atomic replace reads only the old file, and one opened after reads only the new file, never a mix
- [X] T042 [P] [US2] Write `tools/check_readonly.py` and `core/tests/use_demo.cpp` (`loomora_use_demo`, subcommands `list`, `result`, `thumb`; `export` and `edit` are added by later stories): the script marks a reference file read-only, records bytes and mtime, runs the demo, checks nothing changed, then replaces the file atomically while a second process holds the old one open and checks the reader's view
- [X] T043 [US2] Audit `core/src/format/project_reader.cpp`, `io/file_posix.cpp` and `io/file_win32.cpp` so the use-mode path requests only read access and sharing that lets another process replace the file, creates no temporary or lock file, and writes nothing; fix what T041 finds
- [X] T044 [US2] Run `python tools/verify.py quick`, add the `check_readonly.py` line to `tools/verify.py quick` (one line per check, logs under the build directory), commit the story

**Checkpoint**: use mode is proven read-only.

---

## Phase 5: User Story 3 - Keep several frames, layouts, images, spools and results in one project (Priority: P1)

**Goal**: 2D and 3D frames, several layouts per frame, images, project-level spools with per-result selection, several results; dependency-aware editing; copy-through save.

**Independent Test**: quickstart step 3 (`multi` reference: 3 frames, 5 layouts, 4 images, 6 spools, 5 results).

### Tests for User Story 3

- [X] T045 [P] [US3] Write `core/tests/format/test_multi_project.cpp`: reference `multi` (two 2D frames and one 3D frame; two layouts on one 2D frame, a heart and a circle; 4 images of mixed color mode and shape; 6 spools shared by results with different selections and lengths; 5 results; one frame-and-layout combination without result), round trip, display order, a result on a 3D frame with several views each tied to its own image, SC-011 sequence (save, open, remove one result, save again, every reference intact) (the validation errors of invalid items are tested in the corpus of US5)
- [X] T046 [P] [US3] Write `core/tests/format/test_editor_dependents.cpp`: one test per row of the dependency table and per line of the "Which changes count" table of data-model.md, through each function that makes the change (frame removed; frame outline, exclusions, dimensions, surface, body changed through `replace_item`; frame mesh changed through `replace_frame_mesh`; frame `name` or `background` changed; layout removed; nails changed through `replace_layout_nails`, a nail moved, added or removed; nail `label` or layout `name` changed; image removed; image pixels, `color_mode`, `shape`, dimensions changed through `replace_image`; image `name` or thumbnail replaced through `replace_thumbnail`; spool removed or `color`, `opacity`, `diameter_mm` changed; spool `name`, `available_length_mm`, `material`, `domain` changed; result removed or renamed; refused with `LOOMORA_ERR_INVALID_ITEM`: a layout's `frame`, a result's `frame`, `layout`, `images` or `spools`, a frame's `kind`), `loomora_editor_dependents` with `LOOMORA_OP_REMOVE` and `LOOMORA_OP_CHANGE` returning exactly what the matching call removes, `LOOMORA_ERR_HAS_DEPENDENTS` without confirmation with nothing changed, removal with confirmation, an identifier removed is never reused (also after save and reopen), after adding one result every other entry keeps the same SHA-256 (SC-013)
- [X] T047 [P] [US3] Write `core/tests/format/test_mesh_codec.cpp`: `mesh.bin` per `contracts/project-format.md` 5.2 (`LMMS`, flags bit 0 normals and bit 1 texture coordinates, counts `u64`, positions `f32` ×3, indices `u32` ×3, optional normals, optional UVs `f32` ×2 (finite), block CRC table per 1 MiB, table CRC), a mesh written with UVs then stripped of them keeps the same positions, indices and normals and a recomputed block table, range reads that verify the block CRC, index `< vertex_count`, counts equal to `frame.json`, byte-order-independent vectors

### Implementation for User Story 3

- [X] T048 [P] [US3] Implement `core/src/format/codec/mesh_codec.{hpp,cpp}` to pass T047
- [X] T049 [US3] Extend `core/src/format/project_editor.cpp` with 3D frames (`surface` in `mesh primitive extrusion revolution csg`, `body` in `solid shell guide`, mesh description with `has_normals` and `has_uvs`, `frame_set_mesh`, `replace_frame_mesh`), 3D nails (`z, axis, length, side, face`), result views (`position, target, up, projection, size or fov_deg, weight`), and the derivation of `required_features` (`surface_mesh`, `string_levels`, `multi_view_target`)
- [X] T050 [US3] Implement the dependency engine in `core/src/format/project_editor.cpp`: `dependents(kind, id, operation)` (`LOOMORA_OP_REMOVE`, `LOOMORA_OP_CHANGE`) following the dependency table and the "Which changes count" table of data-model.md (old and new value compared field by field), `replace_item` (JSON part; refuses with `LOOMORA_ERR_INVALID_ITEM` a change of a layout's `frame`, of a result's `frame`, `layout`, `images`, `spools`, or of a frame's `kind`), `replace_layout_nails`, `replace_frame_mesh`, `replace_image`, `replace_thumbnail` and `remove_item`, each with `confirm_dependents`, `set_order`, `set_title`, `next_id` never decreasing; layouts cannot change frame
- [X] T051 [US3] Implement copy-through save in `core/src/format/project_writer.cpp`: entries of untouched items are copied as compressed bytes from the old archive through `zip_writer::add_raw` (same SHA-256 in the new manifest), staged items are encoded, the manifest is regenerated; `editor_open_path` loads only the manifest and keeps the old archive open
- [X] T052 [US3] Extend `core/src/format/project_reader.cpp` for 3D: frame mesh info (`LOOMORA_MESH_NORMALS`, `LOOMORA_MESH_UVS`), vertex, triangle, normal and UV range reads with block CRC verification, views in result items
- [X] T053 [US3] Complete the C API for this story in `core/src/c_api_project.cpp` and the export files: `loomora_editor_open_path`, `_frame_set_mesh` (with the UV pointer), `_replace_item`, `_replace_layout_nails`, `_replace_frame_mesh`, `_replace_image`, `_replace_thumbnail`, `_layout_set_label`, `_frame_set_attached_source`, `_remove_item`, `_dependents` (with the operation argument), `_set_order`; `loomora_frame_mesh_info` (`flags`), `_read_vertices`, `_read_triangles`, `_read_normals`, `_read_uvs`; static asserts that `loomora_mesh_info` is 24 bytes
- [X] T054 [US3] Extend `core/tests/refgen.cpp` to write the `multi` reference project (full) and add the expected-content file `core/tests/format/multi.expected.json` produced by `tools/gen_reference.py` (an independent description of the content in JSON) and a `--scale N` option that multiplies the nails of each layout and the steps of each result by N (used by the large-project measurement); the Catch2 test compares the library's reading to it and `tools/check_loomora.py` re-derives the counts
- [X] T055 [US3] Run `python tools/verify.py quick`, run `tools/check_loomora.py` on `multi`, commit the story

**Checkpoint**: a multi-item project with 2D and 3D frames round-trips, edits respect dependencies, saves are copy-through.

---

## Phase 6: User Story 4 - Open files from any machine, including old and big-endian ones (Priority: P1)

**Goal**: version rules, cross-architecture identity, 32-bit and big-endian correctness.

**Independent Test**: quickstart steps 7 and 10, and the self-test hash on every target.

- [X] T056 [P] [US4] Write `core/tests/format/test_version_skew.cpp`: a file of format `1.7` with extra fields and an extra entry under `x-acme/` opens, the unknown entry is kept and byte-identical after an edit and save; format `2.0` is rejected with `LOOMORA_ERR_UNSUPPORTED_VERSION` and the message `This file needs Loomora format version 2.0 or newer`; an unknown required feature is rejected naming it (`LOOMORA_ERR_UNSUPPORTED_FEATURE`); a 2D-only project lists no required feature that only 3D needs
- [X] T057 [P] [US4] Write `core/tests/format/test_byteorder_vectors.cpp`: fixed byte arrays for `sequence.bin`, `mesh.bin`, varints and ZIP headers decoded byte by byte, with the expected values hard-coded, so the same test proves little-endian decoding on a big-endian target
- [X] T058 [US4] Implement the version and feature rules in `core/src/format/project_reader.cpp` and `project_editor.cpp`: parse `min_reader`, ignore unknown fields of a known major, preserve unknown entries and unknown `extensions` on rewrite, reject a higher major and an unsupported required feature with the messages above
- [X] T059 [US4] Spike S-F2: run `loomora_refgen` on Linux x86_64 and Linux i686 (`cmake/toolchains/linux-gcc-i686.cmake`), Windows x86_64 and Windows 2000 i686 (MinGW), and, when available, a big-endian system (QEMU user mode PowerPC on WSL counts as a trial, recorded as such); compare with `python tools/check_loomora.py --compare`; if the i686 doubles differ, add a per-target `-msse2 -mfpmath=sse` (or `-ffloat-store`) switch in the affected toolchain files only and declare it in T062; record the result in `spikes.json` (S-F2)
- [X] T060 [US4] Replace the placeholder constants of `core/tests/format_selftest.cpp` with the content digest (SHA-256 of the decompressed content, research R15) of `basic` and `multi` (full) produced on the host, and run `loomora_format_selftest` on every platform the project can run: Linux x86_64, Linux i686, Windows 11 x64, the Windows 2000 VM (Legacy floor), an Android emulator (`android-ndk-r15c.cmake`), Web through Node (`emscripten-js-fastcomp.cmake` and `emscripten-wasm.cmake`); per spec 001's PoC scope other rows stay `not_tested`; note each result in the verdict notes (not stored again)
- [X] T061 [US4] Add the format result lines to the verdict notes of the platforms run in T060 under `docs/implementation/validation/verdicts/` following the existing record format, and run `python tools/validate_records.py`
- [X] T062 [US4] Store the cross-architecture evidence once, under `docs/implementation/validation/evidence/adr-0021/<platform-id>/`: the reference `.loomora` files and a `hashes.json` per platform (ADR 0014: only what proves something new; no image); add the evidence index entry
- [X] T063 [US4] Declare each fallback actually taken in `docs/implementation/validation/fallbacks.json` with `platform_ids` (`win2000-setfilepointer` for `windows-2000-x86`, `android-api9-lseek64` for the Android platforms below API 21, the x87 switch if T059 needed it), each with `macro`, `min_compiler` and `reason`; run `python tools/validate_records.py`
- [X] T064 [US4] Run `python tools/verify.py quick`, commit the story

**Checkpoint**: version skew and cross-architecture identity are proven on the platforms that can run code.

---

## Phase 7: User Story 5 - Refuse damaged, hostile or unsupported files safely, warn on very large ones (Priority: P1)

**Goal**: every validation rule has a corpus file and an expected code; very large files open with warnings; the reader survives fuzzing.

**Independent Test**: quickstart steps 5 and 8.

### Tests for User Story 5

- [X] T065 [P] [US5] Write `tools/gen_invalid_corpus.py` (standard library only, builds into a directory passed as argument, nothing committed): one file per rule named `<kind>-<code>.loomora` for `zip.*` (absolute path, `..`, backslash, drive letter, NUL, case-duplicate, symbolic link mode, overlapping entries, unknown method, encrypted, missing or misplaced or wrong `mimetype`, truncated central directory, local header mismatch, inconsistent ZIP64), `entry.size`, `entry.sha256`, `entry.crc`, `entry.missing`, `entry.unlisted`, `entry.bomb` (ratio above 250:1 over 1 MiB), `manifest.*` (counts, `next_id` not above an identifier, `order` not a permutation, `entries` incomplete), `version.major`, `feature.unsupported`, `schema.*` (missing field, bad enum, duplicate key, depth 65, `1e999`), `id.duplicate`, `ref.missing`, `ref.layout_of_other_frame`, `ref.spool_not_selected`, `num.not_finite`, `nail.outside_frame`, `nail.duplicate_id`, `seq.header`, `seq.crc`, `seq.count`, `mesh.crc`, `mesh.index_range`, `spool.length_over_available`; truncations of the small reference at every byte (`truncate-NNNN`); and `large-*` files (nails, steps, triangles and item counts above the thresholds at a normal ratio, generated at small real sizes, to be opened with the thresholds lowered by the `--set-limit` option of the CLI or by the `limits()` setters of the tests, as the file name says)
- [X] T066 [P] [US5] Write `core/tests/format/test_validator_rules.cpp`: table-driven over the generated corpus, each file must give exactly its expected `code` with the expected severity and the entry and item named; every `truncate-*` is rejected without crash; every `large-*` opens with `warn.count` naming the element and its count, on open and again on save (FR-025); the minimum-distance case gives `warn.nail_distance` and still opens; a selection length above `available_length_mm` gives `spool.length_over_available`; a frame whose bounding-box centre is more than 0.01 mm from the origin gives `warn.origin_offset`
- [X] T067 [P] [US5] Write `core/tests/format/test_oom.cpp`: an allocation hook (replacement `operator new` in the test binary) fails the n-th allocation during open of the reference and of a large file; every outcome is `LOOMORA_ERR_NO_MEMORY` or success, never a crash or a leak reported by ASan

### Implementation for User Story 5

- [X] T068 [US5] Implement `core/src/format/validate/validator.{hpp,cpp}`: the semantic pass of research R11 and `contracts/project-format.md` section 8: manifest consistency, unique identifiers per kind and nail identifiers unique within the layout, every reference (layout to frame; result to frame, layout, image, view, spool; step to a nail of the result's layout and to an index of the selection; nail to face), "all numbers are finite", nails inside the frame "within 0.01 mm" with exclusions excluded (2D point-in-polygon; 3D: `face.triangle` below the mesh triangle count and barycentric coordinates in [-1e-6, 1 + 1e-6]), selection length not above `available_length_mm`, `sequence.bin` header and CRC and count, `mesh.bin` block CRCs and index range, thumbnails within 256 px and 64 KiB (warning), the centre of each frame's bounding box within 0.01 mm of the origin (warning `warn.origin_offset`, FR-015), minimum nail distance as the sum of the two radii minus 1e-6 mm on a uniform grid in linear time (warning), threshold warnings naming the element and its count
- [X] T069 [US5] Harden `core/src/format/zip/zip_reader.cpp`: the ratio guard (an entry over 1 MiB uncompressed with a ratio above 250:1 is `entry.bomb` / `LOOMORA_ERR_BOMB`), overlap and bounds checks, sizes checked against the archive size before any allocation, cancellation through the progress callback, `std::bad_alloc` mapped to `LOOMORA_ERR_NO_MEMORY`, a failed write or full disk mapped to `LOOMORA_ERR_IO` with the previous file intact (FR-026: memory and disk are not predicted, the error is defined as soon as an allocation or a write fails), nothing extracted to disk
- [X] T070 [US5] Make validation unconditional in `core/src/format/project_reader.cpp` and `project_editor.cpp`: open runs the container pass then the validator, reports every issue (continuing after an error whenever that is safe), returns `LOOMORA_ERR_CORRUPT` or the specific code, and a save runs the thresholds and returns its warnings in the report
- [X] T071 [US5] Complete the C API for this story in `core/src/c_api_project.cpp` and the export files: `loomora_validate_path`, `loomora_validate_memory`, the issue code table of `contracts/c-api-project.md`, `loomora_format_limit` values
- [X] T072 [P] [US5] Create `core/fuzz/fuzz_project.cpp` (libFuzzer entry `LLVMFuzzerTestOneInput` opening and validating the buffer, then reading every item) and `core/fuzz/fuzz_loop.cpp` (a mutation loop over corpus files for hosts without libFuzzer, ASan and UBSan), the CMake target `loomora_fuzz_project` under `LOOMORA_BUILD_FUZZ`, seed corpus recipe from T065 and the references, a 5-second per-input limit
- [X] T073 [US5] Run the fuzzing budget of SC-004 (at least 10 million inputs or one hour, whichever comes first) on the host with Clang, ASan and UBSan; fix every finding with a regression file added to the corpus recipe of T065; record the run (build, flags, inputs, time, result) in the verdict notes, not as stored evidence
- [X] T074 [US5] Measure SC-005 on a current desktop: a project of 5,000 nails and 100,000 steps per result validates and opens in under 1 s, and a project 100 times larger (`loomora_refgen --scale 100`: 500,000 nails, 10,000,000 steps) opens with its warnings and is not refused; add the numbers to the verdict notes
- [X] T075 [US5] Run `python tools/verify.py quick`, add the corpus-and-validator line to it, commit the story

**Checkpoint**: hostile and invalid inputs are rejected with their expected codes; large files open with warnings; fuzzing found nothing left.

---

## Phase 8: User Story 6 - Validate a file from the command line (Priority: P2)

**Goal**: `loomora validate <file>` prints each issue and exits 0, 1 or 2.

**Independent Test**: `contracts/cli-validate.md`, quickstart step 5.

- [X] T076 [P] [US6] Write `tools/check_cli_validate.py` (standard library only): runs `loomora validate` on the valid references (status 0), on every corpus file (status 1 and the expected `code` in the output, from the file name), on a warnings-only file (status 0 and `warn.*` printed), on each `large-*` file with the `--set-limit` options its name gives, on a missing path and on a directory (status 2), checks `--json` output parses and `--quiet` and `--max-issues N` behave as documented, several files give the highest status
- [X] T077 [US6] Implement `cli/loomora/main.cpp` and `cli/loomora/CMakeLists.txt`: options `--json`, `--quiet`, `--max-issues N`, `--progress`, `--version` and the hidden test-only `--set-limit name=value` (documented in `contracts/cli-validate.md`); calls `loomora_validate_path` only; output lines as `contracts/cli-validate.md` (UTF-8 on the Windows console as the existing host tools do); opens files read-only
- [X] T078 [US6] In `CMakeLists.txt` add `add_subdirectory(cli/loomora)` only for host builds (not `LOOMORA_TARGET_ANDROID`, `LOOMORA_TARGET_WEB`, iOS or cross toolchains of the Windows 2000 build), install the program; make `tools/check_std.py` ignore it
- [X] T079 [US6] Add the `check_cli_validate.py` line to `tools/verify.py quick`, run it and commit the story

**Checkpoint**: the command validates the whole corpus with the documented statuses.

---

## Phase 9: User Story 7 - Share a lightweight copy of a project (Priority: P3)

**Goal**: `light` and subset exports, non-editable light copies, mesh geometry preserved.

**Independent Test**: quickstart step 6.

- [X] T080 [P] [US7] Write `core/tests/format/test_export_light.cpp`: a `light` export drops `images/*/source.*`, `target.png`, `results/*/settings.json`, `mesh-source.*`, mesh `construction` and the mesh UVs (`has_uvs` false, `mesh.bin` rewritten without them), and keeps thumbnails and descriptions; the positions, triangle indices and normals of a 3D frame are identical to the full file, compared value by value and by a SHA-256 of those three sections (FR-017a); a subset export holds only the chosen results and the frames, layouts, images and spools they use, with the same `project_id` and correct `counts`, `next_id`, `order`; `editable` is false; editing a light project returns `LOOMORA_ERR_READ_ONLY_PROJECT`; a `light` to `light` subset works, a `light` to `full` export is refused; the light file is at least 50% smaller than the full one when the images have pixels (SC-006); a 5,000-step sequence is under 20 KB; embedded images carry no GPS or camera metadata (SC-009)
- [X] T081 [US7] Implement `core/src/format/project_export.cpp`: closure of a result subset, the stripping rules of `contracts/project-format.md` section 7 (including the rewrite of `mesh.bin` without its UV section and a recomputed block table), thumbnail checks (PNG, within 256 px and 64 KiB), manifest regeneration with `profile` `light` and `editable` false, unknown entries kept only in `full`, atomic path export and stream export, copy-through of the entries that stay unchanged
- [X] T082 [US7] Refuse the editor on a light project in `core/src/format/project_editor.cpp` (`editor_open_path` returns `LOOMORA_ERR_READ_ONLY_PROJECT` with the message "This project is a light copy and cannot be edited") and refuse a `full` export from a light project
- [X] T083 [US7] Complete the C API for this story: `loomora_project_is_light`, `loomora_project_export_path`, `loomora_project_export_stream`, the `LOOMORA_PROFILE_*` constants; export files; add the `export` subcommand to `core/tests/use_demo.cpp`
- [X] T084 [US7] Extend `core/tests/refgen.cpp` with `multi-light` and `one` (a light export of one result), extend `tools/check_loomora.py` with the light rules (no `source.*`, no `settings.json`, mesh SHA-256 equal across profiles with `--compare-mesh`), update the golden hashes of `core/tests/format_selftest.cpp`
- [X] T085 [US7] Repeat the cross-architecture proof for the light references: run `loomora_format_selftest` (with the golden hashes of `multi-light` and `one` added in T084) and `loomora_refgen` on the platforms of T060, compare with `python tools/check_loomora.py --compare`, store only the new light files and their hashes under `docs/implementation/validation/evidence/adr-0021/<platform-id>/` (the rule of T062: only what proves something new), add the result to the verdict notes of T061 and run `python tools/validate_records.py`
- [X] T086 [US7] Run `python tools/verify.py quick`, commit the story

**Checkpoint**: all six stories work; the format feature is complete at the backend level.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: UIs, documents, records, final proof

- [X] T087 Write `specs/002-loomora-file-format/contracts/ui-validate-project.md`: two probe modes: `--validate-project <path>` (Windows, macOS, Linux; intent extra `validate_project` on Android, launch argument `-validate_project` on iOS, URL parameter `?validate_project=<name>` on the Web, the file bundled with the page): the UI opens the file through `loomora_project_open_path` or `_open_memory` and prints one line identical to `loomora validate --quiet`; and `--roundtrip-project <path>` (same launch conventions with `roundtrip_project`): the UI opens the file, writes a full copy of every result through `loomora_project_export_stream` into memory, validates the copy with `loomora_validate_memory`, and prints `roundtrip ok` followed by the validation line of the copy; neither mode does anything else; no application logic in the UI (FR-027, SC-010)
- [X] T088 [P] Extend `tools/run_vectors.py` with `--validate-project FILE` and `--roundtrip-project FILE`: launches the built UI of a platform and variant in that mode, compares the line (for the round trip, the line of the copy) with `loomora validate --quiet` on the same file, same launchers as the probe mode
- [X] T089 [P] Add `--validate-project` and `--roundtrip-project` to the Windows Legacy UI in `ui/legacy/windows-win32/` (plain Win32 C++11; the GUI executable writes the line to a file next to the executable as the probe does)
- [X] T090 [P] Add `--validate-project` and `--roundtrip-project` to the Linux Legacy UI in `ui/legacy/linux-xlib/` and to the Linux Modern UI in `ui/modern/linux-gtk3/`
- [X] T091 [P] Add the Android bridge in `bridge/jni/loomora_jni.cpp` (open, validate, export to a byte array, read of an item and an entry through byte arrays) and the intent extras `validate_project` and `roundtrip_project` in `ui/legacy/android-views/` and `ui/modern/android-compose/`
- [X] T092 [P] Add the `LibraryImport` declarations (including `loomora_project_export_stream` with a callback, and `loomora_validate_memory`) and the `--validate-project` and `--roundtrip-project` arguments to `ui/modern/windows-winui3/` (C#, no C++/CLI)
- [X] T093 [P] Add the Emscripten exports in `bridge/wasm/exports.txt` and `module.cpp` (memory open, validate and export into a buffer; the file is read by the page and passed as a buffer) and the `?validate_project` and `?roundtrip_project` parameters to `ui/legacy/web/` (ES3/ES5-safe) and `ui/modern/web-ts/`
- [X] T094 [P] Add both modes to the Apple UIs (`ui/legacy/macos-appkit`, `ui/legacy/ios-uikit`, `ui/modern/macos-swiftui`, `ui/modern/ios-swiftui`); per spec 001 they are compiled on the `macos-26` runner and not run on old systems: records say "compiled, run `not_tested`" except where the runner can run the Modern UI
- [X] T095 Run `tools/run_vectors.py --validate-project` and `--roundtrip-project` on every UI the host can run (Legacy UI on the lowest testable system, Modern UI on one system and one 64-bit processor, per spec 001's PoC scope), record the verdicts (`SC-010`) in `docs/implementation/validation/verdicts/` and leave the other rows `not_tested`
- [X] T096 Rewrite §9.1, §9.2, §9.3, §10.2 and decision 16.1 item 5 of `docs/requirements-specification.md` from `contracts/project-format.md` (several frames, layouts, images, spools and results; no `progress.json`; no hard limits; light profile rules; `GET export.loomora` selects results; progress is application data keyed by project and result identifiers) and add a line to Annex B (revision history)
- [X] T097 Finalize ADR 0021, 0022 and 0023 in `docs/implementation/adr/` (status `accepted`, evidence and spike results linked, sources with the exact documents read: PKWARE APPNOTE, zlib manual, RapidJSON documentation, Microsoft `MoveFileEx` and `SetFilePointer`, Android NDK notes), mark spikes S-F1, S-F2, S-F3 `done` in `spikes.json`, run `python tools/validate_records.py`
- [X] T098 [P] Update the `tools/verify.py` self-test for the lines added by the story tasks (`check_readonly.py`, the corpus and validator line, `check_cli_validate.py`) and add the line for the UI probe check (`run_vectors.py --validate-project` and `--roundtrip-project` on the UIs the host can run), with logs under the build directory and the usual exit statuses
- [X] T099 Wire the new checks into the GitHub workflows (ADR 0018; FR-030 of spec 001) in `.github/workflows/ci.yml` and `tools/ci/`: on the hosted runners run the format Catch2 tests and `loomora_format_selftest` through `ctest`, `tools/gen_invalid_corpus.py` then `loomora validate` on the corpus, `tools/check_loomora.py` on the references, `tools/check_readonly.py`, `tools/check_cli_validate.py`, and a short fuzz smoke run (about 5 minutes with `loomora_fuzz_project`); the Windows 2000 VM, old Android and Apple old-system runs stay local records, never a CI job
- [X] T100 [P] Update `README.md` (build options `LOOMORA_BUILD_FUZZ`, the `loomora validate` command, the zlib and RapidJSON submodules) and `tools/measure_sharing.py` so the format module counts as shared backend code
- [X] T101 Run a review pass on `core/src/format/` with the `bansai:cpp-bug-hunt` skill (read only; the report the skill writes is moved into `docs/implementation/validation/` of the docs submodule, not kept in the main repository), triage each finding, fix the confirmed ones with a regression test and a commit each
- [ ] T102 Run the whole of `quickstart.md` (steps 1 to 10) on the branch, tick every expected outcome, list what was not run and why (platforms not available), mark FR-031 (the oldest toolchain of each floor built) and FR-032 (the self-test and the tools ran offline, without acceleration and without administrator rights) as checked or not, and report the branch, the commits, what was verified and what was not, then wait for the owner's approval before any merge or pointer move

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependency; T002, T003, T005 to T008 in parallel after T001.
- **Foundational (Phase 2)**: depends on Setup; blocks every story. T009 (S-F1) before T020, T022 and T023; T010 (S-F3) before T017; T013 (header and error codes) before T014; T011, T012, T013, T015, T016, T017 (after T010), T023 and T024 in parallel where their files differ; T019 before T020; T021 before T022; T025 and T026 after T013.
- **US1 (Phase 3, MVP)**: after Phase 2. T027 to T029 (tests) first, then T030 to T032 in parallel, then T033 → T034 → T035 → T036, then T037 to T039, T040 last.
- **US2 (Phase 4)**: after US1 (needs the reader and a reference file); can run in parallel with US3.
- **US3 (Phase 5)**: after US1 (extends the editor, writer and reader).
- **US4 (Phase 6)**: after US1; T059 to T062 need US3's `multi` reference and a running writer on each platform; T056 to T058 can start after US1.
- **US5 (Phase 7)**: after US1; T068 to T071 integrate with US3's items, so run after US3 for the full rule set; tests T065 to T067 can be written earlier.
- **US6 (Phase 8)**: after US5 (needs the validator and the corpus).
- **US7 (Phase 9)**: after US3 (3D meshes and subsets); independent of US6.
- **Polish (Phase 10)**: after the stories it needs; T089 to T094 need the C API of US1, US3, US5, US7.

### User Story Dependencies

- **US1** is independent. **US2**, **US3**, **US4** depend on US1 only. **US5** depends on US1 and is complete after US3. **US6** depends on US5. **US7** depends on US3.

### Within Each User Story

- Tests written first and failing; codecs and independent modules before the editor, writer and reader; the reader, writer and editor before the C API; the C API before tools and UIs; story complete (`verify.py quick` green) before the next priority.

### Parallel Opportunities

- Phase 1: T002, T003, T005, T006, T007, T008.
- Phase 2: T011, T012, T014, T015, T016, T017 (after T010), T023, T024, T013.
- US1: T027, T028, T029, then T030, T031, T032, T037.
- US3: T045, T046, T047, then T048.
- US4: T056, T057.
- US5: T065, T066, T067, T072.
- UIs: T089 to T094 are independent files.
- Agents each use their own worktree outside the repository and their own build directory (`verify.py` default is per worktree); an agent does not merge.

---

## Parallel Example: User Story 1

```bash
# Tests of User Story 1 together:
Task: "Write core/tests/format/test_roundtrip.cpp"            # T027
Task: "Write core/tests/format/test_sequence_codec.cpp"       # T028
Task: "Write core/tests/format/test_image_meta.cpp"           # T029

# Independent codecs together, once the tests exist and fail:
Task: "Implement core/src/format/codec/sequence_codec.{hpp,cpp}"  # T030
Task: "Implement core/src/format/codec/nails_codec.{hpp,cpp}"     # T031
Task: "Implement core/src/format/codec/image_meta.{hpp,cpp}"      # T032
Task: "Write tools/check_loomora.py"                              # T037
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 Setup, Phase 2 Foundational (with spikes S-F1 and S-F3 gating the dependencies and the Windows file layer).
2. Phase 3 US1: a 2D project written deterministically and read back, checked by an independent Python reader.
3. **Stop and validate**: `python tools/verify.py quick`, quickstart step 2, report the branch and wait for the owner's decision.

### Incremental Delivery

1. US2 (read-only proof) and US3 (the multi-item model, the core of the new format) next, in either order.
2. US4 (versions and cross-architecture identity), then US5 (safety) before the format is exposed to files from other machines.
3. US6 (command) and US7 (light and subset exports).
4. Polish: UIs, requirements rewrite, ADRs, final quickstart run.

### Reporting

Each finished phase is reported as a branch with its commits, what was verified (`quick` result, the platforms run) and what was not (`not_tested` rows); nothing is merged, and the docs pointer is not moved, until the owner approves.

---

## Notes

- Counts: 102 tasks (T001 to T102). Setup 8, Foundational 18, US1 14, US2 4, US3 11, US4 9, US5 11, US6 4, US7 7, Polish 16.
- Every task names the files it creates or changes; constraints from data-model.md and the contracts are quoted in the task that implements them.
- Out of scope here (spec Assumptions): the weaving progress, the transports, file association, the open and save dialogs, mesh import, path computation, the edit and use mode screens.

---

## Phase 11: Convergence

**Purpose**: remaining gaps found by `/speckit-converge` on 2026-11-18 (assessment of the code, not of the checkboxes). Most are the findings of the T101 review that were deferred for the owner, plus the unchecked part of T102. Each task that changes behaviour follows the same rule as the stories: failing test first, a commit each, nothing merged without the owner's approval.

- [ ] T103 Build the format module and run `loomora_format_selftest` on the floors still not built (CentOS 5.11 and 6.10 with gcc 4.8.2 devtoolset once the Docker daemon is started, osxcross 10.6, Xcode 5 for iOS 5), or record each as `not_tested` with its reason in the verdict notes of `docs/implementation/validation/verdicts/`, then update `acceptance-2026-11-17.md` (new dated section, the old one is not rewritten) to mark FR-031 as checked or as not checked with the exact missing floors, and tick T102 once the owner has accepted that record per FR-031 and T102 (partial)
- [X] T104 Clamp the length given to `avail_out` in `inflate_some` in `core/src/format/zip/zip_reader.cpp` (for example to 1 GiB, as `kMaxZlibChunk` does in `zip_writer.cpp`) so a read of a multiple of 4 GiB from a deflated entry no longer returns 0 bytes with the message "the entry is shorter than its declared size"; add a regression test that calls the function with a length of 2^32 through a test seam that needs no 4 GiB buffer per FR-027, FR-025 (T101 finding 5, partial)
- [X] T105 Ask the owner what saving through a symbolic link must do (follow it and replace the target, or replace the link as today), then implement the choice in `io/file_posix.cpp` and `io/file_win32.cpp` (temporary file next to the real file when the link is followed), document it in `contracts/c-api-project.md` and add a test with a link to a project file per FR-028 (T101 finding 3, partial)
- [X] T106 Replace the linear scan of `ZipReader::find` with a name index (`std::map`, C++11), make `ProjectReader::read_nails` on a deflated `nails.json` resume instead of restarting the inflate at byte 0 on each call, and load one label without the whole nail table in `loomora_layout_nail_label`; add a test with 5,000 items of one kind (above the 1,000 warning threshold) and a range-by-range read of 500,000 nails that fail on the quadratic cost, per FR-025 and SC-005 (T101 finding 6, partial)
- [X] T107 Ask the owner whether a calendar-invalid time stamp (`2026-02-31T00:00:00Z`, second 60) must be refused, warned about or still accepted, then implement the choice in `is_iso8601_utc` (`core/src/format/model/common.cpp`) and add the corpus file and test; a change that rejects files that open today is a version-skew question and is decided by the owner per FR-006 (T101 finding 4, partial)
- [X] T108 Remove the `<target>.old-xxxxxxxx` file or report a warning when the final `DeleteFileW` fails in `rename_aside` in `core/src/format/io/file_win32.cpp` (Windows before 10 1809), with a fault-injection hook in `testing.hpp` and a test per FR-028 (T101 finding 7, partial)
- [X] T109 Verify the CRC-32 of the compressed-then-inflated bytes (or the stored CRC from the central directory) while `zip_writer::add_raw` copies a kept entry in `project_writer.cpp`, so a project file rewritten in place between open and save cannot give a new file with stale sums; add a test that changes the old file between open and save per FR-023 (T101 finding 8, partial)
- [X] T110 Align the commands and expected outcomes of `quickstart.md` steps 2 to 6 with what the tools do (the self-test prints `format selftest: ok`; `loomora_refgen <dir>` writes `multi.loomora`; the Windows command is `loomora-cli`; `check_readonly.py --build-dir`; `loomora_use_demo export FILE OUT full|light [ID...]`; `--set-limit` for `large-*`; the light mesh keeps identical geometry but not the entry `sha256`), then tick the outcomes that now hold and leave unticked, with a reason, only the ones that need a system that is not available, per the "tick every expected outcome" rule of T102 (partial)
- [X] T111 Move the result records `core/tests/{android_results,cross_arch_results,web_results,perf_sc005,fuzz_run_sc004,spike_results_sf}.{md,json}` into `docs/implementation/validation/` (branch of `../loomora-docs`, only what the verdict notes and the evidence do not already hold), fix the links in `acceptance-2026-11-17.md` and `quickstart.md`, and delete them from `core/tests/`: implementation documentation lives in the submodule and is not part of the product tree per CLAUDE.md "Implementation documentation" (unrequested)

---

## Phase 12: Convergence

**Purpose**: gaps left by the Phase 11 fixes, found by the second `/speckit-converge` run on 2026-11-18. T103 (oldest toolchains and the old-system runs, which need the machines on the other computer) and T102 stay open and are not repeated here.

- [X] T112 Add a multi-threaded test of a shared read-only `loomora_project` handle (several threads reading nail ranges, labels and steps of the same project, run under ThreadSanitizer in the host fuzz/sanitizer build) that exercises the one-slot `std::atomic<NailCursor *>` cache of `ProjectReader` and the `inflated_bytes` hook, and fix any race it finds, per the thread-safety rule of `contracts/c-api-project.md` and FR-027 (partial)
- [X] T113 Add a hook in `core/src/format/testing.hpp` that disables `posix_rename` in `WinSink::replace_target` (`core/src/format/io/file_win32.cpp`) so the `rename_aside` path and the `warn.leftover_file` warning run for real on Windows 10 and 11, and replace the near-vacuous Win32 case of `core/tests/format/test_leftover_file.cpp` with one that fails if the `.old-` file is neither removed nor reported, per FR-028 (partial)
- [X] T114 Follow a dangling symbolic link on save under Windows (`FSCTL_GET_REPARSE_POINT` resolved at run time, never a hard import, no new import for the Windows 2000 build, `tools/check_imports.py --arch x86` still passing) so the behaviour matches POSIX, then remove the limit paragraph from `contracts/c-api-project.md` and un-skip the dangling-link test on Windows, per FR-028 and the T105 owner decision (partial)
- [X] T115 Describe the `validation/results/` category (spec 002 result records) in `docs/implementation` `data-model.md` and in the "Implementation documentation" list of `CLAUDE.md` (owner's file: ask before editing it), so `tools/validate_records.py` no longer reports it as "not described in data-model.md" per CLAUDE.md "Implementation documentation" (partial)
