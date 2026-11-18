# Research: The `.loomora` File Format

Every unknown of the Technical Context is settled here. Format: **Decision**, **Rationale**, **Alternatives considered**. Items marked *to measure* are decided now and gated by a spike (plan, phase 0); if the spike fails, the fallback written under the item applies to the failing platform only (CLAUDE.md, "Fallbacks").

## R1 — Architecture: a streaming format module inside the backend

**Decision**: the format is a module of `libloomora` (`core/src/format/`), behind a second C header. The reader **never holds an entry whole in memory**: it validates by streaming each entry through SHA-256 (and CRC-32 of the ZIP) and serves items on demand; bulk data (nails, steps, mesh) is read **by range** into caller buffers. The writer streams entries into a temporary file or memory sink.

**Rationale**: the spec removes every limit on element counts (FR-025) and asks for 32-bit and 1 to 2 GB machines (spec 001). Streaming bounds memory by one range or one small item, so "a machine that cannot hold a very large file" (User Story 5, scenario 8) only fails when a single small item does not fit.

**Alternatives**: load the whole project into a DOM (simple, but a 500,000-nail layout is hundreds of MB of DOM on a 32-bit device); a separate library (a second boundary, contrary to ADR 0001).

## R2 — ZIP container: own strict reader and deterministic writer

**Decision**: write the ZIP code for the project (about 800 lines): methods 0 (stored) and 8 (deflate) only; ZIP64 (reading always, writing only when a size or the entry count needs it); central directory authoritative; local header checked against it; no encryption, no data descriptors on write (the sink is seekable, so sizes are patched), tolerated on read. The writer sets every header field to a fixed value: version made by and needed, flags (bit 11 only for non-ASCII names), DOS time and date **1980-01-01 00:00:00**, no extra fields except ZIP64, no comment, external attributes fixed, **entry order fixed** (contract).

**Reader strictness** (zip-slip, bombs, FR-026): entry names must be relative, use `/`, contain no `..`, `.` or empty component, no drive letter and no NUL, be unique after case folding, be at most 512 bytes; Unix mode bits marking a symbolic link are rejected; entries must not overlap and must lie before the central directory; the sum of sizes is checked against the archive size and the ratio guard of R11; an unknown compression method or the encryption flag rejects the file.

**Rationale**: byte identity (FR-004) needs control of every byte; a third-party ZIP writer adds timestamps, extra fields and version-dependent choices. Reading needs strictness a general library does not offer, and a read path that never extracts to disk.

**Alternatives**: minizip or libzip (timestamps, extra fields, allocation of whole entries, API tied to files); miniz (single file, handy, but bundles its own deflate whose output differs from zlib's and is less documented across versions); `std::` has no ZIP. Rejected.

## R3 — zlib, a pinned submodule, for deflate and inflate only

**Decision**: include zlib 1.3.1 as the git submodule `third_party/zlib` (tag v1.3.1, upstream tree untouched); the build compiles only the files needed for `deflate`, `inflate` and `crc32` (`cmake/loomora_zlib.cmake`). It is built as a C target `loomora_zlib` and linked into `loomora` (and into `loomora_shared`). The writer calls `deflateInit2` with fixed parameters (level 6, `Z_DEFLATED`, window 15, memLevel 8, `Z_DEFAULT_STRATEGY`) and writes through a fixed-size output buffer so chunking cannot change the stream.

**Rationale**: zlib's deflate is pure portable C and gives the same bytes for the same input and parameters on every architecture and byte order **for one zlib version**; the system zlib differs by platform and version (and some distributions patch it), which would break FR-004. Pinning one source makes the output a property of the project. A later zlib upgrade changes the bytes: the reference hashes change with it, in a visible commit; stored (uncompressed) entries are unaffected.

**Defaults of the writer** (an option, FR-003): JSON entries deflate; `sequence.bin`, `mesh.bin` and image entries are stored (already dense or already compressed). `mimetype` is stored.

**Alternatives**: own deflate (a large, risky algorithm to write); system zlib (not identical, absent on some floors); stored-only writer (valid, but a typical `nails.json` is 10 to 20 times larger).

*To measure*: S-F1 compile and run on the lowest toolchains. If a toolchain cannot build the C sources, only that target falls back to stored-only writing (declared in `fallbacks.json`).

## R4 — JSON: RapidJSON, DOM for small entries, SAX for bulk

**Decision**: include RapidJSON (MIT) as the git submodule `third_party/rapidjson` (pinned commit 297637bc; only its headers are used). Parsing uses `kParseFullPrecisionFlag` (correctly rounded decimal-to-double), `kParseIterativeFlag` (no recursion, so depth cannot overflow the stack), a depth limit (64) and size limit from the entry, and no NaN or Infinity (the default, which enforces "all numbers are finite"). Small entries (manifest, frame, image, result, spools) are parsed into a DOM; **`nails.json` and any large array are parsed with the SAX reader** straight into compact structures, in the order of the contract, with range access for later reads. Writing uses `Writer` (compact, no whitespace) into a buffered sink, with object keys written in the fixed order of the contract.

**Rationale**: RapidJSON converts doubles with its own integer-only routines (Grisu2 for output, an exact algorithm for input), so the text depends neither on the C library, the locale nor `printf` of the platform, which differ on Windows 2000 (`msvcrt`), glibc, bionic and Emscripten. Header-only C++03-style code builds with every compiler of the floors, and SAX keeps memory low.

**Alternatives**: nlohmann/json (C++11, but locale-dependent number output through `snprintf` in older versions and heavy DOM for 100,000 nails); a handwritten JSON subset (about 600 lines but with the double conversion problem of R5 still to solve); cJSON (C, no correct double round trip).

## R5 — Numbers: exact round trip and identical text on every platform

**Decision**: every floating-point value in the file is a double written as the shortest-or-near-shortest decimal that parses back to the same double (RapidJSON's Grisu2), and parsed with full precision. Integers (identifiers, counts) are written as integers. Binary floats (`mesh.bin`) are IEEE 754 binary32 stored through a byte-wise little-endian conversion, never through a cast of a pointer. The model rejects a non-finite value at validation.

**Rationale**: SC-001 (no difference in any field) and SC-002 (byte identity across architectures) both need the decimal text to be a function of the double alone.

*To measure*: S-F2 on i686 (x87 extended precision of old MinGW, NDK r15c, Linux i686) against x86_64. **Fallback** (only if the i686 run differs): compile the format module for that target with `-msse2 -mfpmath=sse` (or `-ffloat-store` where SSE2 does not exist), declared as a fallback for the platforms taking it.

## R6 — SHA-256, CRC-32, UUID

**Decision**: SHA-256 written for the project (FIPS 180-4, about 120 lines, streaming, byte-wise loads so it is correct on big-endian and unaligned access), tested with the NIST short and long message vectors. CRC-32 comes from zlib (`crc32`) for the ZIP entries and for `sequence.bin` and mesh blocks. A project identifier is a UUID v4 **supplied by the caller** (16 random bytes from the platform's random source, through `loomora_uuid4_from_random`), so the format code reads no clock and no random source (determinism, FR-004).

**Alternatives**: a vendored SHA library (more code to vet than the algorithm itself); platform crypto APIs (absent on Windows 2000 and old Android, different results are not possible but availability is).

## R7 — Entry layout and identifiers

**Decision** (full detail in [contracts/project-format.md](contracts/project-format.md)): one folder per item, named by its numeric identifier, grouped by kind:

`mimetype`, `manifest.json`, `spools.json`, `frames/<fid>/{frame.json, mesh.bin, mesh-source.<ext>, layouts/<lid>/{layout.json, nails.json}}`, `images/<iid>/{image.json, source.<ext>, target.png, thumb.png}`, `results/<rid>/{result.json, sequence.bin, thumb.png, settings.json}`.

Identifiers are unsigned 32-bit integers allocated by the project from per-kind counters stored in the manifest (`next_id`), which are never decremented: a removed identifier is never reused (FR-014b). The display order of the items is a list per kind in the manifest. Entry names use only `[A-Za-z0-9._/-]`.

**Rationale**: FR-002 asks that adding, removing or replacing an item not rewrite the others; with one folder per item, an edit touches only that folder, and the manifest. A layout sits under its frame in the path, which states that it belongs to one frame. Counters in the manifest make FR-014b a property of the file, not of the application. §9.1 of the requirements is rewritten from this layout.

**Alternatives**: one big `project.json` (any edit rewrites everything and the whole thing must be parsed to list results); UUID item identifiers (longer paths, no gain, the project already has a UUID); a SQLite file (rejected for the format by decision 5 of the requirements, and unavailable on the floors).

## R8 — `sequence.bin`, `mesh.bin`, nails

**Decision**: `sequence.bin` keeps the §9.1 layout (`LMSQ`, version, flags, varint count, records, CRC-32) with one change: the **spool field is the index into the result's spool selection** (`result.json`), not the project spool identifier, which keeps about 3 bytes per step whatever the identifiers; index 31 is the escape followed by a varint, as in §9.1. `mesh.bin` (new, little-endian): magic `LMMS`, version, flags (bit 0 normals, bit 1 texture coordinates), vertex count `u64`, triangle count `u64`, positions `f32` ×3, indices `u32` ×3, normals `f32` ×3 when flagged, UVs `f32` ×2 when flagged (full profile only; the light export rewrites the entry without them), trailing CRC-32 of the header and body per block of 1 MiB (so a range read can verify what it reads). Nails stay JSON (`nails.json`, an array of compact objects), as decided in §9.1, read by SAX; a range read of nails uses an index of offsets built during the validation pass and kept in memory (8 bytes per 1,024 nails).

**Rationale**: indexed spools remove the dependency between the compact sequence and the project's identifier space; block CRCs give integrity for range reads without hashing the whole mesh again.

**Alternatives**: spool identifiers inside the sequence (a 10-bit varint for a project with many spools, and a cross-reference to the project each decode); binary nails (smaller and faster but against the "readable without Loomora" principle of §9.1, which the owner validated).

## R9 — Images: no decoding in the backend

**Decision**: the format layer **does not decode images**. It stores the bytes the application gives it for source, processed target and thumbnail, checks the signature and basic structure (PNG chunks, JPEG markers, WebP RIFF), and **strips metadata at container level**: PNG `eXIf`, `tEXt`/`iTXt`/`zTXt` carrying XMP or GPS, JPEG APP1 (Exif and XMP) and APP13, WebP `EXIF` and `XMP ` chunks; pixel data chunks are copied byte for byte. Thumbnails are PNG supplied by the application, at most 256 × 256 pixels and 64 KiB (constants); the format layer rejects a larger one on write and warns on read.

**Rationale**: image processing is another feature (spec Assumptions); decoding and resizing in the format code would need image libraries on every floor. Chunk-level stripping does not change a pixel (SC-009, spec assumption), and is a small, auditable walk. After stripping, the entry's SHA-256 is of the stripped bytes.

**Alternatives**: decode and re-encode (changes pixels, needs libraries); trust the caller (violates FR-019).

## R10 — File access, atomic save, per-platform switches

**Decision**: a thin internal layer (`io/`) with three implementations: POSIX, Win32, memory. Paths come in UTF-8 and are converted to UTF-16 on Windows (`MultiByteToWideChar`, already exercised on Windows 2000, ADR 0004 audit 9).

- **Save** writes `<name>.loomora.tmp-<random>` in the **same directory**, flushes (`FlushFileBuffers`, `fsync`, `F_FULLFSYNC` on macOS), then replaces the target: POSIX `rename`; Windows `MoveFileExW(MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH)`, which exists on Windows 2000 (`ReplaceFile` is XP and later and is **not** used). On any failure the temporary file is removed and the previous file is untouched (FR-028).
- **Read** opens for read with sharing that lets another process replace the file: Win32 `FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE`, POSIX plain `open`; a reader that started before a save keeps the old inode or handle and sees the complete old file (User Story 2, scenario 3). Use mode never opens with write access and never creates a file.
- **64-bit offsets** (ZIP64): Win32 `SetFilePointer` with the high word (not `SetFilePointerEx`, XP and later), POSIX `lseek` with `_FILE_OFFSET_BITS=64`; on **Android API 9 to 20** (`off_t` is 32-bit, `lseek64` exists) the layer uses `lseek64`; if a platform offers none, files over 2 GiB are rejected with a defined error (a fallback taken by that platform only).
- **Memory I/O** (Web, mobile sandboxes, tests): the project can be opened from and saved to a caller buffer; the Web build uses it with the browser's file APIs.

**Fallbacks to declare** (`fallbacks.json`, per platform): `win2000-setfilepointer` (high-word seek), `android-api9-lseek64`, and, if S-F3 shows it, a different flush call on a platform that lacks `FlushFileBuffers` semantics. Each is a compile-time switch in `config.hpp`.

**Rationale**: the "complete old or complete new, never a mix" rule needs same-directory temporary and atomic replace; the share-delete flag is what makes the replace work on Windows while a use-mode reader has the old file open.

*To measure*: S-F3 on the Windows 2000 VM (replace over a file held open for reading; behaviour on FAT32 and NTFS).

## R11 — Validation, warnings, bomb guard, thresholds

**Decision**: validation is one pass over the archive then a semantic pass over the manifest and small entries:

1. **Container pass** (streaming): central directory checks (R2), then for each entry inflate or copy through SHA-256 and CRC-32, compare with the manifest (`size`, `sha256`) and the ZIP CRC; reject on mismatch naming the entry.
2. **Semantic pass**: manifest consistency (counts, `next_id` above every identifier, entry list equals archive content except unknown entries), unique identifiers, all references (R7 and FR-024), numbers finite, nails inside their frame, steps referring to nails of the layout and to indices of the selection, spool lengths within the available length, `sequence.bin` header and CRC, `mesh.bin` blocks, thumbnails within the size constants. Nail-to-nail minimum distance (a **warning**) is computed on a uniform grid (linear time), as the sum of the two radii.
3. **Warnings** (never block): counts above the thresholds of `limits.hpp`, nail distance, light copy opened, thumbnail over size, unknown required-feature-free extensions kept.

**Thresholds** (project constants, FR-025; starting values, tunable): nails per layout 100,000; steps per result 5,000,000; triangles per mesh 5,000,000; decompressed total 256 MiB; items per kind 1,000; thumbnail 256 × 256 px and 64 KiB. **Tolerances**: nail inside frame 0.01 mm (3D: face index valid and barycentric coordinates in [−1e-6, 1 + 1e-6]); nail overlap tolerance 1e-6 mm.

**Decompression bomb guard**: per entry, an expansion ratio above **250 to 1** is rejected once the entry exceeds 1 MiB uncompressed (zlib's theoretical maximum is about 1032 to 1, and real JSON compresses 5 to 20 to 1); entries are inflated through a fixed buffer, so a bomb costs time, not memory; the reader also rejects overlapping entries (the usual way to multiply a payload). An out-of-memory or out-of-disk condition anywhere is a defined error (`LOOMORA_ERR_NO_MEMORY`, `LOOMORA_ERR_IO`). A progress and cancel callback lets the application stop a long validation.

**Rationale**: the spec forbids a fixed ceiling and asks to refuse only an abnormal ratio (memory and disk conditions are defined errors when an allocation or a write fails, FR-026).

**Alternatives**: a fixed 256 MiB cap (contradicts the clarification); no guard (a 40 MB file could expand to many gigabytes of hashing time).

## R12 — Edit model and copy-through save

**Decision**: edit mode works on an **editor** object opened on a `full` project (or created empty). Operations stage changes in memory or temporary files: add, replace and remove frame, layout, image, spool and result; each operation computes the **dependents** (data-model rules), and a removal of dependents happens only when the caller passes the confirmation flag (the application asked the person). **Save** writes a new archive: entries of untouched items are **copied as compressed bytes straight from the old archive** (same checksums, SC-013, and no recompression), staged items are encoded, the manifest is regenerated (counts, entries, `modified_at` given by the caller, `next_id` counters), then the atomic replace of R10. A `light` project refuses the editor (FR-017b).

**Rationale**: copy-through keeps a save of a large project proportional to what changed and keeps untouched entries byte-identical by construction.

**Alternatives**: decode and re-encode everything (slow, risks changing bytes); in-place append to the ZIP (unsafe on crash, contradicts FR-028).

## R13 — C API shape

**Decision** ([contracts/c-api-project.md](contracts/c-api-project.md)): opaque handles (`loomora_project*`, `loomora_editor*`, `loomora_report*`) created and destroyed by the library; **all data crosses in caller-supplied buffers**, with the size-query convention (call with `cap = 0`, get the length, call again); metadata items as UTF-8 JSON documents (their schema is the contract); bulk data as fixed-layout C structs read by range (`loomora_layout_read_nails(handle, layout_id, first, count, out, cap)`); strings in and out UTF-8; every function returns a negative `LOOMORA_ERR_*` code on failure and never throws. The validation report is a handle with accessors (severity, code, entry, item kind and identifier, message), not a text blob, so a UI can show it its own way.

**Rationale**: matches the existing API conventions (buffers, error codes, no C++ type) and the language reach of the UIs (Java 5-era JNI, C# `LibraryImport`, Objective-C, JavaScript on Emscripten, all of which can handle opaque pointers and byte buffers, and all of which parse JSON natively or through the platform).

**Alternatives**: one function per field (hundreds of functions); returning library-allocated JSON (breaks ADR 0001 rule 3 more widely); a callback-based visitor (awkward from Java and JavaScript).

ADR 0023 records the extension of rule 3 to opaque handles with explicit destroy functions.

## R14 — Versions and required features

**Decision**: format `major 1, minor 0`; `min_reader` is the string `"1.0"`. The implementation reads `1.x` for any `x` (ignores unknown fields, keeps unknown entries listed in the manifest on rewrite, FR-020); rejects `major > 1` with the message `This file needs Loomora format version M.m or newer` (FR-021). Required features known to this implementation: `surface_mesh`, `string_levels`, `multi_view_target` (all handled). A project with only 2D frames lists none that 3D needs (FR-022). The profile is in the manifest (`full`, `light`); `light` is also written as the manifest key `editable: false`.

**Rationale**: the format has not shipped, so no migration code is needed now; the rule for later majors is already in the contract.

## R15 — Test strategy and proofs

**Decision**: five layers.

1. **Catch2 unit tests** per module on the host (SHA-256 vectors, varint, ZIP read and write including ZIP64 and every rejection rule, JSON limits, codecs, validator rules, editor dependency removal).
2. **Self-test on every target** (`loomora_format_selftest`, plain C++11): builds a fixed reference project in memory, writes it, checks the **SHA-256 of the produced bytes against a constant embedded in the source** (cross-architecture identity of FR-004 and SC-002, with no file exchange), reads it back, compares field by field, runs a truncation sweep over a small file and a set of embedded hostile inputs. It is run on each platform the project can run code on (Windows 2000 VM included).
3. **Independent checker** `tools/check_loomora.py`: reads every reference file with `zipfile`, `json`, `hashlib` only, verifies manifest checksums, schemas (`jsonschema`), decodes `sequence.bin` and `mesh.bin`, and re-derives the counts: it proves the claim "readable without Loomora" and cross-checks the implementation with an unrelated reader.
4. **Corpus**: `tools/gen_invalid_corpus.py` builds, at build time, one hostile or invalid file per validation rule (and per ZIP rule), truncations of a small file at every byte, and the large-file cases (sparse data compressed to a few KB, generated with the standard library); `loomora validate` and the Catch2 tests must give the expected issue for each.
5. **Fuzzing**: a libFuzzer harness on the host (Clang with `-fsanitize=fuzzer,address,undefined`), seeded from the reference and invalid corpora, run for a fixed budget per platform with a time limit per input (SC-004). Where libFuzzer is unavailable, the harness is a loop over mutated corpus files with ASan and UBSan.

**Evidence stored** (CLAUDE.md): the reference files written on each architecture and the determinism hashes, once, under `validation/evidence/adr-0021/<platform-id>/`; big-endian files only if a big-endian system is available (QEMU user mode on PowerPC counts as a trial, recorded as such). Re-runs are noted in verdicts, not stored. No screenshot.

## R16 — Requirements and spec 001 consistency

**Decision**: the implementation changes §9.1 (entry layout, spools at project level, results, layouts, no `progress.json`, no hard limits, light profile rules, `sequence.bin` spool index), §9.2 (`GET export.loomora` selects results and profile; the `progress` route belongs to application data), §9.3 (BLE sends a light copy of selected results), §10.2 (progress is application data keyed by project and result identifier) and decision 16.5 (the format is implemented as specified by `contracts/project-format.md`). The rewrite is a task of phase 8, in the main repository, on its own branch. Spec 001 is not edited.

## Sources

References for the decisions, to be cited with their versions in the ADRs when they are written (none was fetched for this plan; each ADR cites the exact document read):

- PKWARE ZIP APPNOTE 6.3.x (ZIP, ZIP64), RFC 1951 (deflate), RFC 1950 (zlib), zlib manual and FAQ (maximum deflate ratio, deterministic output), zlib `deflateInit2` parameters.
- NIST FIPS 180-4 and the NIST example values for SHA-256.
- RapidJSON documentation: parse flags (`kParseFullPrecisionFlag`, `kParseIterativeFlag`), the Grisu2 `dtoa` of `Writer`.
- Microsoft documentation: `MoveFileEx`, `CreateFile` sharing modes (`FILE_SHARE_DELETE`), `SetFilePointer`, minimum supported client of `SetFilePointerEx` and `ReplaceFile`.
- Android NDK notes on `off_t` and `lseek64` before API 21; Apple `fcntl(F_FULLFSYNC)`.
- PNG specification (chunks `eXIf`, `tEXt`, `iTXt`), JPEG (APPn markers), WebP RIFF container (`EXIF`, `XMP `).
- Existing project records: ADR 0001, 0004, 0011, 0013, 0014, 0015, `validation/fallbacks.json`.
