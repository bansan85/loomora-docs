# C++ Bug Review - core/src/format/ and core/src/c_api_project*.cpp (T101)

- **Date**: 2026-11-17
- **Repo commit**: 6076136 (feat/loomora-format, reviewed state; the fixes are on feat/fmt-review)
- **Scope**: folder `core/src/format/` (all of it) and `core/src/c_api_project.cpp`, `c_api_project_read.cpp`, `c_api_project_edit.cpp`, `c_api_project_export.cpp` and their headers
- **Comparison base**: none (folder review)
- **Quality tooling detected**: strict warnings (`-Wall -Wextra -Werror`, `-pedantic-errors` in the `std` check of `tools/verify.py`); no clang-tidy, no sanitizer in the quick verification. Everything else analysed broadly.
- **Files covered**: read in full: the four C API files, `zip/zip_reader.cpp`, `zip/zip_writer.cpp`, `project_reader.cpp`, `project_writer.cpp`, `project_editor.cpp`, `project_export.cpp`, `io/file_posix.cpp`, `io/file_win32.cpp`, `io/memory_io.*`, `codec/*.cpp`, `json/json_in.*`, `json/json_out.cpp`, `report.cpp`, `limits.*`, `sha256.cpp`, `endian.hpp`; read in part: `validate/validator.cpp`, `validate/geometry.cpp`, `model/common.cpp`, `model/layout.cpp`, `model/project.cpp`, `model/model_json.cpp`, `item_json.cpp`
- **Method note**: single-agent review (no fan-out), run on Sonnet; the reading was by hand, guided by the skill's grid.
- **Count by impact**: Bad behavior without a crash 3 (1 confirmed and fixed, 1 confirmed and fixed without test, 1 deferred) - Omissions 2 (deferred) - No user-visible impact 0 - Improvement 3 (deferred); 5 candidates rejected as false positives (listed at the end)

**Analysis limitations**: `model/frame.cpp`, `model/result.cpp`, `model/image.cpp`, `model/spool.cpp` and most of `item_json.cpp` were skimmed, not read line by line. `utf8.cpp` and `errors.hpp` are outside the scope. Nothing was run under ASan, UBSan or TSan. The Windows 2000 code paths (`LOOMORA_FMT_WIN2000_SEEK`, `rename_aside`) were read only. The verification is `tools/verify.py quick` on Windows clang (baseline only) and on Linux g++ (WSL, with and without the fixes); `records` fails on both hosts because `fallbacks.json` is on the unmerged docs branch.

## 1. Opening a named pipe hangs the caller

**Impact**: Bad behavior without a crash
**Category**: Logic (API misuse)
**Triage**: CONFIRMED, fixed by commit `0d4b396` (fix: do not block opening a named pipe) with the test `core/tests/format/test_source_special_files.cpp` (fails before the fix: the call returned after 600 ms, when a helper process opened the pipe; passes after).

**Location**: `core/src/format/io/file_posix.cpp`, `open_file_source`: `::open(path, O_RDONLY | O_CLOEXEC)` followed by `fstat` and `S_ISREG`.

**Reasoning**:
- The code intends to refuse anything that is not a regular file [1].
- The check runs after the `open` call, and `open(O_RDONLY)` on a FIFO blocks until a writer opens the other end.
- So the check is never reached for a FIFO that nobody writes to.

**Trigger scenario**: `loomora validate some.fifo`, or any C API open function with a path naming a FIFO.

**Consequence**: the thread hangs forever (a UI freezes) instead of getting `LOOMORA_ERR_IO`.

**Suggested fix**: open with `O_NONBLOCK` (no effect on a regular file) so that the `S_ISREG` check refuses the pipe at once. Applied.

**References**:
1. core/src/format/io/file_posix.cpp:274-291 (reviewed state) - the open, then the regular-file check

## 2. Data race on the counter of the temporary file name (POSIX)

**Impact**: Bad behavior without a crash
**Category**: Concurrency
**Triage**: CONFIRMED, fixed by commit `8706c35` (fix: keep no shared state in the temp file name). No failing-first test: a data race on a counter has no deterministic symptom (O_EXCL keeps the names unique), it only shows under ThreadSanitizer. This deviates from the "test that fails first" rule on purpose; the owner may drop the commit.

**Location**: `core/src/format/io/file_posix.cpp`, `random_suffix`: `static std::uint32_t counter` incremented with `++counter`; the Windows twin uses `InterlockedIncrement`.

**Reasoning**:
- The contract says two handles are independent and may be used from two threads [1].
- Two editors saving at once both call `create_file_sink`, hence `random_suffix` [2].
- The unsynchronised increment of a static is a data race, which is undefined behaviour in C++11.

**Consequence**: formally undefined behaviour; in practice two threads may draw the same suffix, which O_EXCL turns into a retry. No corruption.

**Suggested fix**: no shared state: time in microseconds, process id, address of a local and the attempt number. Applied.

**References**:
1. specs/002-loomora-file-format/contracts/c-api-project.md:12 - thread safety rule
2. core/src/format/io/file_posix.cpp:119-136, 311-313 (reviewed state)

## 3. Saving through a symbolic link replaces the link

**Impact**: Bad behavior without a crash
**Category**: Logic
**Triage**: DEFERRED for the owner. The temporary file is renamed over the path, so when the path is a symlink the link itself is replaced by a regular file and the pointed file is left unchanged (POSIX `rename`; Windows `MoveFileEx` behaves the same for a link). Fixing it means `realpath` of the target before creating the temporary file (a behaviour choice: follow or not follow, and the temporary file must then sit next to the real file). Not in the contract; owner decision.

**References**:
1. core/src/format/io/file_posix.cpp:200-203 (commit) and 323-330 (sink creation)

## 4. Calendar validity of the time stamps is not checked

**Impact**: Omissions
**Category**: Logic
**Triage**: DEFERRED. `is_iso8601_utc` accepts `2026-02-31T00:00:00Z` (day 1 to 31 for every month, second up to 60). The data model says "ISO 8601 UTC, YYYY-MM-DDTHH:MM:SS[.fff]Z" without a calendar rule, and tightening the reader would reject files that open today; owner decision.

**References**:
1. core/src/format/model/common.cpp:46-65
2. specs/002-loomora-file-format/data-model.md:34

## 5. `inflate_some` truncates the length to 32 bits

**Impact**: Omissions
**Category**: UB (integer truncation)
**Triage**: DEFERRED (confirmed by reading, not testable). `zs.avail_out = static_cast<uInt>(len)` wraps for a request of exactly a multiple of 4 GiB on a 64-bit host (a 4 GiB buffer given to `loomora_project_read_entry` on a deflated entry larger than 4 GiB): `avail_out` becomes 0, the call returns 0 bytes and reports "the entry is shorter than its declared size". Other lengths only produce a shorter read. A test needs a 4 GiB buffer and a 4 GiB entry. One-line fix when wanted: clamp `len` to 1 GiB in `inflate_some`, as the writer does with `kMaxZlibChunk`.

**References**:
1. core/src/format/zip/zip_reader.cpp:565-582
2. core/src/format/zip/zip_writer.cpp:21 - the clamp the writer already has

## 6. Linear costs on large projects

**Impact**: Improvement
**Category**: Logic (performance)
**Triage**: DEFERRED. (a) `ZipReader::find` is a linear scan and the reader calls it for each item (quadratic with many items). (b) `ProjectReader::read_nails` on a deflated `nails.json` restarts the inflate from byte 0 for each call, so reading a layout by chunks is quadratic. (c) `loomora_layout_nail_label` loads the whole nail table for one label. (d) `MeshReader` rebuilds its block table on every C API call. None is a correctness issue; a name index (`std::map`) in `ZipReader` is the cheap first step.

**References**:
1. core/src/format/zip/zip_reader.cpp:481-486
2. core/src/format/project_reader.cpp:276-307
3. core/src/c_api_project_read.cpp:288-307

## 7. Windows 2000 fallback may leave the `.old-` file

**Impact**: Improvement
**Category**: Ownership (resource)
**Triage**: DEFERRED. In `rename_aside`, if the final `DeleteFileW(aside)` fails (virus scanner), the `<target>.old-xxxxxxxx` file stays next to the project with no error. Only reached on systems without the POSIX rename (before Windows 10 1809) when a reader holds the target. Cannot be tested deterministically here.

**References**:
1. core/src/format/io/file_win32.cpp:393-405

## 8. Raw copy of kept entries is not re-verified at save time

**Impact**: Improvement
**Category**: Logic (TOCTOU)
**Triage**: DEFERRED. A save of an edited project copies the compressed bytes of kept entries straight from the old file and trusts the sizes and SHA-256 verified when the project was opened. If another process rewrites the old file in place between the open and the save (not by replacing it: the open descriptor or handle keeps the old inode), the new file can carry stale sums. The result fails its own validation when opened, so it is detected, not silent. A CRC check during the copy would close it at a cost of one pass.

**References**:
1. core/src/format/project_writer.cpp:300-311, 336-344
2. core/src/format/zip/zip_writer.cpp:417-442

## Rejected candidates (FALSE POSITIVE)

- **Nails outside the frame after a frame geometry or mesh change** (`replace_frame_json`, `replace_frame_mesh`, `add_layout` before the mesh): the editor does not recheck the layouts. This is the documented design: data-model.md row "Frame geometry changed ... a nail may now be outside: validation reports it".
- **Position of `EntrySource` after an exception** (`project_reader.cpp:150-175`): the cached stream and offset are not advanced when a read throws, but every caller drops the source after an exception.
- **Editor state after an out-of-memory in `push_back`** (`add_frame`, `add_image`, ...): the item is in its map but not in the order list; the next add of the same kind uses the same identifier (the counter was not moved), overwrites the entry and repairs the order. Self-healing.
- **`ProjectEditor::layout_nails` is `const` but fills a cache**: only the editor does it, and the editor handle is single-thread by contract; the shared read-only `loomora_project` has no mutable state (every `EntrySource` and `MeshReader` is per call).
- **`encode_mesh` on a 32-bit host with more than 2^32 triangles**: the size is truncated to `size_t`, but the caller cannot hold such an array there, and the writer then fails with an internal error instead of corrupting.

Checked and found correct (no finding): bounds of every size read from the ZIP structures (central directory, local headers, ZIP64, extents), the inflate and CRC accounting, the nail index and range read, the sequence and mesh codecs (varint, block table, ranges), PNG/JPEG/WebP metadata stripping, the JSON depth and duplicate-key checks, SHA-256 padding, the sink state machines (commit, abort, destructor, double close), and the exception boundary of every C API function.
