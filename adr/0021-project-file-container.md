# ADR 0021: Project file container, entry layout and identifiers

- **Status**: proposed
- **Date**: 2026-11-17
- **Spike / requirement**: S-F2, S-F3 / FR-001 to FR-004, FR-014b, FR-020, FR-026, FR-028 (spec 002)

## Context

Spec 002 defines the `.loomora` project file: frames, nail layouts, images, spools and results in one file, readable without Loomora, never rewritten as a whole for a small edit (FR-002), byte-identical when the same project is written again with the same options, on any platform, architecture, byte order and date (FR-004), and safe to open when the file is hostile (FR-026). The floors are old (Windows 2000, 32-bit devices with 1 to 2 GB of memory). The detail of the format is the contract `contracts/project-format.md` of spec 002; this record keeps the reasons.

## Decision

- **Container**: a ZIP archive (extension `.loomora`, MIME type `application/vnd.loomora+zip`), written and read by project code (about 800 lines, `core/src/format/zip/`). Methods 0 (stored) and 8 (deflate) only; no encryption. ZIP64 is read whenever present and written only when a size, an offset or the entry count needs it. The central directory is authoritative and each local header is checked against it.
- **Deterministic writer**: every header field has a fixed value (version made by and needed, flags, DOS date 1980-01-01 00:00:00, no extra field except ZIP64, no comment, fixed external attributes) and the entry order is fixed by the contract. `mimetype` is first, stored, at offset 0. Deflate uses fixed parameters (level 6, window 15, memLevel 8, default strategy, see ADR 0022). No data descriptor is written (the sink is seekable, sizes are patched).
- **Strict reader** (zip-slip, bombs): relative names with `/`, no `.`, `..` or empty component, no drive letter, no NUL, unique after case folding, at most 512 bytes; symbolic links rejected; entries must not overlap and must lie before the central directory; an unknown method or the encryption flag rejects the file; an expansion ratio above 250 to 1 after 1 MiB is rejected; everything is inflated through a fixed buffer. Nothing is ever extracted to disk.
- **Entry layout**: one folder per item, named by its decimal identifier, grouped by kind: `manifest.json`, `spools.json`, `frames/<fid>/{frame.json, mesh.bin, layouts/<lid>/{layout.json, nails.json}}`, `images/<iid>/...`, `results/<rid>/{result.json, sequence.bin, thumb.png, settings.json}`. An edit touches only the folder of the item and the manifest. Entries unknown to a version are listed, kept and copied on rewrite (FR-020).
- **Identifiers**: unsigned 32-bit integers allocated per kind from counters stored in the manifest (`next_id`), never decremented, so a removed identifier is never reused (FR-014b). The display order is a list per kind in the manifest.
- **Save**: a temporary file in the same directory, flush, then atomic replace (POSIX `rename`; Windows `MoveFileExW(MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH)`, never `ReplaceFile`, which needs Windows XP). On failure the temporary file is removed and the previous file is untouched (FR-028). Entries of untouched items are copied as compressed bytes from the old archive.
- **Read** opens with sharing that lets another process replace the file (Win32 `FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE`) and never with write access.
- **64-bit offsets**: Win32 `SetFilePointer` with the high word (not `SetFilePointerEx`), POSIX `lseek` with `_FILE_OFFSET_BITS=64`, `lseek64` on Android API 9 to 20. Each is a compile-time switch in `core/src/internal/config.hpp`, taken only by the platform that needs it and declared in `validation/fallbacks.json` (`win2000-setfilepointer`, `android-api9-lseek64`).

Two points are decided now and gated by spikes: the byte identity on i686 (x87 extended precision, **pending spike S-F2**; if it differs, only the affected target is compiled with `-msse2 -mfpmath=sse` or `-ffloat-store`, declared as a fallback) and the Windows 2000 file calls (replace over a file held open, high-word seek, UTF-8 to UTF-16 paths, **pending spike S-F3**).

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
- To verify: S-F2 (identity across architectures, big-endian only when a system is available, recorded as a trial) and S-F3 (Windows 2000, NTFS and FAT32 if available). Until then the Windows 2000 and Android fallbacks above are planned, not measured. Evidence goes under `validation/evidence/adr-0021/<platform-id>/`, once.

## Sources

Documents read to write this draft (no external source was fetched):

- `specs/002-loomora-file-format/research.md`: R2, R3, R5, R7, R10, R11, R12, R15
- `specs/002-loomora-file-format/plan.md`: Technical Context, Constitution Check, Project Structure
- `specs/002-loomora-file-format/contracts/project-format.md`: sections 1 to 3 (container, entry order, manifest; the rest not read for this draft)
- `specs/002-loomora-file-format/spec.md`: FR-001 to FR-004 (the other requirements are cited from `research.md`)
- To read before acceptance (listed by `research.md`, not read yet): PKWARE APPNOTE 6.3.x, Microsoft `MoveFileEx`, `CreateFile` and `SetFilePointer`, Android NDK notes on `off_t` and `lseek64`.
