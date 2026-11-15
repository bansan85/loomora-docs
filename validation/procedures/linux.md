# Procedure: Linux (Modern UI with gtkmm, Legacy UI with Xlib)

Replayable procedure for the platform that was really run in this project (FR-016, SC-004, SC-008):

| Platform record | Variant | UI | Verdict |
|---|---|---|---|
| `linux-ubuntu-24.04-x86_64` | modern | C++11 with gtkmm-3.0 on GTK 3 (`ui/modern/linux-gtk3`, executable `loomora-gtk3`) | `validation/verdicts/linux-ubuntu-24.04-x86_64-modern.json` |
| `linux-ubuntu-24.04-x86_64` | legacy | C++11 with Xlib, text with Xft and fontconfig (`ui/legacy/linux-xlib`, executable `loomora-xlib`) | `validation/verdicts/linux-ubuntu-24.04-x86_64-legacy.json` |

Every command is the one that was run, taken from `tools/`, ADR 0006 and 0007, the platform records and the verdicts.

## Status of this document

- Written on 2026-11-14 and replayed the same day on the machine below: the builds, `tools/verify.py quick`, `tools/run_vectors.py` for both UIs, the probe runs (including the mixed Unicode string), the Xlib window run with `xlib-driver.c`, and the gtkmm window driver and probe series (`startup default` scenarios). The results quoted are those of that replay; the stored evidence is named where it is the reference.
- **One machine only**: Ubuntu 24.04.5 in WSL2 with WSLg on a Windows 11 host. No bare-metal install, no other distribution, no old distribution, no other X server or desktop, no kernel older than 6.18, no native 32-bit system, no real keyboard (the window runs inject synthetic key events). The verdicts say so as restrictions and this procedure says nothing more.
- Per ADR 0007 a Linux binary is **built on the machine that runs it**; there is no glibc or libstdc++ ceiling to check and no cross build. The Linux floor is therefore not determined by this procedure.
- The i686 build (`--arch i686`, platform record `linux-i686`) is built and run through the 32-bit multiarch libraries of the same machine; its Legacy verdict is `validation/verdicts/linux-i686-legacy.json` (written 2026-11-14 from a real run: build with `python3 tools/build.py linux legacy --arch i686`, then L.5, L.6 and L.7 on that executable, with `--exe` for the vectors; the probe line reports the host platform id) and its Modern variant is `not_tested` with a reason in `validation/verdicts/linux-i686-modern.json`.
- Not covered by anything here: Ubuntu 22.04 and the other targets of T059 (no record), GTK older than 3.24, a real GTK 3.10 system (the 3.10 floor is compiled with `LOOMORA_GTK_FLOOR=ON` but no gtkmm 3.10 system was available), the end-of-life banner on a real old kernel (it is simulated, see L.7).

## L.1 System

`validation/platforms/linux-ubuntu-24.04-x86_64.json`: Ubuntu 24.04.5 LTS (Noble Numbat), x86_64, glibc 2.39, in the WSL2 distribution **`Ubuntu-24.04`** (WSL 3.0.5.0, WSLg 1.0.79, Microsoft WSL2 kernel 6.18.58.1-microsoft-standard-WSL2) on a Windows 11 host (10.0.26300.9974, MSI GL63 8RD, Intel Core i5-8300H, 8 logical CPUs and 20003 MB visible in the VM). GTK 3.24.41, gtkmm-3.0 3.24.9 (glibmm 2.66.7, libsigc++ 2.12.1); optional GTK 4.14.5 with gtkmm-4.0 4.10.0. WSLg provides `DISPLAY=:0` (XWayland) and `WAYLAND_DISPLAY=wayland-0`; the shell has `LANG=C.UTF-8`. The GTK UI uses Wayland by default and X11 with `GDK_BACKEND=x11` (the window drivers set it). No window decorations: captures show the client area.

The way the distribution was installed (which image) is not recorded in the repository beyond its name and version. Record `cat /etc/os-release`, `uname -srm`, `ldd --version | head -1` and `pkg-config --modversion gtk+-3.0 gtkmm-3.0 xft fontconfig` of the machine you replay on; a different system is a new platform record, not a replay.

Fonts: the base image has only DejaVu, Ubuntu and MathJax fonts, on which CJK, Thai, Devanagari and most emoji show as boxes (measured, `evidence/scroll-long-greeting/linux-ubuntu-24.04-x86_64/glyphs-fonts-coverage.txt`). The trials ran with `sudo apt install fonts-noto-cjk fonts-noto-color-emoji fonts-noto-core` (429 fontconfig fonts on 2026-11-14). Which glyphs are drawn depends on the fonts: record `fc-list | wc -l`.

## L.2 Tools

Versions read on 2026-11-14 on that machine:

| Tool | Version |
|---|---|
| gcc / g++ | 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04.1) |
| cmake, ninja | 3.31.12, 1.11.1 |
| python3 | 3.12.3, with `python3-jsonschema` 4.10.3 and, for the GUI drivers, `python3-gi` 3.48.2 |
| pkg-config | 1.8.1 |
| libx11-dev, libxft-dev, libfontconfig-dev | 1.8.7, 2.3.6, 2.15.0 |
| libgtkmm-3.0-dev | 3.24.9 (pulls `libgtk-3-dev`) |
| optional: libgtkmm-4.0-dev, gcc-multilib, xvfb, libxtst6 | 4.10.0, 4:13.2.0-7ubuntu1, 2:21.1.12, 1.2.3 |

```
sudo apt install g++ cmake ninja-build pkg-config python3 python3-jsonschema \
    libgtkmm-3.0-dev libx11-dev libxft-dev libfontconfig-dev
sudo apt install python3-gi libxtst6          # window drivers
sudo apt install xvfb                         # tools/verify.py full (private display)
sudo apt install libgtkmm-4.0-dev             # optional GTK 4 build
sudo dpkg --add-architecture i386 && sudo apt update && \
    sudo apt install gcc-multilib g++-multilib libx11-dev:i386 libxft-dev:i386 libfontconfig-dev:i386   # i686 Xlib build
```

Without `libxft-dev` and `libfontconfig-dev` the Xlib UI is still built, with the core X11 font (fallback `xlib-core-font-glyphs`, see L.6); `tools/build.py` prints the packages to install. Without 32-bit Xft the i686 build takes that fallback too.

Rules (CLAUDE.md): any WSL build goes under `$HOME/loomora-build/<name>`, never in `/tmp` and never in the Windows tree; `tools/build.py` and `tools/verify.py` already do. A CMake build folder is tied to one source folder: when the repository is checked out more than once, give each checkout its own root with `LOOMORA_BUILD_DIR=$HOME/loomora-build/<name>` (read by `tools/build.py` and `tools/run_vectors.py`).

Get the code: `git -c protocol.file.allow=always submodule update --init` (records in `docs/implementation`; the product build never needs them). From Windows, every command below can be given as `wsl -d Ubuntu-24.04 --cd <repo as /mnt/<drive>/...> -- <command>`; `tools/build.py linux ...` and `tools/run_vectors.py` also work from the Windows side and re-run themselves in WSL.

## L.3 Build the two UIs

Inside WSL, from the repository root:

```
python3 tools/build.py linux modern
python3 tools/build.py linux legacy [--arch x86_64|i686|all]
```

Modern: CMake + Ninja, Release, `LOOMORA_BUILD_LINUX_MODERN=ON`, under `$HOME/loomora-build/linux-modern`; the backend as the static library `libloomora.a` compiled with `-std=c++11` and the UI sources (checked in `compile_commands.json`), libstdc++ and libgcc linked dynamically; the tool checks that the executable needs only the gtkmm / GTK family, the C++ runtime and the C library. Legacy: CMake on `ui/legacy/linux-xlib`, native build on this machine, `$HOME/loomora-build/linux-legacy-x86_64` (and `linux-legacy-i686`), libX11 plus, when pkg-config finds them, libXft and fontconfig. The default `--arch` is `all` (x86_64 and i686).

Expected (replayed 2026-11-14):

```
artifact: out/linux-modern/loomora-gtk3 (167192 bytes)
deliverable record: docs/implementation/validation/deliverables/linux-ubuntu-24.04-x86_64-modern.json
artifact: out/linux-legacy/x86_64/loomora-xlib (80216 bytes)
deliverable record: docs/implementation/validation/deliverables/linux-ubuntu-24.04-x86_64-legacy.json
```

and for the Legacy UI the note `linux legacy builds natively on this machine ... (ADR 0007)`. The sizes are those of the verdicts; the hash can change between builds, compare sizes and the library lists. Direct libraries (`objdump -p`): Modern `libgtkmm-3.0.so.1 libatkmm-1.6.so.1 libgiomm-2.4.so.1 libpangomm-1.4.so.1 libglibmm-2.4.so.1 libsigc-2.0.so.0 libstdc++.so.6 libgcc_s.so.1 libc.so.6`; Legacy with Xft `libX11.so.6 libXft.so.2 libfontconfig.so.1 libstdc++.so.6 libgcc_s.so.1 libc.so.6`. The i686 executable measured 73,348 bytes on 2026-11-14 with 32-bit Xft installed (67,388 bytes on the core-font path before that).

`tools/build.py` rewrites the deliverable records in the docs submodule (only the notes change): a replay that does not release a new build discards them with `git -C docs/implementation checkout -- validation/deliverables`.

Optional variants of the Modern UI, built into their own folders by `python3 tools/verify.py quick --variants`: the GTK 3.10 floor (`-DLOOMORA_GTK_FLOOR=ON`, 3.10 code path only, rejects any GTK/GDK symbol newer than 3.10; executable 167,120 bytes on 2026-11-14), GTK 4 (`-DLOOMORA_GTK_API=4`, `libgtkmm-4.0-dev`; 169,920 bytes) and the i686 Xlib build. They were run on 2026-11-14 (15 of 15 vectors, 0 warnings) and are described in the verdict notes; the GTK variants have no verdict of their own (the i686 Xlib build has one, for `linux-i686`).

## L.4 Quick check

```
python3 tools/verify.py quick
```

Builds core and both UIs into `$HOME/loomora-build/verify-<hash of the repository path>`, runs `ctest`, `tools/run_vectors.py` on each UI, `tools/check_std.py` and `tools/validate_records.py`. Expected (replayed 2026-11-14):

```
PASS build core + loomora-gtk3, loomora-xlib
PASS ctest 2/2
PASS run_vectors linux legacy 15/15
PASS run_vectors linux modern 15/15
PASS std
PASS records
quick: 6 passed, 0 failed
```

`ctest` runs the backend self-test and `loomora_tests` (the vectors of `core/tests/vectors.json`); `check_std.py` builds the backend with every GCC and Clang found with `-std=c++11 -pedantic-errors -Wall -Wextra -Werror` and fails on an exported symbol other than the `loomora_*` C functions. **Do not run `verify.py full`** to check that nothing broke: it is only for changes to graphics (it starts a private Xvfb display, takes one screenshot per UI and scenario, and runs the Xlib harness under ASan and UBSan); screenshots it writes are candidates, never copied into the docs submodule without the owner's authorization.

## L.5 Probe runs and the probe line

The UIs print the probe line on the standard output with `--probe` and exit by themselves after about 2 s. WSLg gives `DISPLAY=:0`; with `LOOMORA_BUILD_DIR` unset the executables are `$HOME/loomora-build/linux-modern/loomora-gtk3` and `$HOME/loomora-build/linux-legacy-x86_64/loomora-xlib`.

```
~/loomora-build/linux-legacy-x86_64/loomora-xlib --probe > ~/line.json
python3 tools/check_probe.py ~/line.json          # prints OK, exit 0
~/loomora-build/linux-modern/loomora-gtk3 --probe --probe-input Marie
```

Expected line for the Legacy UI (replayed 2026-11-14; `display_ms`, `peak_rss_kb`, `timestamp_utc` vary):

```
{"schema":1,"platform_id":"linux-ubuntu-24.04-x86_64","ui_variant":"legacy","input":"Coucou","message":"Bonjour Coucou","greeting_matches":true,"core":"cpp","compiler":"gcc 13.3.0","displayed":true,"display_ms":33,"peak_rss_kb":9184,"eol_warning_shown":false,"disabled_features":[],"timestamp_utc":"2026-11-14T17:03:04Z"}
```

For the Modern UI: `"ui_variant":"modern"`, `display_ms` 135 and `peak_rss_kb` 46164 in the replay (stored series of 2026-11-14: `peak_rss_kb` about 46,000), same fields otherwise, `disabled_features` empty, `eol_warning_shown` false. `platform_id` is derived from `/etc/os-release` and `uname` (`linux-<ID>-<VERSION_ID>-<machine>`).

Mixed Unicode string for the Xlib UI (the one of `validation/evidence/linux-build-on-target/linux-ubuntu-24.04-x86_64/xlib-xft-probe-mixed.txt`; the output file is any file under `$HOME`):

```
export DISPLAY=:0
S=$'Zoé €5 αβγ Привет 日本語 שלום مرحبا é \U0001F98A \U0001F468‍\U0001F469‍\U0001F467 x​y ͸'
~/loomora-build/linux-legacy-x86_64/loomora-xlib --probe --probe-input "$S" > ~/mixed.txt
python3 tools/check_probe.py ~/mixed.txt
```

Expected: `OK`, `input` equal to the stored one, `greeting_matches` true, `displayed` true and `disabled_features` `["missing-glyph"]` (U+0378 has no glyph in any font; the greeting is unchanged). Replayed 2026-11-14: identical `input` to the stored line.

Series of 10 + 10 runs with exit status, standard error, wall time, `ru_maxrss` and leftover-process checks (the Modern trial's script; works on any of the two executables):

```
python3 tools/trials/linux-ubuntu-24.04-x86_64/probe-series.py <exe> <outdir under $HOME> <suffix> 10
```

It writes `probe-default-run1-<suffix>.json`, `probe-marie-run1-<suffix>.json`, the two `.jsonl` series and `trial-measurements-<suffix>.json`; then `python3 tools/check_probe.py` on the first lines (`--input Marie` for the second series). Replayed with 3 runs on 2026-11-14.

## L.6 The reference vectors

```
python3 tools/run_vectors.py linux-ubuntu-24.04-x86_64 modern
python3 tools/run_vectors.py linux-ubuntu-24.04-x86_64 legacy
```

(from Windows, the same command without `wsl` re-runs itself in WSL; `--exe <path on the Linux side>` runs another executable). Expected (replayed 2026-11-14, both UIs):

```
summary: 15 vector(s) run, 15 passed, 0 failed, 16 skipped; 2 truncated to 1000 chars; ... s with 4 worker(s)
result: PASSED
```

The 16 skipped vectors are error cases that cannot be typed into a field; the 2 truncated ones are the 4,096-byte vectors, expected cut at their first 1,000 characters (code points on Linux).

Xlib core-font path (the fallback `xlib-core-font-glyphs`, declared in `validation/fallbacks.json` with an empty `platform_ids` on purpose: it is chosen per build): on a machine that has Xft, force it at run time with `LOOMORA_XLIB_NO_XFT=1 python3 tools/run_vectors.py ... legacy` (15 of 15; the mixed string then reports `["glyph-substitution"]` and draws `?` for characters the core font `fixed` lacks; under WSLg the core font is Latin-1 only), or at build time with `-DLOOMORA_XLIB_XFT=OFF`. `LOOMORA_XLIB_NO_XIM=1` runs the key path without an input method.

## L.7 Window runs

Both need a display; use WSLg (`DISPLAY=:0`). Other Loomora windows on the same X server confuse a driver that looks windows up by title: close them first (the Modern trial hit exactly this with the Legacy UI).

**Xlib UI** (`tools/trials/linux-ubuntu-24.04-x86_64/xlib-driver.c`: finds the window titled `Loomora` and sends commands; keys are synthetic `XSendEvent`s with a remapped spare keycode, not a physical keyboard):

```
B=$HOME/loomora-build/xlib-trial; mkdir -p $B
gcc -O2 -o $B/xlib-driver tools/trials/linux-ubuntu-24.04-x86_64/xlib-driver.c -lX11
DISPLAY=:0 ~/loomora-build/linux-legacy-x86_64/loomora-xlib &
sleep 1.5
DISPLAY=:0 $B/xlib-driver wait 500 key BackSpace key BackSpace key BackSpace key BackSpace key BackSpace key BackSpace type Marie wait 500 shot $B/marie.ppm close
wait $!; echo "exit $?"
python3 tools/trials/linux-ubuntu-24.04-x86_64/ppm2png.py $B/marie.ppm
```

Expected (replayed 2026-11-14): `typed 5 characters`, `shot 480x320 -> ...`, `close sent`, UI exit status 0, no `loomora-xlib` process left; the picture shows `Marie` in the field and `Bonjour Marie` in the label (without the six BackSpace the typed text is appended: `CoucouMarie`). Other commands: `sym <hex>` (a keysym such as `0x7e1`), `size <w> <h>`, `wait <ms>`. The window is 480x320 for the Xlib UI.

**GTK UI** (`tools/trials/linux-ubuntu-24.04-x86_64/gtkmm-gui-driver.py`; real X11 key events through XTEST, the window found by title and `_NET_WM_PID`, closed with `WM_DELETE_WINDOW`; run inside WSL):

```
python3 tools/trials/linux-ubuntu-24.04-x86_64/gtkmm-gui-driver.py ~/loomora-build/linux-modern/loomora-gtk3 <outdir under $HOME> <suffix> startup startup-wayland default limit eol --preload <fake-uname.so>
python3 tools/trials/linux-ubuntu-24.04-x86_64/gtkmm-glyphs-shoot.py <exe> <outdir> <suffix> <vector>
```

Scenarios: `startup` (6 s without `--probe`, X11 backend, still running, empty standard error, closed), `startup-wayland` (6 s on the default Wayland backend, ended with SIGTERM), `default` (initial screenshot, select all, type `Marie`, screenshot, delay, contrast, close), `limit` (types 1,100 characters, screenshots at the top and bottom of the scrolled greeting), `eol` (needs `--preload`). `dark`, `highcontrast`, `highcontrast-inverse` force `GTK_THEME` (`Adwaita:dark`, `HighContrast`, `HighContrastInverse`) and were run on the C build only, not on the gtkmm one. Replayed 2026-11-14: `startup` and `default` (`still_running_before_close` true, exit status 0 about 8 ms after the close message, empty standard error, no leftover). The end-of-life banner is a **simulation**: build the shim and preload it,

```
mkdir -p ~/loomora-build/trial-scratch && gcc -shared -fPIC -o ~/loomora-build/trial-scratch/fake-uname.so tools/trials/linux-ubuntu-24.04-x86_64/fake-uname.c -ldl
LD_PRELOAD=$HOME/loomora-build/trial-scratch/fake-uname.so ~/loomora-build/linux-modern/loomora-gtk3 --probe     # eol_warning_shown true
```

(the shim only makes `uname(2)` report kernel 2.6.32; the real kernel is 6.18, so this is not the banner on an old system). Both scripts write their files (`gui-<scenario>-<suffix>.json`, PNGs, `trial-measurements-<suffix>.json`) into the output folder, with LF line endings.

Sanitizers: `verify.py full` (its `sanitizer` check) builds a copy of `tools/trials/linux-ubuntu-24.04-x86_64/xlib-ui-harness-cpp.cpp` under ASan and UBSan and runs it with Xft and with the core font (3 fontconfig allocations suppressed in LeakSanitizer). Stored result of 2026-11-14: 0 sanitizer errors in 5 runs. Not replayed in this session (`full` is for graphics changes). The header of that harness still shows an old compile line (`xftdl.cpp`, `-ldl`); `verify.py` has the current one.

## L.8 What the verdicts claim, and what this procedure does not cover

Both verdicts are `ok_restricted`, decision `keep`. Modern: gtkmm build 44cba2ef (167,192 bytes) with 15 of 15 vectors, the probe series, the X11 and Wayland startups, the floor, GTK 4 and static-runtime builds run for the vectors; the dark and high-contrast themes, the contrast ratios and the 1,100-character limit were measured on the C build of 2026-11-13 and not re-run on gtkmm. Legacy: the Xft build (80,216 bytes) with 15 of 15 vectors (also without Xft and for i686), the mixed Unicode string with real glyphs and one empty box, typing, the 1,000-character limit and a clean close; **no text shaping and no bidirectional layout by decision** (accepted by the owner). Neither covers another system, a real keyboard or input method, HiDPI, or the end-of-life banner on a real old system.

## L.9 What to record, and where

- Per CLAUDE.md, store evidence only when it proves something new; the tests (`run_vectors.py`, `ctest`, `check_std.py`) are the proof that behaviour is unchanged. A re-run is mentioned in the verdict notes with its result and is not stored again. Never edit stored evidence. The text files of the evidence folders keep their LF line endings (`validation/evidence/.gitattributes`) because the verdicts hold their SHA-256.
- New evidence goes in `validation/evidence/<adr-or-change>/linux-ubuntu-24.04-x86_64/` of the docs submodule. Images and screenshots need the owner's authorization first.
- Update `validation/verdicts/linux-ubuntu-24.04-x86_64-modern.json` or `-legacy.json` and, for a new build, the deliverable records written by `tools/build.py`. Any fallback taken by a case must be declared in `validation/fallbacks.json` (CLAUDE.md, Fallbacks and Linux Modern UI rules). Then `python tools/validate_records.py` must report 0 errors.

## L.10 Linux Legacy UI on the lowest testable system: manylinux2010 in Docker (platform `linux-centos-6.10-x86_64`)

Written and replayed on 2026-11-15 (PoC scope rule: the lowest testable Linux system; success there counts for newer ones). Verdict `validation/verdicts/linux-centos-6.10-x86_64-legacy.json` (`ok_restricted`), platform record `validation/platforms/linux-centos-6.10-x86_64.json`, evidence `validation/evidence/linux-manylinux2010/linux-centos-6.10-x86_64/`. Commands run from Git Bash or PowerShell on the Windows host (Docker Desktop; `docker` is not available in WSL); from Git Bash set `MSYS_NO_PATHCONV=1` so that `/opt/...` paths are not rewritten.

What the system is: a container of the PyPA image `quay.io/pypa/manylinux2010_x86_64` (CentOS 6.10, glibc 2.12, devtoolset-8 gcc 8.3.1, cmake 3.23.3). It shares the kernel of Docker Desktop (the WSL2 kernel), so only the userland is the old system's. The platform id comes from `/etc/redhat-release` (the image has no `/etc/os-release`, fallback `linux-platform-id-redhat-release`).

1. The image (adds Xft and fontconfig development files, Xvfb and fonts; works around the 403 of vault.centos.org with archive.kernel.org and keeps `/usr/local/lib` in the ld configuration, see the comments of the file):

```
PS> docker build -t loomora-manylinux2010 tools/trials/linux-manylinux2010-x86_64
```

2. Build inside the container (the worktree is mounted at /src, the build directory is the Docker volume `loomora-build` mounted as `/root/loomora-build`, outside the repository; Python 3.8 of the image runs `tools/build.py`, which needs 3.8 or later; it builds with Unix Makefiles and writes `out/linux-legacy/x86_64/loomora-xlib` and the deliverable record):

```
PS> docker volume create loomora-build
PS> docker run --rm -v "${PWD}:/src" -v loomora-build:/root/loomora-build -w /src loomora-manylinux2010 /opt/python/cp38-cp38/bin/python3 tools/build.py linux legacy --arch x86_64
```

3. ctest (selftest and the Catch2 tests; Catch2 first: `git -c protocol.file.allow=always submodule update --init third_party/catch2`):

```
PS> docker run --rm -v "${PWD}:/src" -v loomora-build:/root/loomora-build -w /src loomora-manylinux2010 bash -c "cmake -S . -B ~/loomora-build/core-gcc -DCMAKE_BUILD_TYPE=Release && cmake --build ~/loomora-build/core-gcc -j4 && ctest --test-dir ~/loomora-build/core-gcc --output-on-failure"
```

4. The 15 reference vectors, from the Windows host (the launcher starts one container with a private Xvfb on :99 and runs each vector through `docker exec`; `python3 tools/run_vectors.py ... --docker` works the same from a Linux host with Docker):

```
PS> python tools/run_vectors.py linux-centos-6.10-x86_64 legacy --docker loomora-manylinux2010 --evidence-dir docs/implementation/validation/evidence/<change>/linux-centos-6.10-x86_64
```

Expected: `15 vector(s) run, 15 passed, 0 failed, 16 skipped; 2 truncated to 1000 chars`, `result: PASSED`.

5. Window run and probe facts: `tools/trials/linux-manylinux2010-x86_64/window-run.sh` (compiles `xlib-driver.c`, starts its own Xvfb :98, types, compares two screenshots by hash, closes; expected exit status 0, empty stderr, 2 distinct screenshots), run with `docker run --rm -v "${PWD}:/src" -v loomora-build:/root/loomora-build -v "${PWD}/tools/trials/linux-manylinux2010-x86_64/window-run.sh:/run.sh" loomora-manylinux2010 bash /run.sh`. The probe line alone: `docker run ... bash -c "Xvfb :97 & sleep 2; DISPLAY=:97 /src/out/linux-legacy/x86_64/loomora-xlib --probe"`.

Result of 2026-11-15: build with no warning (131,712 bytes, needs GLIBC_2.4 and GLIBCXX_3.4 at most), ctest 2 of 2, 15 of 15 vectors, window opened, typed into and closed with exit 0. Not covered: a CentOS 6 kernel, a real X server, the end-of-life banner (the kernel is the host's), a real keyboard, any other distribution, i686 and ARM.

## L.11 Linux Legacy UI on an older userland: manylinux1 in Docker (platform `linux-centos-5.11-x86_64`)

Written and replayed on 2026-11-15, after L.10, at the owner's request to try an older Linux userland than CentOS 6.10. Verdict `validation/verdicts/linux-centos-5.11-x86_64-legacy.json` (`ok_restricted`), platform record `validation/platforms/linux-centos-5.11-x86_64.json` (`is_floor` true; that of `linux-centos-6.10-x86_64` became false, its verdict is kept), evidence `validation/evidence/linux-manylinux1/linux-centos-5.11-x86_64/`. Same method as L.10: build and run inside the container, commands from Git Bash or PowerShell on the Windows host (`MSYS_NO_PATHCONV=1` in Git Bash).

What the system is: a container of the PyPA image `quay.io/pypa/manylinux1_x86_64` (CentOS 5.11, glibc 2.5, devtoolset-2 gcc 4.8.2, Python 2.4 as system Python and 3.5 to 3.9 under `/opt/python`, no CMake). Only its schema-2 manifest is pullable by Docker 29 (images of 2015 and before, schema 1, are not). The kernel is the Docker Desktop one (WSL2 6.18), so only the userland is the old system's. The platform id comes from `/etc/redhat-release` (`CentOS release 5.11 (Final)`, no `/etc/os-release`: the same fallback `linux-platform-id-redhat-release`, no change of code).

The image is used UNMODIFIED, but it needs a kernel that maps the vsyscall page: glibc 2.5 reads the time through it, and with the default WSL2 kernel (`vsyscall=none`) bash, perl, python3, yum and the compiler proper die with exit status 139. The owner enabled `vsyscall=emulate` in the kernel command line of WSL2 (`kernelCommandLine` in `.wslconfig`, then `wsl --shutdown` and a Docker restart). Check before anything else: `docker run --rm quay.io/pypa/manylinux1_x86_64 bash -c "cat /proc/cmdline; echo ok"` must end with `vsyscall=emulate` and print `ok` (exit 139 means the setting is missing). A first run of 2026-11-15 on the default kernel used a libc patched at its two vsyscall call sites; it was replaced by the unpatched run and the patch script was removed. Two defects of the old Xvfb are worked around in the Dockerfile: Xvfb 1.1.1 needs `rgb.txt` (`xorg-x11-server-utils`) or the UI silently falls back to the core font (probe lines then carry `glyph-substitution` where Xft would give `missing-glyph`: check that the mixed string gives `missing-glyph`), and a wrapper gives it a font path, without which every probe takes 2 to 5 seconds to display. CMake is the pip wheel `cmake<3.23` (manylinux1 build).

1. The image (the yum repositories are already on `http://archive.kernel.org/centos-vault/5.11/`; http only):

```
PS> docker build -t loomora-manylinux1 tools/trials/linux-manylinux1-x86_64
PS> docker volume create loomora-build-m1
```

2. Build inside the container (Python 3.8 of the image runs `tools/build.py`; gcc 4.8 has no `-dumpfullversion`, `tools/build.py` falls back to `-dumpversion`):

```
PS> docker run --rm -v "${PWD}:/src" -v loomora-build-m1:/root/loomora-build -w /src loomora-manylinux1 /opt/python/cp38-cp38/bin/python3 tools/build.py linux legacy --arch x86_64
```

3. ctest (as L.10 step 3, plus `-DPython3_EXECUTABLE=/opt/python/cp38-cp38/bin/python3` because the system Python is 2.4, and `-DLOOMORA_XLIB_XFT=OFF`, the build directory being a second one).

4. The 15 reference vectors, from the Windows host (the launcher is unchanged; the volume is the one of step 2):

```
PS> python tools/run_vectors.py linux-centos-5.11-x86_64 legacy --docker loomora-manylinux1 --docker-volume loomora-build-m1 --evidence-dir docs/implementation/validation/evidence/<change>/linux-centos-5.11-x86_64
```

Expected: `15 vector(s) run, 15 passed, 0 failed, 16 skipped; 2 truncated to 1000 chars`, `result: PASSED`, `display_ms` below about 100.

5. Window run: `tools/trials/linux-manylinux1-x86_64/window-run.sh`, run as in L.10 step 5 (`docker run --rm -v "${PWD}:/src" -v loomora-build-m1:/root/loomora-build -w /src loomora-manylinux1 bash /src/tools/trials/linux-manylinux1-x86_64/window-run.sh`); expected exit status 0, empty stderr, 2 distinct screenshots.

Result of 2026-11-15 (unmodified image, evidence files `unpatched-*`): build with no warning (66,208 bytes, needs GLIBC_2.2.5, GLIBCXX_3.4 and CXXABI_1.3 only, so the stock libstdc++ 4.1.2 suffices), ctest 2 of 2, 15 of 15 vectors, window opened, typed into and closed with exit 0. Not covered: a CentOS 5 kernel (2.6.18), a real X server, the end-of-life banner (the kernel is the host's), a real keyboard, CentOS 5.0 to 5.10, any other distribution, i686 and ARM.
