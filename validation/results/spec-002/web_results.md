# Web results (spec 002, T009 Web part, T060, T093)

Machine-readable version: `web_results.json`. Run on one Windows 11 x64 machine, build and node runs in WSL Ubuntu 26.04
(`$HOME/loomora-build/<preset>`), real Chrome 157.0.8103.39 on the Windows host for the T093 Modern page. The raw reference
files are kept outside the repository (`$HOME/loomora-build/xtarget-refs/<preset>/`).

## The three Web builds

| Preset | Toolchain | Built | Ran (node) | ctest | Six reference files | S-F1 spike |
|---|---|---|---|---|---|---|
| `web-wasm` | Emscripten 7.0.0, WebAssembly (`loomora.js` 72,147 B + `loomora.wasm` 696,734 B) | yes | yes, node 24 | selftest, format_selftest: pass | identical | `S-F1: ok`, same three lines |
| `web-js` | Emscripten 7.0.0, `-sWASM=0` (wasm2js, ES2021 glue, 1,598,550 B) | yes | yes, node 24 | pass | identical | `S-F1: ok` |
| `web-js-fastcomp` | Emscripten 1.39.20 fastcomp, asm.js, `--memory-init-file 0` (1,787,058 B) | yes | yes, node 14.18.2 | pass | identical | `S-F1: ok` |

The six files written by `loomora_refgen` under node have the SHA-256 of `cross_arch_results.md` on all three builds, with no
Web-specific switch (no determinism bug found):

| File | SHA-256 |
|---|---|
| basic | `fcdf97f0257076ba1c3ecae1867e57ebdc397440286a278f2cbe34eff8ce5f96` |
| empty | `8ca4dc6a2139539feb87f92b11cc0d5685fdff051fa044b692a7be5d984d7913` |
| escape40 | `b17920aabf9cc165a5c3b9290ba15f22e99903592426a8e55c96dc67ae570e2a` |
| multi | `8bcd3ae91876dcf6048c3a5dc689754a8eab90367faed373942b819a7c3f8607` |
| multi-light | `b32c8a390657d4f56ec9bd448e6c291e9781dfbb3b34b8164d049af67a7c4a8d` |
| one | `fc3fb5ebd929689bd7ee32da020b279ca81b3412c6c947eda8c9e7e3ddb846c5` |

The spike prints the three lines of `spike_results_sf.md` (`zlib 1.3.1 level 6 ... 0f01cbcd...`, `level 1 ... 2d7f429d...`,
`rapidjson ... 9dff6554...`), with `sizeof(void*)=4 sizeof(long)=4` as on the 32-bit MinGW run. This closes the Web row of
S-F1 (T009, old toolchain part): `emscripten-js-fastcomp.cmake` was run.

Fastcomp: it was not installed. The ADR (0017) names it, the network worked, so it was installed as the ADR says:
`~/emsdk-fastcomp` = emsdk tag 3.1.29, `./emsdk install 1.39.20-fastcomp` (emcc 1.39.20 `45d43807`, node 14.18.2). It runs under
Python 3.14 with only `SyntaxWarning` lines. `python3 tools/build.py web legacy` builds both Legacy modules and checks that
`app.js` and `loomora.js` parse as ECMAScript 3 (they do, with the T093 code in `app.js`).

## T093: the Web project modes (`?validate_project=` and `?roundtrip_project=`)

The code written without being built compiled and worked as it was: no defect found in `ui/legacy/web/app.js`,
`ui/modern/web-ts/src/*.ts` (TypeScript 7.0.2, no error) or `bridge/wasm/module.cpp` (`loomora_web_alloc`, `_free`, `_export`).
Each run is compared, line and exit status, with `loomora validate --quiet` of the g++ build of this worktree, by
`tools/run_vectors.py --validate-project / --roundtrip-project` (the same rows as the other UIs: references, the generated
corpus, a missing path, three usage rows).

| Launcher | Runs | Passed |
|---|---|---|
| `web-node-wasm legacy` (page-harness.js, DOM stub) | 242 | 242 |
| `web-node-js legacy` (no WebAssembly, BigInt, Promise: the fastcomp `loomora.js`) | 242 | 242 |
| `web-node-wasm modern` | 242 | 242 |
| `web-chrome-157-windows-11-x64 modern` (real Chrome, headless, over HTTP, `fetch`) | 51 | 51 |

Tooling added: `page-harness.js --files DIR` (XMLHttpRequest and fetch serve the bundled `.loomora`, `setAttribute` in the
stub, project result and exit status), `run_project()` for `WebNodeLauncher` and `WebChromeHostLauncher`, and
`WebProbeServer.files_dir`. The Chrome run used 6 references and 18 corpus files (valid, warning, invalid, bomb, encrypted);
the Node runs used the whole corpus (114 files).

## Code changes (this target only)

- `core/tests/CMakeLists.txt`: `loomora_refgen` is built for the Web presets (link flags `-sNODERAWFS=1 -sENVIRONMENT=node`
  and the exported `main`); `loomora_spike_fmt_deps` gets its own export list (`bridge/wasm/exports-main.txt`, only `_main`),
  because it links only zlib and `wasm-ld` rejected the `loomora_*` list. Nothing in `libloomora`, no `config.hpp` switch.
- Tooling only (above) for T093.

## not_tested

- The Legacy page's project modes in a real old browser (Firefox 10 on the Windows 2000 VM): not run, node and DOM stub only.
- The fastcomp `loomora.js` in a browser: node 14 only.
- Firefox, Safari, Edge for the Modern page: only Chrome 157 was run.
- Emscripten 1.29.12 plain arrays (IE 6 to 9): dropped by ADR 0017.
- No docs-repository record was written (a later task); the `tools/build.py web` deliverable records were not written because
  the docs submodule is not checked out in this worktree.

## verify.py quick

Run after the changes, as before (only `records` fails: `fallbacks.json` and the spike records are on the unmerged docs branch,
same four undeclared macros and S-F1..S-F3 as in `cross_arch_results.md`):

- WSL (`python3 tools/verify.py quick`): 14 passed, 1 failed (`records`).
- Windows 11, clang (`python tools/verify.py quick`): 11 passed, 1 failed (`records`), 2 skipped (windows modern not built, `std` needs g++).
