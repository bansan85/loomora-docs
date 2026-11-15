# Procedure: Web (Legacy page, JavaScript path)

Replayable procedure for the platform that was really run in this project (FR-016, SC-004, SC-008):

| Platform record | Variant | Path | Verdict | Where it is run |
|---|---|---|---|---|
| `web-firefox-10-windows-2000-x86` | legacy | `js` | `validation/verdicts/web-firefox-10-windows-2000-x86-legacy.json` | Firefox 10.0 in the VirtualBox VM "Windows 2000" |
| `web-firefox-10-windows-2000-x86` | modern | `js` | `validation/verdicts/web-firefox-10-windows-2000-x86-modern.json` (`not_tested`) | not written yet |
| `web-chrome-157-windows-11-x64` | modern | `wasm` | `validation/verdicts/web-chrome-157-windows-11-x64-modern.json` | Chrome 157.0.8103.51, headless, on the Windows 11 x64 host (last section below) |
| `web-chrome-157-windows-11-x64` | legacy | `wasm` | `validation/verdicts/web-chrome-157-windows-11-x64-legacy.json` (`not_tested`) | not run on purpose: Web Legacy is validated on Firefox 10 |

Every command is the one that was run, taken from `tools/` and the records.

## Status of this document

- Written and run on 2026-11-15. The WebAssembly path was built and run under node and in a current headless Chrome (task T085, spike S5), but has **no verdict and no platform record** in this project: the PoC rule is that success on the lowest testable browser stands for newer ones, and the WebAssembly path is only smoke-tested.
- The browser is Firefox 10.0 (build ID 20120129021758) on Windows 2000 build 2195 without service pack. Internet Explorer up to 9 is dropped by the owner (no typed arrays). No other browser or operating system was run for the JavaScript path.
- Safari (Legacy page, JavaScript path) is considered covered by inference from Firefox 10 Legacy (PoC scope rule, owner decision 2026-11-15). It was never run: there is no Safari platform record and no Safari verdict, and nothing here says it works.

## Prerequisites

- WSL Ubuntu 24.04 with the current emsdk (`~/emsdk`, WebAssembly path) and the Emscripten 1.39.20 fastcomp one (`~/emsdk-fastcomp`, JavaScript path), see spike S5 and `tools/build.py`. Builds go in `$HOME/loomora-build/` (set `LOOMORA_BUILD_DIR` to give each checkout its own folder).
- The VirtualBox VM "Windows 2000" (`validation/platforms/windows-2000-x86.json`), **running**, with the account `name` logged on and Firefox 10.0 installed in `C:\Program Files\Mozilla Firefox\`. The launcher never starts, stops, snapshots or restores the VM. On 2026-11-15 the VM was found paused by the host power management (`VBoxManage controlvm resume` answers "VM is paused due to host power management"); a `savestate` made the VM process abort, so the owner's snapshot "Claude Firefox 10" was restored and the VM started again. If that happens again, resume it from the VirtualBox window (or restore the snapshot) before the run.
- The guest reaches the host at `10.0.2.2` (VirtualBox NAT). Settings are the environment variables of `tools/run_vectors.py`: `LOOMORA_VM_NAME`, `LOOMORA_VM_USER` (default `name`), `LOOMORA_VM_PASSWORD` (default empty), `LOOMORA_VM_DIR`, `LOOMORA_VBOXMANAGE`, and `LOOMORA_WEB_DIST` (the dist folder as seen from Windows, for example `//wsl.localhost/Ubuntu-24.04/home/<user>/loomora-build/web-vm/web-legacy/dist`).

## Build

```
wsl -d Ubuntu-24.04 --cd <repo> -- bash -lc 'LOOMORA_BUILD_DIR=$HOME/loomora-build/web-vm python3 tools/build.py web legacy'
```

Builds the WebAssembly module and the JavaScript build, runs both self-tests under node, checks that `app.js` and `loomora.js` parse as ECMAScript 3 (`tools/trials/web-node/es-level.js`), assembles `dist/` (`index.html`, `app.js`, `loomora.js`, `loomora-wasm.js`, `loomora.wasm`) and writes `validation/deliverables/web-firefox-10-windows-2000-x86-legacy.json` (the platform record must exist). Expected (2026-11-15): `app.js` and `loomora.js` "ecmaVersion 3", `loomora.js` 91,529 bytes, `app.js` 16,143 bytes.

## Without a browser (node)

```
python tools/run_vectors.py web-node-js legacy --exe <dist as seen by WSL>
python tools/run_vectors.py web-node-wasm legacy --exe <dist as seen by WSL>
```

15 of 15 each. This is the page script in a DOM stub, not a browser: the stub does not sanitise the field value, which is why the line feed of `control_characters` survives there.

## Firefox 10 in the VM (the verdict)

```
python tools/run_vectors.py web-firefox-10-windows-2000-x86 legacy --timeout 120 --evidence-dir <folder>
```

`WebFirefoxVmLauncher` (in `tools/run_vectors.py`) does the following, serially:

1. Checks the VM state, `firefox.exe` and the dist files; creates the guest folder (`C:\Documents and Settings\name\loomora-web` by default), a new Firefox profile in it (`ff-profile\user.js`, only the preferences below), the helper `ffprocs.exe` (`tools/guest/ffprocs.c`, built in WSL by `tools/guest/build_ffprocs.sh`: count and terminate `firefox.exe` with the tool help library of kernel32; WMI answers "unspecified error" in this guest, Windows 2000 has no `tasklist` or `taskkill`) and the `.bat` files (the old Guest Additions pass no argument to a program).
2. Starts an HTTP server on every address of the host (any free port, `Cache-Control: no-store`, every request logged with its user agent).
3. For each valid vector: writes `fflaunch.bat` (`start "" firefox.exe -no-remote -profile <profile> "<url>"`, started with `VBoxManage guestcontrol start`, not `run`, because firefox.exe inherits the output of the bat file) with the URL `http://10.0.2.2:<port>/index.html?probe=1&input=<percent-encoded text>&platform=web-firefox-10-windows-2000-x86&post=1&run=<n>` (above 1,500 characters the URL goes through a redirect of the server, `/v/<n>`, because `cmd.exe` cuts a line at 8,191 characters), waits for the POST of the page, closes the window with Alt+F4 (`VBoxManage controlvm keyboardputscancode`), counts the `firefox.exe` processes until none is left, and checks the record with `tools/check_probe.py` and the vector.
4. Writes `run-vectors-<platform>-legacy-log.txt`, `-probe-lines.tsv` and `-server-requests.txt` in the evidence folder.

Preferences of the profile (`FIREFOX_PROFILE_PREFS`): `browser.shell.checkDefaultBrowser=false`, `browser.startup.homepage="about:blank"`, `browser.startup.page=0`, `startup.homepage_welcome_url=""`, `browser.startup.homepage_override.mstone="ignore"`, `browser.rights.3.shown=true`, `browser.sessionstore.resume_from_crash=false`, `browser.tabs.warnOnClose=false`, `browser.warnOnQuit=false`, `toolkit.telemetry.prompted=2`, `toolkit.telemetry.rejected=true`, `app.update.enabled=false`, `app.update.auto=false`, `extensions.update.enabled=false`, `extensions.blocklist.enabled=false`, `browser.search.update=false`. Nothing touches JavaScript, the cache or the network.

Expected (2026-11-15): 15 vectors run, 14 passed, 1 failed, 16 skipped, 2 truncated to 1,000 UTF-16 units. The failing vector is `control_characters`: an HTML `<input type="text">` removes the line feed from its value (a current Chrome does the same), so `input` is `a<tab>bc<U+0001>`; the rest of the record is correct. Every record has `core` `cpp-js`, the request log shows `index.html`, `app.js` and `loomora.js` and never `loomora-wasm.js` or `loomora.wasm`, and no `firefox.exe` is left after any run. `display_ms` 124 to 143, 704 to 843 ms from the start of Firefox to the posted record.

## Real typing and the screenshot

```
python tools/trials/web-firefox-10-windows-2000-x86/typing-check.py <folder>
```

A probe run with `input=Marie`, then the page without parameters: screenshot of the default greeting "Bonjour Coucou", Tab, Ctrl+A, "Marie" from scancodes (the guest layout is French AZERTY, see `validation/platforms/windows-2000-x86.json`), screenshot "Bonjour Marie", Alt+F4, process count. The screenshots are taken through `VBoxManage controlvm screenshotpng`; only one of them is kept as evidence (`screenshot-firefox10-typed-marie.png`).

## Notes

- `post=1` in the URL makes the page POST its probe record (`/probe`, or `/probe-error` with the message when the page fails) to the server that served it; it exists because Firefox 10 has no headless mode and no DOM dump. Without it the page behaves as before.
- Firefox 10 has no `performance.memory`: `peak_rss_kb` is `null`.

# Web Modern in Chrome on the Windows 11 host (WebAssembly path)

Run on 2026-11-15 (task T063). PoC rule (owner decision): the Modern UI is validated on ONE system and ONE current browser, here the Chrome installed on the Windows 11 x64 host of `validation/platforms/windows-11-x64.json`; Firefox and Safari (recent) are not required and were not run. The Legacy page is not run here (`not_tested` verdict): Chrome does not run on Windows 2000 and Firefox 10 alone covers Web Legacy.

## Prerequisites

- Chrome (stable) in `C:\Program Files\Google\Chrome\Application\` (or `LOOMORA_CHROME` = a `chrome.exe`); 157.0.8103.51 on 2026-11-15. The launcher takes the version from the version folder next to `chrome.exe` (`chrome.exe --version` prints nothing on Windows) and refuses to run when its major differs from the one in the platform id.
- WSL Ubuntu-24.04 with the current emsdk (`~/emsdk`, Emscripten 7.0.0), node and npm (TypeScript 7.0.2 comes from `package-lock.json`).
- Nothing touches the owner's Chrome profile: every run uses a new `--user-data-dir` in a folder of the system temp folder, deleted by the launcher. The launcher never closes chrome.exe by name: it only counts and terminates (`taskkill /PID`) the processes whose command line names the run's own profile folder.

## Build

```
wsl -d Ubuntu-24.04 --cd <repo> -- bash -lc 'LOOMORA_BUILD_DIR=$HOME/loomora-build/web-chrome python3 tools/build.py web modern'
```

Builds the WebAssembly module and its self-test, compiles `ui/modern/web-ts` with `tsc` in a copy (`npm ci`), assembles `dist/` (`index.html`, `main.js`, `native.js`, `loomora.js`, `loomora.wasm`; 3,384 + 6,146 + 6,215 + 14,152 + 23,971 bytes on 2026-11-15, identical sha256 in two builds) and, because `validation/platforms/web-chrome-157-windows-11-x64.json` exists, writes `validation/deliverables/web-chrome-157-windows-11-x64-modern.json`. Then `python tools/measure_sharing.py --write --platform web-chrome-157-windows-11-x64` stores the line counts (493 shared, 0 glue, 420 UI, 54.0%, below 80% like the Legacy page: a web page is almost all UI).

## Vectors in Chrome (the verdict)

```
$env:LOOMORA_WEB_DIST = "\\wsl.localhost\Ubuntu-24.04\home\<user>\loomora-build\web-chrome\web-modern\dist"
python tools/run_vectors.py web-chrome-157-windows-11-x64 modern --timeout 60 --evidence-dir <folder>
```

`WebChromeHostLauncher` (in `tools/run_vectors.py`) serves the dist folder over HTTP on `127.0.0.1` (a free port, `Cache-Control: no-store`, every request logged) and, for each valid vector, serially, starts

```
chrome.exe --headless=new --disable-gpu --no-first-run --no-default-browser-check --disable-extensions --disable-component-update --disable-background-networking --disable-sync --metrics-recording-only --dump-dom --virtual-time-budget=15000 --user-data-dir=<temp>\runN "http://127.0.0.1:<port>/index.html?probe=1&input=<percent-encoded text>&platform=web-chrome-157-windows-11-x64"
```

It takes the DOM Chrome dumps on standard output, requires the title `LOOMORA_PROBE_DONE`, reads the probe record from `<pre id="probe">`, then counts (and terminates) the chrome.exe processes that still name the run's profile. The record goes through the same `check_probe.py` and vector checks as every other launcher (`message` = "Bonjour " + input, `platform_id`, `ui_variant` modern, `core` cpp-wasm). A text longer than 1,000 UTF-16 units must come back cut to its first 1,000.

Expected (2026-11-15): 15 vectors run, 15 passed, 16 skipped, 2 truncated to 1,000 UTF-16 units, about 24 s; `core` cpp-wasm in every record; the request log has one `loomora.wasm` request per run; no chrome.exe left. `control_characters` passes (the Modern page uses a `<textarea>`, which keeps the line feed; the Legacy `<input type="text">` on Firefox 10 does not). `display_ms` 74 to 110: `--virtual-time-budget` virtualises the page clock, so it is not a wall-clock figure; chrome.exe start to exit 0.9 to 1.2 s.

Without Chrome, the launcher is covered by `python tools/run_vectors.py --self-test` (platform id rules, the DOM parser); the page itself also runs under node: `python tools/run_vectors.py web-node-wasm modern --exe <dist as seen by WSL>`.

## Default series, typing and colour scheme

```
python tools/trials/web-chrome-157-windows-11-x64/default-series.py
node tools/trials/web-chrome-157-windows-11-x64/typing-check.mjs "C:\Program Files\Google\Chrome\Application\chrome.exe" <dist folder>
```

`default-series.py` runs the default text ten times in new profiles (cold start) and prints the probe figures and the wall time of chrome.exe: 'Bonjour Coucou' every time, `display_ms` 77 to 237 (median 90), start to exit 891 to 1,113 ms (median 981.5).

`typing-check.mjs` serves the dist folder, starts Chrome with `--remote-debugging-port=0` (the port is read from `DevToolsActivePort` of the throw-away profile) and drives it through the DevTools protocol with node's own WebSocket: it opens the page without probe mode (greeting 'Bonjour Coucou'), focuses and selects the textarea, types 'Marie' with `Input.dispatchKeyEvent` (greeting 'Bonjour Marie'), reads the computed colours with `Emulation.setEmulatedMedia` prefers-color-scheme dark and light and computes the contrast, closes the browser with `Browser.close` and counts the leftover processes. It is not a person at a keyboard (see the restrictions of the verdict). Run it from PowerShell: Git Bash rewrites the `\\wsl.localhost\...` path of the dist folder.

## Screenshots

Two screenshots only, with `chrome.exe --headless=new --disable-gpu --window-size=800,420 --virtual-time-budget=15000 --user-data-dir=<temp> --screenshot=<file> [--force-dark-mode] http://127.0.0.1:<port>/index.html` (any static server on `127.0.0.1` for the dist folder, for example `python -m http.server <port> --bind 127.0.0.1 --directory <dist>`, stopped by PID afterwards): the light scheme (default) and `--force-dark-mode`. In this headless Chrome `--blink-settings=preferredColorScheme=1` did not switch the scheme (`evidence/web-chrome/web-chrome-157-windows-11-x64/screenshots-log.txt`).

## Notes

- Everything above was observed on 2026-11-15; the restrictions of `web-chrome-157-windows-11-x64-modern.json` list what was not (other browsers, headed mode, a person typing, forced colours, DPI, file://, HTTPS).
- Chrome's user agent hides the real version of Windows (`Windows NT 10.0` on Windows 11) and the minor parts of the Chrome version (`157.0.0.0`): the page cannot read either, which is why it asks the backend about family Web, version 0.0.
