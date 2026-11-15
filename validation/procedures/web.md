# Procedure: Web (Legacy page, JavaScript path)

Replayable procedure for the platform that was really run in this project (FR-016, SC-004, SC-008):

| Platform record | Variant | Path | Verdict | Where it is run |
|---|---|---|---|---|
| `web-firefox-10-windows-2000-x86` | legacy | `js` | `validation/verdicts/web-firefox-10-windows-2000-x86-legacy.json` (`ok_restricted`, 15 of 15 vectors in the second series) | Firefox 10.0 in the VirtualBox VM "Windows 2000" |
| `web-firefox-10-windows-2000-x86` | modern | `js` | `validation/verdicts/web-firefox-10-windows-2000-x86-modern.json` (`not_tested`) | not written yet |

Every command is the one that was run, taken from `tools/` and the records.

## Status of this document

- Written and run on 2026-11-15, then re-run the same day (second series) after three changes: the field of the page is a `<textarea>`, the page shows the end-of-life notice, and the launcher starts `firefox.exe` once more when a start is lost. The WebAssembly path was built and run under node and in a current headless Chrome (task T085, spike S5), but has **no verdict and no platform record** in this project: by the owner's decision of 2026-11-15 Chrome does not run on Windows 2000, so Firefox 10.0 alone stands for the Web Legacy ladder under the PoC rule (success on the lowest testable browser stands for newer ones), and the WebAssembly path is exercised only by the node harness and that headless check, which are not verdicts.
- The browser is Firefox 10.0 (build ID 20120129021758) on Windows 2000 build 2195 without service pack. Internet Explorer up to 9 is dropped by the owner (no typed arrays). No other browser or operating system was run for the JavaScript path.

## Prerequisites

- WSL Ubuntu 24.04 with the current emsdk (`~/emsdk`, WebAssembly path) and the Emscripten 1.39.20 fastcomp one (`~/emsdk-fastcomp`, JavaScript path), see spike S5 and `tools/build.py`. Builds go in `$HOME/loomora-build/` (set `LOOMORA_BUILD_DIR` to give each checkout its own folder).
- The VirtualBox VM "Windows 2000" (`validation/platforms/windows-2000-x86.json`), **running**, with the account `name` logged on and Firefox 10.0 installed in `C:\Program Files\Mozilla Firefox\`. The launcher never starts, stops, snapshots or restores the VM. On 2026-11-15 the VM was found paused by the host power management (`VBoxManage controlvm resume` answers "VM is paused due to host power management"); a `savestate` made the VM process abort, so the owner's snapshot "Claude Firefox 10" was restored and the VM started again. If that happens again, resume it from the VirtualBox window (or restore the snapshot) before the run. The second series (same day) ran in the VM as it was, running, with no restore. Every `guestcontrol start` of the launcher leaves a guest session open and VirtualBox allows 32: after a few series (or a failed one) run `VBoxManage guestcontrol "Windows 2000" closesession --all` (it changes no snapshot and no state) before the next one.
- The guest reaches the host at `10.0.2.2` (VirtualBox NAT). Settings are the environment variables of `tools/run_vectors.py`: `LOOMORA_VM_NAME`, `LOOMORA_VM_USER` (default `name`), `LOOMORA_VM_PASSWORD` (default empty), `LOOMORA_VM_DIR`, `LOOMORA_VBOXMANAGE`, and `LOOMORA_WEB_DIST` (the dist folder as seen from Windows, for example `//wsl.localhost/Ubuntu-24.04/home/<user>/loomora-build/web-vm/web-legacy/dist`).

## Build

```
wsl -d Ubuntu-24.04 --cd <repo> -- bash -lc 'LOOMORA_BUILD_DIR=$HOME/loomora-build/web-vm python3 tools/build.py web legacy'
```

Builds the WebAssembly module and the JavaScript build, runs both self-tests under node, checks that `app.js` and `loomora.js` parse as ECMAScript 3 (`tools/trials/web-node/es-level.js`), assembles `dist/` (`index.html`, `app.js`, `loomora.js`, `loomora-wasm.js`, `loomora.wasm`) and writes `validation/deliverables/web-firefox-10-windows-2000-x86-legacy.json` (the platform record must exist). Expected (2026-11-15, textarea page): `app.js` and `loomora.js` "ecmaVersion 3", `loomora.js` 91,529 bytes, `app.js` 17,067 bytes, `index.html` 1,696 bytes. `python3 tools/measure_sharing.py --write --platform web-firefox-10-windows-2000-x86` then stores the sharing figures in the deliverable record (the build rewrites the record without them).

## Without a browser (node)

```
python tools/run_vectors.py web-node-js legacy --exe <dist as seen by WSL>
python tools/run_vectors.py web-node-wasm legacy --exe <dist as seen by WSL>
```

15 of 15 each. This is the page script in a DOM stub, not a browser. The field is a `<textarea>` and the stub gives it the new-line normalisation of a browser (`textarea.value` reads CRLF and a lone CR back as LF), so the line feed of `control_characters` is carried as in a browser; an `<input type="text">` would have removed it. `page-harness.js --ua <user agent>` sets `navigator.userAgent`: with `Mozilla/5.0 (Windows NT 5.0; rv:10.0) Gecko/20100101 Firefox/10.0` the page shows the end-of-life notice and the harness checks that `eol_warning_shown` of the record and the notice agree; with a `Windows NT 10.0` user agent or the default one there is no notice.

## End-of-life notice

The notice text and the thresholds are the backend's (`loomora_is_eol`, `loomora_eol_warning`, `core/src/eol.cpp`); the page only reads the system version. A browser cannot read it, but its user agent states `Windows NT <major>.<minor>` truthfully up to NT 6.3. `ui/legacy/web/app.js` (`systemVersion`) hands the family Windows and that version to `loomora_is_eol` when the major is below 10 (Windows 2000 to 8.1 are end of life by the backend rule: below NT 11.0), and the family Web with version 0.0, never end of life, otherwise: Windows 10 and 11 both say NT 10.0, and macOS, Linux and Android give no usable version. So the notice appears on Windows 2000 (this platform, `eol` true) and never on Windows 10 and later or on other systems, whatever the browser. No change of the C API (six exports).

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

Expected (2026-11-15, second series, textarea page): 15 vectors run, 15 passed, 0 failed, 16 skipped, 2 truncated to 1,000 UTF-16 units. `control_characters` passes with its line feed (in the first series, with an `<input type="text">`, it failed: that field removes the line feed from its value, as a current Chrome does too, so `input` was `a<tab>bc<U+0001>`; 14 of 15). Every record has `core` `cpp-js` and `eol_warning_shown` true (Windows 2000, see above), the request log shows `index.html`, `app.js` and `loomora.js` and never `loomora-wasm.js` or `loomora.wasm`, and no `firefox.exe` is left after any run. `display_ms` 127 to 230, 520 to 1415 ms from the start of Firefox to the posted record.

About one start of `firefox.exe` in fifteen is lost in this 168 MB guest (the page is never requested). The launcher waits `WEB_START_WAIT` (30 s) for the request of the page; without any it ends the `firefox.exe` processes, starts it once more and prints "(firefox.exe lost at the first start, started once more)" after the vector. Without this retry three attempts at the series each failed one vector with "no probe record posted in 120 s" (14 of 15); the stored series needed the retry once (`accented`).

## Real typing and the screenshot

```
python tools/trials/web-firefox-10-windows-2000-x86/typing-check.py <folder>
```

A probe run with `input=Marie`, then the page without parameters: screenshot of the default greeting "Bonjour Coucou", Tab, Ctrl+A, "Marie" from scancodes (the guest layout is French AZERTY, see `validation/platforms/windows-2000-x86.json`), screenshot "Bonjour Marie", Alt+F4, process count. The screenshots are taken through `VBoxManage controlvm screenshotpng`; only one of them is kept as evidence (`screenshot-firefox10-typed-marie.png`).

For the textarea page and the end-of-life notice (second series), `python tools/trials/web-firefox-10-windows-2000-x86/typing-check-textarea.py <folder>` takes exactly one screenshot after typing (`screenshot-firefox10-textarea-eol-notice.png`: a one-row text area, the greeting and the notice box). In a textarea Tab does not select the text and the driver's Ctrl+A is Ctrl+Q on the AZERTY layout, so "Marie" lands in front of "Coucou" (field and greeting "MarieCoucou" / "Bonjour MarieCoucou"): the live update of the greeting is what it shows. That screenshot is a candidate, left untracked in the evidence folder `validation/evidence/web-firefox-fixes/web-firefox-10-windows-2000-x86/`, until the owner authorises it.

## Notes

- `post=1` in the URL makes the page POST its probe record (`/probe`, or `/probe-error` with the message when the page fails) to the server that served it; it exists because Firefox 10 has no headless mode and no DOM dump. Without it the page behaves as before.
- Firefox 10 has no `performance.memory`: `peak_rss_kb` is `null`.
- The field of the page is `<textarea id="name" rows="1" wrap="off" maxlength="1000">`: a single-line `<input>` cannot carry a line feed, a textarea can, and a user can also type Enter in it (the greeting then holds the line feed).
