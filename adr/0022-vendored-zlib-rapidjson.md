# ADR 0022: zlib and RapidJSON as pinned git submodules

(The file name keeps the word "vendored" of the first draft, which copied the sources into the repository; spikes and links refer to it. The decision is now two pinned git submodules, not copies.)

- **Status**: accepted (2026-11-17; the Android NDK and Emscripten runs of spike S-F1 were made later, 2026-11-17, and recorded 2026-11-18; the Windows 2000 VM run, the ARM ABIs and any browser run were not made, see Results of the spikes)
- **Date**: 2026-11-17
- **Spike / requirement**: S-F1, S-F2 / FR-003, FR-004 (spec 002)

## Context

The `.loomora` format needs deflate and inflate (FR-003), and a JSON parser and writer whose text for a double depends only on the double (FR-004, SC-001, SC-002). The platforms print floating-point numbers differently: `msvcrt` on Windows 2000, glibc, bionic, Emscripten. zlib's deflate is portable C and gives the same bytes for the same input and parameters on every architecture and byte order **for one zlib version**; the system zlib differs by platform and version, and some distributions patch it. The product build must not need a system package that an old floor lacks: both libraries come from the repository itself, pinned to one exact upstream version.

## Decision

- **zlib**, included as the **git submodule** `third_party/zlib` (upstream `https://github.com/madler/zlib.git`), **pinned to the tag v1.3.1** (commit 51b7f2a), not copied into the repository. The project builds only the zlib sources it needs (`adler32`, `crc32`, `deflate`, `inflate`, `inffast`, `inftrees`, `trees`, `zutil`) through `cmake/loomora_zlib.cmake`, as a separate C target `loomora_zlib` (so it is outside the C++11 target rule), linked into `loomora` and `loomora_shared`; the other zlib sources (gzip, minigzip, contrib, examples) are never compiled. A zlib upgrade is a change of the pinned commit in a visible commit. The writer calls `deflateInit2` with fixed parameters and writes through a fixed-size output buffer so chunking cannot change the stream. CRC-32 also comes from zlib.
- **RapidJSON** (MIT, headers only), included as the **git submodule** `third_party/rapidjson` (upstream `https://github.com/Tencent/rapidjson.git`), **pinned to the commit 297637bc0a0f00dc504dbc1a47ebdaa577f8ae19**, not copied into the repository; its `include` directory is used as a SYSTEM directory so the warning flags of the backend do not apply to it. Parsing uses `kParseFullPrecisionFlag` (correctly rounded decimal to double) and `kParseIterativeFlag` (no recursion), a depth limit of 64 and no NaN or Infinity. Small entries are parsed into a DOM; `nails.json` and other large arrays are read with the SAX reader. Writing uses the compact `Writer`, keys in the fixed order of the contract. Its own integer routines (Grisu2 for output) make the text independent of the C library and the locale.
- **Submodules, not copies**: both libraries are fetched with `git submodule update --init third_party/zlib third_party/rapidjson` (with Catch2, `third_party/catch2`); the exact bytes are those of the pinned upstream commit, so there is no local change to track and no copy to keep in step. The rationale is unchanged: one pinned source, deterministic output, no system library.
- **Fallback, per platform only**: a toolchain that cannot build the zlib C sources writes stored-only archives on that target (declared in `validation/fallbacks.json`), never everywhere. No platform takes it: spike S-F1 found no toolchain that failed (see Results of the spikes), so the fallback stays declared in the decision and unused. The identity of the double text on i686 was checked by spike S-F2 (ADR 0021): it holds, once the test fixtures are on SSE2.

## Results of the spikes

**S-F1 (confirmed for every toolchain the question names, 2026-11-17, completed with the Android and Web runs recorded 2026-11-18).** `core/tests/spike_fmt_deps.cpp` (target `loomora_spike_fmt_deps`) deflates and inflates a 1 MiB buffer (levels 6 and 1) through the pinned zlib 1.3.1 and writes and parses 2000 doubles through RapidJSON with `kParseFullPrecisionFlag | kParseIterativeFlag` (every double compared bit for bit, the written text compared). It prints `S-F1: ok` and the same three hash lines with Windows 11 x64 clang (Visual Studio 18 LLVM) + Ninja, with g++ 15.2 `-std=c++11 -pedantic-errors -Wall -Wextra -O2` on Ubuntu 26.04 (WSL), and with MinGW-w64 GCC 13-win32 and `mingw-w64-win2000-i686.cmake` (cross-built in WSL, run as a 32-bit process on Windows 11, not on a Windows 2000 system), and, in the later runs below, with the Android NDK r15c and Emscripten:

```
zlib 1.3.1 level 6: in=1048576 deflated=151981 crc32=0f7970e9 sha256(deflated)=0f01cbcd35f279f6da31a279aec6240e527db69bae5d0faf4fa1f90dee69355f
zlib 1.3.1 level 1: in=1048576 deflated=207271 crc32=0f7970e9 sha256(deflated)=2d7f429daec9d478b1346e469df04ceb771efdd1010ff5976f4470e921ff3f44
rapidjson: 2000 doubles, 45684 bytes, sha256(json)=9dff655478f4534e34de87a66054c3715d3419bc5ff9f5b4dee1fee1244d8d72
```

So zlib and RapidJSON need no fallback on these toolchains, and a toolchain that prints these three lines produces the same deflate bytes and the same JSON text. The six reference files written through them are byte-identical on all the targets of ADR 0021 (S-F2).

**Android and Web runs (2026-11-17, recorded 2026-11-18).** The program printed `S-F1: ok` and, as the result files state, the three lines of `spike_results_sf.md` with:

- **NDK r15c** (15.2.4203891, Clang 5.0.300080, gnustl, `android-ndk-r15c.cmake`): x86 built for API 9 and run on an Android 4.1.2 (API 16) x86 emulator; x86 (API 16 sysroot, PIE) run as a 32-bit process on Android 7.0 (`sizeof(void*)=4`); x86_64 (API 21 sysroot, PIE) run on Android 7.0 and on Android 15 (`sizeof(void*)=8`, a system with no platform record). The armeabi, armeabi-v7a and arm64-v8a builds link and were ELF-checked, never run.
- **Emscripten** under node: 1.39.20 fastcomp (`emscripten-js-fastcomp.cmake`, asm.js, node 14.18.2), 7.0.0 WebAssembly (`emscripten-wasm.cmake`, node 24) and 7.0.0 wasm2js (`emscripten-js.cmake`, node 24), `sizeof(void*)=4 sizeof(long)=4`. Linking the program needed its own export list (`bridge/wasm/exports-main.txt`, only `_main`) because it links only zlib and `wasm-ld` rejected the `loomora_*` list of the toolchain: a build detail of the test program, not of the libraries.

The raw output of these runs was not stored (ADR 0014: the same three lines as above). No fallback of this ADR is taken on any of them. The NDK r15c zip is not covered by a published checksum (none was found); its measured SHA-256 is in `validation/spikes.json`.

**Not run (`not_tested`)**: the program on the Windows 2000 VM (not started); the ARM ABIs of the NDK (no ARM system); Android 2.3 to 4.0 and the API levels not listed; the Emscripten builds in any browser (node only); Emscripten 1.29.12 plain arrays (dropped by ADR 0017); its MinGW-w64 x86_64 build was `blocked_by_antivirus`. The status `confirmed` in `validation/spikes.json` means that every toolchain named by the question (GCC, Clang, MinGW-w64 for Windows 2000 i686, NDK r15c, Emscripten 1.39.20 fastcomp) built and ran the program at least once and gave the same output; it does not mean that it ran on every floor system. The raw lines of the first three toolchains are in `validation/evidence/adr-0021/windows-11-x64/s-f1-fmt-deps-output.txt`.

**Other results tied to this record.** `libloomora` with these two libraries survived 12.6 million fuzz inputs under ASan and UBSan (SC-004) and opens a 41 MB project with 10,000,000 steps in about 1 s (SC-005); the references are byte-identical across the targets run (ADR 0021, `validation/evidence/adr-0021/`).

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

- Two third-party sources are pinned as git submodules (zlib v1.3.1, RapidJSON 297637bc), each with its own licence in its tree; a clone needs `git submodule update --init` for them, and they are not compiled under `-pedantic-errors`.
- A later zlib upgrade (a new pinned commit) changes the compressed bytes: the reference hashes (self-test constants, `validation/evidence/adr-0021/`) change with it, in a visible commit. Stored entries are unaffected.
- This is the deviation "two third-party libraries" (named "two vendored libraries" in the Complexity Tracking of spec 002); the simpler routes (own code, or system libraries) cost more code and risk, or lose byte identity.
- The build and run results on the Android NDK r15c (x86 and x86_64 emulators) and Emscripten (node) toolchains are known (S-F1 above); on the Windows 2000 VM, the ARM ABIs and in browsers they are not (`not_tested`).

## Sources

Documents read for the draft (2026-11-17, no external source was fetched):

- `specs/002-loomora-file-format/research.md`: R2 to R6 (R3 and R4 in particular), R15
- `specs/002-loomora-file-format/plan.md`: Technical Context, Complexity Tracking, delivery phase 0
- `specs/002-loomora-file-format/tasks.md`: T002, T003, T009

Documents read for the acceptance (2026-11-17):

- `core/tests/cross_arch_results.json` and `cross_arch_results.md` of the main repository (branch `feat/loomora-format`): per target built / ran / hashes, fallbacks needed, import checks, `blocked_by_antivirus`, `not_tested`
- `core/tests/spike_results_sf.md`: S-F1 and S-F3 results, including the raw output of the Windows 11 run (the probe source `tools/guest/fileio_probe.c` itself was not re-read for this record)
- `core/tests/android_results.md` and `android_results.json`, `core/tests/web_results.md` and `web_results.json` (added 2026-11-18, branches `feat/fmt-android` and `feat/fmt-web` of the main repository): the S-F1 outcome per Android and Web target; the raw output of those runs was not available as a file and the three lines are quoted from the result files, not re-read from a run
- `core/tests/fuzz_run_sc004.md` (SC-004 fuzz run) and `core/tests/perf_sc005.md` (SC-005 numbers)
- the raw reference files of each target (`$HOME/loomora-build/xtarget-refs/<target>/` in WSL), re-hashed on 2026-11-17 and compared with `python tools/check_loomora.py --compare`
- **Not read** (listed by `research.md`, no external document was fetched or read for this record; the results above rest on measurements, not on these texts): the zlib manual and FAQ, the RapidJSON documentation (parse flags, `dtoa`), the licences of both libraries. A claim in this record that comes from one of them is the claim of `research.md`, not checked here.
