# Procedure: Windows (Legacy and Modern UIs)

Replayable procedure for the platforms that were really run in this project (FR-016, SC-004, SC-008):

| Platform record | Variant | Verdict | Where it is run |
|---|---|---|---|
| `windows-11-x64` | legacy | `validation/verdicts/windows-11-x64-legacy.json` | the physical Windows 11 host (part A) |
| `windows-2000-x86` | legacy | `validation/verdicts/windows-2000-x86-legacy.json` | the VirtualBox VM "Windows 2000" (part B) |
| `windows-11-x64` | modern (WinUI 3) | `validation/verdicts/windows-11-x64-modern.json` | the physical Windows 11 host (part C) |

Every command is the one that was run, taken from `tools/`, the ADRs, the platform records and the verdicts.

## Status of this document

- Written on 2026-11-14 from the records and the scripts. Part A was replayed on the same host on that day: the build, the host quick check, the probe runs, `tools/run_vectors.py` for both executables, the import check, the self-tests and the Windows 11 trial driver, with the results quoted below. Part B (the VM) was **not** replayed on 2026-11-14: the VM was powered off and this procedure never starts it. Its expected results are those stored in the verdict and the evidence.
- The Modern UI (WinUI 3) was built and run on 2026-11-15 on the Windows 11 x64 host only (part C). It has no Windows 10 1809 verdict (no machine) and no ARM64 verdict (built, never run; no ARM64 platform record), so the validator still reports the missing verdicts as warnings.
- Windows 2000 was run on build 2195 **without service pack**. The decided floor (SP4), Windows XP, Vista, 7, 8.1, 10, ARM64 and real hardware were not run: nothing here says anything about them.
- The ordinary build and run need only the main repository. The records (`validation/`) are in the docs submodule: `git -c protocol.file.allow=always submodule update --init`.

## Common prerequisites

Repository state: a checkout of the main repository with the submodule `docs/implementation` initialised. The commands below are run from the repository root.

Tools (versions read on 2026-11-14 on the machine of the trials; the trial records themselves only name the compiler `gcc 13-win32` and the operating systems, so these are the versions known to work, not a proof of the exact ones used on 2026-11-12):

| Where | Tool | Version |
|---|---|---|
| Windows host | Python | 3.13.17 (3.8 or later is enough), package `jsonschema` 4.26.0 |
| Windows host | git | 2.56.0.windows.1 |
| Windows host | clang, cmake, ninja (host build of `verify.py quick`) | clang 23.1.1, cmake 4.4.0, ninja 1.12.1 |
| Windows host | VirtualBox (part B only) | 7.2.22 r175245, `C:\Program Files\Oracle\VirtualBox\VBoxManage.exe` |
| WSL | distribution | `Ubuntu-24.04` (WSL 3.0.5.0) |
| WSL | MinGW-w64, **win32 thread model, msvcrt runtime** | `i686-w64-mingw32-gcc` and `x86_64-w64-mingw32-gcc` "13-win32" (packages `gcc-mingw-w64-i686-win32`, `gcc-mingw-w64-x86-64-win32`, `g++-mingw-w64-*-win32` 13.2.0-6ubuntu1+26.1; select the `-win32` variants with `update-alternatives --config i686-w64-mingw32-g++`, `-gcc` and the x86_64 ones, see `MINGW_INSTALL_HELP` in `tools/build.py`) |
| WSL | cmake, ninja, python3 | cmake 3.31.12, ninja 1.11.1, python3 3.12.3 |

Rules that apply to every WSL build (CLAUDE.md): build under `$HOME/loomora-build/`, never in `/tmp` and never in the Windows tree. `tools/build.py` and `tools/guest/*.sh` already do. When several checkouts of the repository exist, give each its own folder, because a CMake build folder is tied to one source folder: `LOOMORA_BUILD_DIR=$HOME/loomora-build/<name>` (read by `tools/build.py`).

## Part A: Windows 11 x64, Legacy UI (host)

### A.1 System

`validation/platforms/windows-11-x64.json`: Windows 11 Pro 26H2, 10.0.26300.9974, French (fr-FR), 64-bit, on a physical MSI GL63 8RD laptop (Intel Core i5-8300H, 24 GiB), system DPI 96, light theme, high contrast off, Windows running on top of the Microsoft hypervisor (WSL2 installed). No image: this is the machine of the trial.

Record the facts of the machine you replay on before running anything (the trial driver of A.7 writes them to `host-environment.json`): `cmd /c ver`, `(Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion').DisplayVersion`, system DPI, and `AppsUseLightTheme` (`HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize`). A different Windows version or machine is a new platform record, not a replay.

Do not touch the keyboard while a window run is in progress: the window takes the focus and a stray keystroke goes into the field. On 2026-11-14, one of the 40 probe runs of the trial driver read `input` = `vsCoucou` (message `Bonjour vsCoucou`, `greeting_matches` true); the source of the two keystrokes was not identified. Such a run is not a failure of the program; repeat the series.

### A.2 Get the code and the build tools

```
git -c protocol.file.allow=always submodule update --init
python -m pip install jsonschema
wsl -d Ubuntu-24.04 -- sudo apt install g++-mingw-w64-i686-win32 g++-mingw-w64-x86-64-win32 cmake ninja-build python3
```

### A.3 Build the executables (MinGW-w64 in WSL)

```
wsl -d Ubuntu-24.04 --cd <repo as /mnt/<drive>/...> -- python3 tools/build.py windows legacy
```

`tools/build.py windows legacy` builds twice with CMake + Ninja (toolchains `cmake/toolchains/mingw-w64-win2000-i686.cmake` and `mingw-w64-win-x86_64.cmake`) under `$HOME/loomora-build/windows-legacy-x86` and `-x64`, checks every imported function against `tools/win2000_imports.json` (`tools/check_imports.py`) and copies the executables to `out/windows-legacy/x86/loomora.exe` and `out/windows-legacy/x64/loomora.exe`.

Expected (replayed 2026-11-14):

```
ok: .../windows-legacy-x86/ui/legacy/windows-win32/loomora.exe (x86): 112 imports from 4 DLLs, all on the Windows 2000 SP4 allowlist (0 to verify)
ok: .../windows-legacy-x64/ui/legacy/windows-win32/loomora.exe (x64): 116 imports from 4 DLLs, all on the Windows XP x64 / Server 2003 x64 allowlist (15 to verify)
artifact: out/windows-legacy/x64/loomora.exe (703383 bytes)
artifact: out/windows-legacy/x86/loomora.exe (696127 bytes)
```

The sizes are those of the verdict. The SHA-256 differs from one build to the next (the MinGW linker stamps the build time, as the verdict says), so compare the sizes and the import counts, not the hash. The 4 DLLs are `gdi32`, `kernel32`, `msvcrt`, `user32`.

The tool also rewrites `validation/deliverables/windows-11-x64-legacy.json` and `windows-2000-x86-legacy.json` in the docs submodule (only the hash in the notes changes). A replay that does not release a new build discards that change: `git -C docs/implementation checkout -- validation/deliverables`.

### A.4 Host build and quick check (clang + Ninja)

```
python tools/verify.py quick
```

On the Windows host it configures `-G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++` under `C:\Users\<user>\loomora-build\verify-<hash>\default`, builds the backend and the Win32 UI, runs `ctest`, runs `tools/run_vectors.py` on that executable, then `check_std.py` and `validate_records.py`. Expected (replayed 2026-11-14):

```
PASS build core + loomora.exe
PASS ctest 2/2
PASS run_vectors windows legacy 15/15
SKIP std: no g++ on this host (clang with the MSVC STL cannot do strict C++11); run quick in WSL
PASS records
quick: 4 passed, 0 failed, 1 skipped
```

The strict C++11 check is run in WSL (`wsl -d Ubuntu-24.04 --cd <repo> -- python3 tools/check_std.py`, part of `verify.py quick` there, see `procedures/linux.md`). Do not run `verify.py full` for this: it is for graphics changes only.

### A.5 One probe run of each executable, and the probe line

Copy the executable to a scratch folder (the probe file `loomora-probe.json` is written next to it), run it with `--probe` (it has no console and exits by itself after about 2 s), then check the line:

```
$d = New-Item -ItemType Directory -Path "$env:TEMP\loomora-run" -Force
Copy-Item out\windows-legacy\x64\loomora.exe $d
Start-Process -Wait -FilePath "$d\loomora.exe" -ArgumentList '--probe'
Get-Content "$d\loomora-probe.json"
python tools\check_probe.py "$d\loomora-probe.json"
```

Expected: the check prints `OK` (exit 0) and the line has this shape (values of `display_ms`, `peak_rss_kb` and `timestamp_utc` vary):

```
{"schema":1,"platform_id":"windows-11-x64","ui_variant":"legacy","input":"Coucou","message":"Bonjour Coucou","greeting_matches":true,"core":"cpp","compiler":"gcc 13-win32","displayed":true,"display_ms":64,"peak_rss_kb":14248,"eol_warning_shown":false,"disabled_features":[],"timestamp_utc":"2026-11-14T16:59:31Z"}
```

Same with `-ArgumentList '--probe','--probe-input','Marie'` and `python tools\check_probe.py "$d\loomora-probe.json" --input Marie`: `input` `Marie`, `message` `Bonjour Marie`. The x86 executable (`out\windows-legacy\x86\loomora.exe`) runs under WOW64 and reports the same `platform_id` (`windows-11-x64`): it reports the architecture of the operating system. Delete the scratch folder afterwards.

### A.6 The reference vectors

```
python tools/run_vectors.py windows-11-x64 legacy --arch x64
python tools/run_vectors.py windows-11-x64 legacy --arch x86
```

Each vector gets its own scratch copy of the executable, started with `CreateProcess` and a UTF-16 command line. Expected (replayed 2026-11-14, both architectures):

```
summary: 15 vector(s) run, 15 passed, 0 failed, 16 skipped; 2 truncated to 1000 UTF-16 units; ... s with 4 worker(s)
result: PASSED
```

The 16 skipped vectors are the error cases that cannot be typed into a field (the backend tests cover them); the 2 truncated ones are the 4,096-byte vectors, expected to be cut to their first 1,000 characters.

### A.7 Import check, self-tests and the window run

Import check on any executable (in WSL, with the MinGW `objdump` of its architecture):

```
wsl -d Ubuntu-24.04 --cd <repo> -- python3 tools/check_imports.py out/windows-legacy/x64/loomora.exe --objdump x86_64-w64-mingw32-objdump
```

Expected: `ok: ... (x64): 116 imports from 4 DLLs, all on the Windows XP x64 / Server 2003 x64 allowlist (15 to verify)`. For `out/windows-legacy/x86/loomora.exe` use `i686-w64-mingw32-objdump`: 112 imports.

Backend self-test (no UI), built in WSL, run on the Windows host:

```
wsl -d Ubuntu-24.04 --cd <repo> -- sh tools/guest/build_selftest.sh
out\windows-legacy\guest\loomora_selftest-x64.exe
out\windows-legacy\guest\loomora_selftest.exe
```

`build_selftest.sh` builds under `$HOME/loomora-build/selftest-w2k` and `selftest-x64` and also runs the import check on the two programs (`loomora_selftest.exe`: x86, run on Windows 11 under WOW64; `loomora_selftest-x64.exe`: x64; `loomora_selftest-noshim.exe` is a control for part B). Expected: each program prints `all passed` and exits with status 0 (replayed 2026-11-14).

Window run and measurements (40 probe runs, two window runs, host facts):

```
python tools/trials/windows-11-x64/trial-driver.py
```

About 2 minutes. For each architecture it runs `--probe` 10 times and `--probe --probe-input Marie` 10 times, then starts the window without `--probe`, reads the EDIT and the STATIC with `WM_GETTEXT`, captures it with `PrintWindow`, selects the field (`EM_SETSEL`) and types `Marie` with `WM_CHAR`, types 1,100 characters, closes with `WM_CLOSE`. Expected (from `gui-<arch>.json` of the replay of 2026-11-14 and the stored evidence): initial EDIT `Coucou`, STATIC `Bonjour Coucou`; after typing, EDIT `Marie`, STATIC `Bonjour Marie`; `em_getlimittext` 1000; after 1,100 characters the EDIT holds 1,000 and the STATIC 1,008 characters (`Bonjour ` + 1,000); exit status 0 about 15 ms after `WM_CLOSE`; no `loomora.exe` left. Typed input is window messages, not a keyboard. The script writes its output files **next to itself** (`tools/trials/windows-11-x64/`, untracked): move them out, they are evidence only when they prove something new (see A.9).

### A.8 What the verdict claims, and what this procedure does not cover

The stored verdict covers: the two executables of 2026-11-12 (and the re-run of the rebuilt ones after ADR 0004), 40 probe runs, 15 of 15 vectors for each architecture, the window with its glyph limits (only `Coucou` and `Marie` were looked at in captures), the light and dark theme at 96 DPI. Not covered: other Windows versions, DPI above 100%, high contrast, per-monitor DPI, other machines, the 4.5:1 contrast measure. A replay that finds a different result on another Windows version must write a new platform record and verdict, not edit this one.

Known gaps in the stored scripts: `tools/trials/windows-11-x64/shoot-window.py` and `compare-imports.py` (ADR 0004 captures and import comparison) still compute their root and the path of `trial-driver.py` from the folder they had before they were moved to `tools/trials/`, so they do not run as they are; the vector logs of the re-run after ADR 0004 were not stored (CLAUDE.md: no re-storing of the same proof).

### A.9 What to record, and where

- Per CLAUDE.md, store evidence only when it proves something new (a new feature or behaviour); the tests (`run_vectors.py`, the self-tests, `ctest`) are the proof that behaviour is unchanged. A re-run is mentioned in the verdict notes with its result and is not stored again. Never edit stored evidence.
- New evidence goes in `validation/evidence/<adr-or-change>/windows-11-x64/` of the docs submodule. Images and screenshots need the owner's authorization first.
- Update `validation/verdicts/windows-11-x64-legacy.json` (status, restrictions, notes) and, if the build changed, the deliverable record written by `tools/build.py`. The `evidence` items hold a SHA-256: `python tools/validate_records.py` must still report 0 errors.

## Part B: Windows 2000 x86, Legacy UI (VirtualBox VM)

### B.1 System

`validation/platforms/windows-2000-x86.json`: Windows 2000 Professional 5.00.2195 (Uniprocessor Free), **no service pack**, English (US), 32-bit, in a VirtualBox 7.2.22 (r175245) VM named **`Windows 2000`**: 1 vCPU, 168 MB RAM, PIIX3 chipset, PIIX4 IDE disk, BIOS, NAT, 16 MB video memory, VirtualBox Graphics Adapter 800x600x32 at 96 DPI, Guest Additions 6.1.18 r142142 (run level 3). The system DLLs are the RTM ones of 1999-12-07 (kernel32 5.0.2191.1, user32 5.0.2180.1, msvcrt 6.1.8637.0), Windows Script Host 5.1, Internet Explorer 5.00.2920.0000. The user layout is French AZERTY while the system default layout is US.

Account and state, prepared by the owner: auto-logon of the user **`name`** with an **empty password**; a snapshot named **`Claude`** taken before the trials (`VBoxManage snapshot "Windows 2000" list` on 2026-11-14 shows two snapshots named `Claude`, the current one nested in the first). The trials never restored it, never changed a VM setting and left the VM running as found (`VMStateChangeTime` identical before and after). The installation medium of the guest and the way it was installed are **not recorded** in the repository: replaying on another Windows 2000 installation gives another platform, and the owner's VM (or its snapshot) is the reference system. Restoring or starting the VM is an action of the owner; the launcher below refuses to run on a VM that is not running and never starts, stops, restores or reconfigures one (ADR 0003).

All guest files are under `C:\Documents and Settings\name\` (the launcher folder is `C:\Documents and Settings\name\loomora-vectors`, `LOOMORA_VM_DIR`). The system DLLs live in `C:\WINNT\system32`.

### B.2 Prepare the VM

1. The VM is running and the user `name` is logged on to the desktop (auto-logon). Starting it is the owner's action (VirtualBox Manager; `VBoxManage startvm "Windows 2000"` is the command-line equivalent, not run in the trials). Wait until the desktop is visible.
2. Check from the host that the guest answers (the launcher does the same checks, so this is optional):
   ```
   VBoxManage showvminfo "Windows 2000" --machinereadable          (VMState="running")
   VBoxManage guestproperty get "Windows 2000" /VirtualBox/GuestInfo/OS/Product     (Value: Windows 2000)
   ```
3. Settings are environment variables, needed only if they differ from the defaults `Windows 2000`, `name`, empty password: `LOOMORA_VM_NAME`, `LOOMORA_VM_USER`, `LOOMORA_VM_PASSWORD` (a non-empty password is passed through a temporary `--passwordfile`, never on a command line), `LOOMORA_VM_DIR`, `LOOMORA_VBOXMANAGE`.
4. Build the executable as in A.3 (`out/windows-legacy/x86/loomora.exe`). The guest helper is built by `tools/run_vectors.py` when `out/windows-legacy/guest/runw.exe` is missing or older than its source; to build it by hand: `wsl -d Ubuntu-24.04 --cd <repo> -- sh tools/guest/build_runw.sh` (MinGW-w64 i686, `$HOME/loomora-build/guest-helper`, about 5 KB, `kernel32.dll` only).

### B.3 How the guest is driven (ADR 0003)

The Guest Additions 6.1.18 do not pass command-line arguments to a guest program, and Windows Script Host 5.1 converts a command line to the ANSI code page (1252: every other character becomes `?`) and expands `%VARIABLES%`. So `VBoxManage guestcontrol ... run` can only start a program or a `.bat` **with no arguments**, and each run does:

1. `VBoxManage guestcontrol "Windows 2000" copyto --username name --password "" <host file> "<guest path>"` copies, into the guest folder: `loomora.exe`, `runw.exe` (checked by SHA-256 after the copy), three small `.bat` files (`run.bat`, `kill.bat`, `launch.bat`, each `cd /d` to the folder and start `runw.exe` with no option, `--kill`, `--nowait`) and `cmdline.txt` = the whole command line as UTF-16LE with a byte order mark, no line ending, for example `"C:\Documents and Settings\name\loomora-vectors\loomora.exe" "--probe" "--probe-input" "Marie"` (each argument quoted with the C runtime rules, `quote_windows_arg` in `tools/run_vectors.py`).
2. `VBoxManage guestcontrol "Windows 2000" run --username name --password "" --exe "C:\Documents and Settings\name\loomora-vectors\run.bat" --wait-stdout --wait-stderr --timeout <ms>` runs `run.bat`, which starts `runw.exe`. `runw.exe` (source `tools/guest/runw.c`) reads `cmdline.txt` without any line parsing, deletes `loomora-probe.json`, starts that exact Unicode command line with `CreateProcessW`, waits, and prints to the standard output that comes back: `LOOMORA-EXIT <status>`, `LOOMORA-WALLMS <ms>`, `LOOMORA-PROBE <first line of loomora-probe.json>`, `LOOMORA-LEFTOVER <number of loomora.exe still alive, terminated>`.
3. `runw.exe --nowait` (through `launch.bat`) starts the window and returns at once (`LOOMORA-STARTED <pid>`), for screenshots and typed input; `runw.exe --kill` (`kill.bat`) ends every `loomora.exe` and `runw.exe` (`LOOMORA-KILLED <count>`).

Windows 2000 has no `tasklist` or `taskkill`; the trial driver counts processes with WMI (`guest-procs.vbs`, started by a `.bat` with `cscript //nologo`).

### B.4 The reference vectors in the guest

```
python tools/run_vectors.py windows-2000-x86 legacy
```

(`--arch x86` is the only accepted value.) The runs are serial, about 2.3 s each. Expected, from `validation/evidence/first-trial/windows-2000-x86/run-vectors-windows-2000-x86-legacy-log.txt`:

```
launcher: VirtualBox VM 'Windows 2000' (168 MB RAM), guest Windows 2000 5.0.2195 ..., guest user 'name', folder C:\Documents and Settings\name\loomora-vectors, ... out/windows-legacy/x86/loomora.exe (... bytes, sha256 ...)
summary: 15 vector(s) run, 15 passed, 0 failed, 16 skipped; 2 truncated to 1000 UTF-16 units; ~35 s with 1 worker(s)
result: PASSED
```

Also stored: 25 hard command-line texts (quotes, backslashes, `%PATH%`, tabs and newlines, control characters, a BOM, 1,100 and 1,500 characters, an emoji at the 1,000 limit): `python tools/trials/windows-2000-x86/roundtrip-vectors-gen.py hard.json` then `python tools/run_vectors.py windows-2000-x86 legacy --vectors hard.json` (25 of 25 passed on 2026-11-12). Write `hard.json` outside the repository.

### B.5 One probe run and the probe line

The vectors run already checks each probe line with `tools/check_probe.py`. For a single line, the trial driver's `runs` step does 10 runs of `--probe` and 10 of `--probe --probe-input Marie` and checks every line:

```
python tools/trials/windows-2000-x86/trial-driver.py runs
```

Expected line (stored `probe-default-run1.json`; `display_ms` was 3 to 13 and `peak_rss_kb` a constant 1084 over the 20 runs of 2026-11-12):

```
{"schema":1,"platform_id":"windows-2000-x86","ui_variant":"legacy","input":"Coucou","message":"Bonjour Coucou","greeting_matches":true,"core":"cpp","compiler":"gcc 13-win32","displayed":true,"display_ms":10,"peak_rss_kb":1084,"eol_warning_shown":true,"disabled_features":["dpi-awareness","dark-mode"],"timestamp_utc":"2026-11-12T23:01:20Z"}
```

What differs from Windows 11, and must: `platform_id` `windows-2000-x86`; `eol_warning_shown` **true** (the end-of-life banner is shown); `disabled_features` `["dpi-awareness","dark-mode"]` (the system has no `SetProcessDPIAware` and no `AppsUseLightTheme`). The timestamp is the guest clock, which is not synchronised with the host. 20 of 20 runs exit with status 0 and leave no process.

### B.6 Imports against the real system DLLs

The loader of Windows 2000 refuses an executable whose import table names a function that its DLL does not export (spike S1, ADR 0002). With the VM running:

```
python tools/trials/windows-2000-x86/trial-driver.py exports
```

It copies `kernel32`, `user32`, `gdi32`, `advapi32`, `ntdll`, `psapi`, `msvcrt` and `dwmapi` from `C:\WINNT\system32` to a scratch folder inside the launcher scratch (Microsoft binaries are never stored in the repository), then checks every function of the x86 allowlist, every import of `loomora.exe`, and the functions looked up at run time. Expected: 112 of 112 imports of `loomora.exe` exported by the guest DLLs, every allowlist function exported; `dwmapi.dll` is absent (the dark title bar is only used when the dark theme is on). The same check on a folder of DLLs: `python tools/check_imports.py --verify-dlls <folder> --arch x86` and `python tools/check_imports.py <exe> --against <folder>`.

### B.7 Self-test in the guest

```
wsl -d Ubuntu-24.04 --cd <repo> -- sh tools/guest/build_selftest.sh
python tools/trials/windows-2000-x86/selftest-driver.py
```

`selftest-driver.py` checks the imports of `loomora_selftest.exe` (must pass) and of `loomora_selftest-noshim.exe` (control: must fail on `KERNEL32!GetThreadId`), checks them against the DLLs of the guest, copies both programs and a `.bat` into a new guest folder `<LOOMORA_VM_DIR>-selftest` (which must not exist), runs it and deletes the folder. Expected (`validation/evidence/selftest-no-iostream/windows-2000-x86/selftest-noiostream-vm-run.txt`): `loomora_selftest.exe` prints `all passed`, exit code 0 (67 imports, none outside the allowlist, none missing in the DLLs); `loomora_selftest-noshim.exe` prints nothing, exit code 128 (the loader refuses it). The `selftest` step of `trial-driver.py` is the first attempt, before `<iostream>` was removed from `core/tests/selftest.cpp`; it ends with 128 for both programs and is kept as history, do not use it as the check. `tools/trials/windows-2000-x86/loader-control.sh` builds two tiny control programs (`plain.exe`, `vista.exe`) for the loader behaviour.

### B.8 The window, typed input and glyphs

```
python tools/trials/windows-2000-x86/trial-driver.py gui glyphs
```

Run `facts` first (the `gui` step reads the keyboard layout from the `guest-facts-output.txt` that `facts` writes next to the script; without it the driver assumes US scancodes). `gui` starts `loomora.exe` without `--probe` (`runw.exe --nowait`), takes screenshots of the guest screen with `VBoxManage controlvm "Windows 2000" screenshotpng <file>`, and types with the virtual keyboard: `VBoxManage controlvm ... keyboardputstring Marie` assumes a US layout and, on this French AZERTY user layout, types `?qrie` (kept in the evidence as the measured limit); the driver then sends `VBoxManage controlvm ... keyboardputscancode ...` with the scancodes of the physical keys (Home, Shift+End to select `Coucou`, then the keys that give M a r i e on AZERTY), then Alt+F4. Expected (stored `gui-run.json`, `window-*.png`): window with the end-of-life banner, `Bonjour Coucou`; after the scancodes `Marie` / `Bonjour Marie`; after Alt+F4 no `loomora.exe` process. `glyphs` photographs a probe-mode window with `Zoé Hélène çà 日本語 שלום 😀`: accented Latin drawn, the CJK characters and the emoji as missing-glyph boxes (the fonts of this guest, not an error of the string). The keyboard, the 1,000-character limit by keystrokes, high contrast, dark theme and DPI above 96 were not tried in this guest.

Other steps of the same driver: `facts` (read-only `guest-facts.vbs`: registry, WMI and file versions, writes `guest-facts-output.txt` and `vm-environment.json`), `memory` (cross-checks `peak_rss_kb` with `memtest.c` and a WMI poll), `cleanup` (deletes the guest folder `C:\Documents and Settings\name\loomora-vectors`). Default steps: `facts exports runs memory gui glyphs selftest`. The driver writes its output files next to itself (`tools/trials/windows-2000-x86/`, untracked): move them out.

### B.9 What the verdict claims, and what this procedure does not cover

The verdict is `ok_restricted` for the x86 executable on Windows 2000 build 2195 RTM in this VM: the window shows `Bonjour Coucou`, updates to `Bonjour Marie`, shows the end-of-life banner, takes the skipped-feature path for DPI awareness and dark mode, writes a valid probe line (20 of 20 runs, 15 + 25 of 40 vectors), exits cleanly (2 s probe, Alt+F4), leaves no process. It says nothing about SP4, real hardware, other languages or any other VM. The measurements describe the executable of 2026-11-12 (sha256 `16e78596...7494e`); the rebuilt one (ADR 0004, `67a695c8...5a0`) was re-verified (15 of 15 vectors, self-test, 112 of 112 imports, the new message font) but the 10-run probe series, the memory cross-check and the glyph screenshot were not re-run on it.

### B.10 What to record, and where

Same rules as A.9, with the folder `validation/evidence/<adr-or-change>/windows-2000-x86/` and the verdict `validation/verdicts/windows-2000-x86-legacy.json`. Record the guest facts (`vm-environment.json`: VM state before and after, snapshot name, Guest Additions version) with every new run, and state in the notes whether the snapshot was restored. After a trial, remove the guest folder (`trial-driver.py cleanup`) and leave the VM as found.

## Part C: Windows 11 x64, Modern UI (WinUI 3, host)

Verdict `validation/verdicts/windows-11-x64-modern.json`, deliverable `validation/deliverables/windows-11-x64-modern.json`, evidence `validation/evidence/winui3/windows-11-x64/`. Run on 2026-11-15 on the same physical host as part A. The commands below are the ones that were run, from the repository root, in PowerShell on the Windows host (no WSL, no Visual Studio window).

### C.1 What must be installed

To BUILD (the machine of the trial; versions read on 2026-11-15):

| Tool | Version |
|---|---|
| Visual Studio 2026 Community (C++ desktop workload, and for ARM64 the component "MSVC Build Tools for ARM64/ARM64EC (Latest)") | MSVC toolset 14.52.37025 (`cl.exe` 19.52.37029); 14.44 is also installed and not used |
| CMake | 4.4.0 (the `Visual Studio 18 2026` generator needs 4.2 or later) |
| .NET SDK | 10.0.404, pinned by `ui/modern/windows-winui3/global.json` (`rollForward` `latestPatch`); SDKs 9.0.321 and 11.0.100 are also installed and are not used by the build (see C.8) |
| NuGet (network, or a filled cache) | `Microsoft.WindowsAppSDK.WinUI` and `Microsoft.WindowsAppSDK.Runtime` 2.6.5 (with `.Base`, `.Foundation`, `.InteractiveExperiences` 2.6.5), `Microsoft.Windows.SDK.BuildTools` 10.0.26100.4654, `Microsoft.Windows.SDK.BuildTools.MSIX` 1.7.251221100, `Microsoft.Web.WebView2` 1.0.3719.77, `Microsoft.Windows.SDK.NET.Ref` 10.0.19041.57, the `Microsoft.NETCore.App.Runtime.win-x64` / `win-arm64` packs 8.0.34 and 10.0.15 (all restored from nuget.org, cache `NUGET_PACKAGES`) |

To RUN a published folder on a machine (unpackaged, framework-dependent: the folder carries neither the .NET runtime nor the Windows App Runtime, which is why it is 38 MB and not the 100 to 250 MB of a self-contained one):

1. the **.NET Desktop Runtime** of the build: 8 (`net8` zip, the Windows 10 1809 fallback, `RollForward` `Major` so a later major is used when 8 is missing) or 10 (`net10` zip), x64 or ARM64 as the zip;
2. the **Windows App Runtime 2.6** of the same architecture (package `Microsoft.WindowsAppRuntime.2`, version 2.6.x). Without it the program says so and exits with status 1 (message box outside probe mode, standard error in probe mode); it is found by the bootstrapper API (`Bootstrap.TryInitialize(0x00020006)` in `Program.cs`);
3. Windows 10 1809 (build 17763) or later. Installing the runtimes needs administrator rights on the machine; the application itself needs none.

The host of the trial had the .NET runtimes 6.0.36, 8.0.34, 9.0.23, 10.0.15 and 11.0.0 and the Windows App Runtime packages `Microsoft.WindowsAppRuntime.2` 2.4.0, 2.5.1, 2.6.4 and 2.6.5 (x64; 2.6.5 also x86) installed before the trial.

### C.2 Build

```
PS> python tools/build.py windows modern
```

It runs `cmake --preset host-msvc` and `--build --preset host-msvc` (x64) and, when `Hostx64\arm64\cl.exe` exists, the same with `host-msvc-arm64`: `loomora.dll` by MSVC with `/MT` and `cl.exe`'s default `/std:c++14` (MSVC has no `/std:c++11`; CMake's `CXX_STANDARD 11` target property adds no flag), exporting exactly the six `loomora_*` functions and importing only `KERNEL32.dll` (`validation/evidence/winui3/windows-11-x64/loomora-dll-msvc.txt`). Then `dotnet publish` of `ui/modern/windows-winui3` for each of `net8.0-windows10.0.19041.0` and `net10.0-windows10.0.19041.0` and for `win-x64` and `win-arm64` (`-p:Platform=...`, `-p:LoomoraDll=...`), checks the machine type of the three native files of each folder, and zips each folder: `out/windows-modern/loomora-winui-<net8|net10>-<x64|arm64>.zip`, published folders in `build/windows-modern/<net8|net10>-<x64|arm64>/`. Measured on 2026-11-15:

| Target | Published folder | Zip |
|---|---|---|
| net8 x64 | 34 files, 37,792,741 bytes | 9,925,788 bytes |
| net10 x64 | 34 files, 37,803,469 bytes | 9,930,841 bytes |
| net8 arm64 | 34 files, 38,063,631 bytes | 9,906,558 bytes |
| net10 arm64 | 34 files, 38,070,775 bytes | 9,909,684 bytes |

The biggest file is `Microsoft.Windows.SDK.NET.dll` (24.9 MB, the Windows SDK projection of the target). With the `Microsoft.WindowsAppSDK` metapackage instead of the two component packages the net10 x64 folder was 81,863,410 bytes: the metapackage adds the AI, ML, Search and Widgets components (`onnxruntime.dll`, `DirectML.dll`, about 44 MB), so the project references only `Microsoft.WindowsAppSDK.WinUI` and `.Runtime` (same version 2.6.5). The ARM64 folders were built, their native files are ARM64 (machine 0xAA64), and they were never run: there is no ARM64 machine. Windows App SDK 2.6.5 accepted `net8.0` (no older SDK was needed). The zips of an identical source are byte-identical for net10 (the sha256 came out the same in four builds); the net8 zips differ by a few bytes from one build to the next.

The XAML compiles with `dotnet build`/`dotnet publish` of the .NET SDK 10.0.404: the Visual Studio MSBuild was not needed. `EnableMsixTooling` is on: without it `dotnet publish` of an unpackaged app leaves `Loomora.WinUI.pri` (the XAML resources) out of the folder and the window fails to load (`XamlParseException`).

### C.3 The reference vectors

```
PS> python tools/run_vectors.py windows-11-x64 modern --evidence-dir docs/implementation/validation/evidence/winui3/windows-11-x64
PS> python tools/run_vectors.py windows-11-x64 modern --exe build/windows-modern/net8-x64/Loomora.WinUI.exe --evidence-dir docs/implementation/validation/evidence/winui3/windows-11-x64
```

The launcher `WindowsModernLauncher` copies the published folder to a scratch folder, starts `Loomora.WinUI.exe --probe --probe-input <text>` once per vector (serially, one WinUI process at a time, through an argument list), reads the probe line from standard output (a pipe), compares it with `loomora-probe.json` next to the executable and runs the checks of `tools/check_probe.py`. Result for the net10 and for the net8 folder: 15 valid vectors run, 15 passed, 16 error or API-only vectors skipped, two truncated to 1,000 UTF-16 units (limit_4096_ascii, limit_4096_two_byte_chars), about 40 s each. The net8 folder ran on the .NET 8.0.34 runtime (`rollforward-net8-resolution.txt`). The `control_characters` vector (`a<TAB>b<LF>c<SOH>`) needs the field to accept a line break set by the program: a single-line WinUI `TextBox` drops everything after it (measured: 14 of 15 passed with `AcceptsReturn` off), so the field has `AcceptsReturn` on, the Enter key is swallowed so that it behaves as a single-line field, and WinUI's lone carriage return is converted back to the line feed.

`python tools/verify.py quick` includes `run_vectors windows modern` when `python tools/build.py windows modern` has left `build/windows-modern/net10-x64/` (it does not build it: the build needs Visual Studio, the .NET SDK and NuGet).

### C.4 Probe line, memory, time to the first frame

```
PS> python tools/trials/windows-11-x64/winui-trial.py series build/windows-modern/net10-x64/Loomora.WinUI.exe --runs 10 --out <file>
```

Ten runs of `Loomora.WinUI.exe --probe` from a scratch copy: all exit status 0, no `Loomora.WinUI.exe` process left after any run, every line valid (`greeting_matches` true, `platform_id` `windows-11-x64`, `ui_variant` `modern`, `compiler` `msvc 195237029`, `disabled_features` empty, `eol_warning_shown` false). `display_ms` (from the creation of the process, so it includes the start of the .NET runtime and of the Windows App Runtime, to the first frame drawn after `Activate`): 2053 (the first run after a build, cold), then 365 to 545; median 436.5. `peak_rss_kb` 112,696 to 112,952. The probe line goes to standard output (a pipe in the vectors run; when the process has no standard output handle, as when started from a console without redirection, the program attaches to the parent console with `AttachConsole` and writes to `CONOUT$`: `python tools/trials/windows-11-x64/winui-trial.py console <exe> --out <file>` starts it from a process that owns a console and reads the line back from the screen buffer, `probe-attachconsole.txt`) and to `loomora-probe.json` next to the executable. The Legacy UI measures from `WinMain`, this one from the process start: the two numbers are not comparable.

### C.5 The window: typing, screenshots, themes

```
PS> powershell -NoProfile -ExecutionPolicy Bypass -File tools/trials/windows-11-x64/winui-typing.ps1 -Exe <abs path>\Loomora.WinUI.exe -Text Marie
PS> python tools/trials/windows-11-x64/winui-shoot.py <exe> <png>                                  # default start
PS> python tools/trials/windows-11-x64/winui-shoot.py <exe> <png> --probe-input "Zoé € Ελ 日本"    # inside the 2 s probe window
PS> python tools/trials/windows-11-x64/winui-shoot.py <exe> <png> --dark                          # dark theme, registry value restored afterwards
```

`winui-typing.ps1` starts the UI normally, reads the field and the greeting through UI Automation (`Coucou` / `Bonjour Coucou`), focuses the field, selects its content (Ctrl+A) and types `Marie` with `SendKeys` (keyboard events, not a value set through UI Automation): the greeting reads `Bonjour Marie` 177 to 189 ms after the keys were sent, including the time `SendKeys` takes; the window closes with `WM_CLOSE` and the exit code is 0 (`typing-uia.json`). `winui-shoot.py` captures the window with `PrintWindow` and crops it to the visible frame. The two screenshots stored (owner's authorization: at most two, one per rendering scenario) show the default start and the mixed Unicode input `Zoé € Ελ 日本`: accent, euro sign, Greek and CJK are drawn, no missing-glyph box. A dark-theme capture was taken, looked at and NOT stored: white text on `#202020`. The contrast of the greeting text measured on the captured pixels is 15.68:1 (light, `#1a1a1a` on `#f3f3f3`) and 16.29:1 (dark, `#ffffff` on `#202020`), both above 4.5:1 (`contrast-measure.txt`). The colours are theme resources only (`ApplicationPageBackgroundThemeBrush`, `TextFillColorPrimaryBrush`), so a high contrast theme applies by itself, but no high contrast theme was switched on.

### C.6 RollForward of the .NET 8 build

`net8` has `<RollForward>Major</RollForward>` (only for that target framework). The measurements (`winui-trial.py rollforward` and `resolution`, a private .NET root made for the child process only through `DOTNET_ROOT`, no global setting touched; evidence `rollforward-net8-*.txt`):

- Machine with .NET 8.0.34 and 11.0.0 (the host): `Major` loads 8.0.34; `DOTNET_ROLL_FORWARD=LatestMajor` loads 11.0.0 although 8 is installed, which is why `LatestMajor` was NOT kept.
- Private root with only .NET 10.0.15, and with only .NET 11.0.0: the net8 build with `rollForward` `Major` starts and writes a valid probe line (exit 0); without `rollForward` (the default policy) the apphost shows the message "You must install or update .NET to run this application" (Framework Microsoft.NETCore.App 8.0.0) and does not start; with `DOTNET_ROLL_FORWARD=LatestMajor` it starts.
- So the .NET 8 build starts on a machine that has only .NET 10 or only .NET 11 and uses .NET 8 where it is installed. Windows 10 1809 itself was not available: nothing is known about it.

The apphost of a `WinExe` shows a message box on a framework error: the trial script reads its text and ends the process.

### C.7 Spike S8, WinUI half

WinUI 3 (C#, `LibraryImport`, no C++/CLI and no C++/WinRT) calls the C++11 `loomora.dll` built by MSVC: the six functions are imported with `LibraryImport` (`ui/modern/windows-winui3/Native.cs`), the ABI version is checked at start, the greeting, the end-of-life test and the probe record come from the DLL, and 15 of 15 reference vectors pass on Windows 11 x64 for the net8 and the net10 build. Recorded in `validation/spikes.json` (S8).

### C.8 The .NET SDKs

`global.json` pins the SDK 10.0.404 (`latestPatch`): both target frameworks are built by it (net8 and net10 reference packs come with the SDK or from NuGet). The SDK 11.0.100 builds the project as well when `global.json` is not there: see the verdict notes for the result of that check. The .NET 11 runtime ran the net8 build (rollForward).

### C.9 What the verdict claims, and what this procedure does not cover

Covered: the x64 build of the net10 and net8 folders on this Windows 11 host (15 of 15 vectors each, 10 probe runs of net10, the typing run, two screenshots, light and dark contrast, the roll-forward measurements). Not covered: Windows 10 1809 or any Windows 10 (no machine: no verdict, the net8 fallback is declared in `validation/fallbacks.json` with no platform), ARM64 execution (built only, no verdict: there is no platform record for an ARM64 machine), the end-of-life InfoBar (this system is not end of life, so `eol_warning_shown` is false in every run and the InfoBar was never seen on screen), high contrast, DPI above 100%, the 1,000-character limit by keystrokes, other machines.

### C.10 What to record, and where

As A.9: evidence in `validation/evidence/winui3/windows-11-x64/`, the verdict `validation/verdicts/windows-11-x64-modern.json`, the deliverable records written by `tools/build.py windows modern` (`windows-11-x64-modern.json`, and `windows-arm64-modern.json` for the ARM64 build, which has no platform record yet). A rebuild of the net8 zip changes its sha256 by a few bytes: a verdict that names it must be updated with it.
