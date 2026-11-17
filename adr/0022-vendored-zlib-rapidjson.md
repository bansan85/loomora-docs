# ADR 0022: zlib and RapidJSON vendored in the repository

- **Status**: proposed
- **Date**: 2026-11-17
- **Spike / requirement**: S-F1, S-F2 / FR-003, FR-004 (spec 002)

## Context

The `.loomora` format needs deflate and inflate (FR-003), and a JSON parser and writer whose text for a double depends only on the double (FR-004, SC-001, SC-002). The platforms print floating-point numbers differently: `msvcrt` on Windows 2000, glibc, bionic, Emscripten. zlib's deflate is portable C and gives the same bytes for the same input and parameters on every architecture and byte order **for one zlib version**; the system zlib differs by platform and version, and some distributions patch it. The product build must not need a submodule or a system package that an old floor lacks.

## Decision

- **zlib 1.3.x**, the files needed for `deflate`, `inflate` and `crc32` only, in `third_party/zlib/`, with its licence and a `VENDORED.md` (version, source hash, files kept, local changes: none). It is a separate C target `loomora_zlib` (so it is outside the C++11 target rule), linked into `loomora` and `loomora_shared`. The writer calls `deflateInit2` with fixed parameters and writes through a fixed-size output buffer so chunking cannot change the stream. CRC-32 also comes from zlib.
- **RapidJSON** (MIT, headers only) in `third_party/rapidjson/`, with the same kind of `VENDORED.md`, included as a SYSTEM directory so the warning flags of the backend do not apply to it. Parsing uses `kParseFullPrecisionFlag` (correctly rounded decimal to double) and `kParseIterativeFlag` (no recursion), a depth limit of 64 and no NaN or Infinity. Small entries are parsed into a DOM; `nails.json` and other large arrays are read with the SAX reader. Writing uses the compact `Writer`, keys in the fixed order of the contract. Its own integer routines (Grisu2 for output) make the text independent of the C library and the locale.
- **Vendored under `third_party/`**, not as a submodule, so building the product never needs one.
- **Fallback, per platform only**: a toolchain that cannot build the zlib C sources writes stored-only archives on that target (declared in `validation/fallbacks.json`), never everywhere. **Pending spike S-F1** (zlib and RapidJSON build and round trip on GCC and Clang `-std=c++11 -pedantic-errors`, MinGW-w64 for Windows 2000 i686, NDK r15c, Emscripten 1.39.20 fastcomp): the per-platform results and any fallback are written here when the spike ends. **Pending spike S-F2** for the identity of the double text on i686.

## Rejected alternatives

- **The system zlib**: output differs by platform and version, and it is absent or old on some floors; it would break byte identity (FR-004).
- **minizip**: it wraps zlib in a file-oriented ZIP API with timestamps and extra fields (ADR 0021); only the zlib core is needed.
- **miniz**: it carries its own deflate, whose output is not zlib's and is less documented across versions.
- **nlohmann/json**: C++11, but number output went through `snprintf` in older versions (locale and C library dependent), and its DOM is heavy for 100,000 nails.
- **A handwritten JSON subset**: about 600 lines, and the correct double conversion would still be ours to write and prove.
- **cJSON**: C, no correct double round trip.
- **Writing deflate ourselves**: a large, risky algorithm.
- **A stored-only writer for everyone**: valid, but a typical `nails.json` is 10 to 20 times larger; kept only as the per-platform fallback above.

## Consequences

- Two third-party sources are maintained in the repository, each with its licence and `VENDORED.md`; they are not compiled under `-pedantic-errors`.
- A later zlib upgrade changes the compressed bytes: the reference hashes (self-test constants, `validation/evidence/adr-0021/`) change with it, in a visible commit. Stored entries are unaffected.
- This is the deviation "two vendored libraries" of the Complexity Tracking of spec 002; the simpler routes (own code, or system libraries) cost more code and risk, or lose byte identity.
- The build and run results on the lowest toolchains are not known yet (spike S-F1).

## Sources

Documents read to write this draft (no external source was fetched):

- `specs/002-loomora-file-format/research.md`: R2 to R6 (R3 and R4 in particular), R15
- `specs/002-loomora-file-format/plan.md`: Technical Context, Complexity Tracking, delivery phase 0
- `specs/002-loomora-file-format/tasks.md`: T002, T003, T009
- To read before acceptance (listed by `research.md`, not read yet): the zlib manual and FAQ, the RapidJSON documentation (parse flags, `dtoa`), the licences of both libraries.
