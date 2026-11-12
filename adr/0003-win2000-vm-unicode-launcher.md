# ADR 0003: Start the Legacy UI in the Windows 2000 VM with a native Unicode helper

- **Status**: accepted
- **Date**: 2026-11-12
- **Spike / requirement**: S1 (T066) / FR-027, SC-011, FR-016

## Context

The Windows 2000 trial runs in a VirtualBox 7.2.22 guest (Windows 2000 build 2195, no service pack, Guest Additions 6.1.18) that is driven from the Windows 11 host with `VBoxManage guestcontrol`. `tools/run_vectors.py` must give `loomora.exe` the exact text of every reference vector (SC-011: accents, CJK, right-to-left, emoji, combining marks, quotes, backslashes, the empty string, 1,000 characters). Measured on that guest:

- `guestcontrol run` does not pass command-line arguments to the guest program (the program comes up without them); only an `.exe` or a `.bat` with no arguments can be run, and its standard output comes back.
- Windows Script Host 5.1 (`WshShell.Run`, the only way to start a program from a script on this system) converts the command line to the ANSI code page (1252 here): every character outside it became `?` (one per UTF-16 unit: the CJK vector gave `????????`, an emoji `??`) and `%PATH%` was expanded to the value of the variable. A script file in UTF-16 does not help, the loss is in `Run`. `WshShell.CurrentDirectory` does not exist in WSH 5.1 (measured: runtime error) and `Exec` was only added in WSH 5.6.
- The system has no `tasklist` or `taskkill`; leftover processes can be counted with WMI or the Toolhelp API.

## Decision

Each run copies `cmdline.txt` (the whole command line as UTF-16LE with a byte order mark) into a guest folder and runs a fixed `run.bat`, which starts `tools/guest/runw.exe`. `runw.c` (about 5 KB, `kernel32.dll` only, no C runtime, built with the MinGW-w64 i686 compiler by `tools/guest/build_runw.sh` in WSL under `$HOME`) reads the file without any line parsing, calls `CreateProcessW` with that exact string, waits, and prints the exit status, the first line of `loomora-probe.json` and the number of `loomora.exe` processes still alive (which it terminates). The host side (`WindowsVirtualBoxLauncher` in `tools/run_vectors.py`) quotes each argument with the rules of the C runtime that `probe.cpp` applies in reverse. `runw.exe --nowait` leaves a window open on the desktop for screenshots and typed input, `runw.exe --kill` cleans up after a timeout.

The launcher takes the VM name, the guest account and its password from `LOOMORA_VM_NAME`, `LOOMORA_VM_USER` and `LOOMORA_VM_PASSWORD` (a non-empty password goes through a temporary `--passwordfile`, never on a command line), fails with a clear message if the VM is not running or the guest rejects the credentials, and never starts, stops, snapshots, restores or reconfigures the VM. Runs are serial.

## Rejected alternatives

- **A VBScript starting the program with `WshShell.Run`**: measured to lose every non-ANSI character and to expand environment variables (see Context).
- **A `.bat` carrying the text**: an ANSI file cannot hold the vectors.
- **WMI `Win32_Process.Create`, `Shell.Application.ShellExecute`**: no wait and no exit status, and the process would not necessarily be on the interactive desktop.
- **Hex- or base64-encoding the input and decoding it inside the product**: would add a test-only code path to the UI, which the probe contract (`--probe-input <utf8>`) deliberately avoids.

## Consequences

- Everything a Windows command line can carry reaches the executable unchanged: tabs, newlines, control characters, U+2028, a byte order mark, quotes, backslashes, `%`, emoji and 1,500-character texts were checked (25 hard texts plus the 15 reference vectors). Only a NUL is not representable, as on the local Windows launcher.
- A second small program is part of the test tooling and must be built (`tools/guest/build_runw.sh`); `run_vectors.py` builds it when `out/windows-legacy/guest/runw.exe` is missing or older than its source.
- The same launcher serves any further old Windows guest (XP, a Windows 2000 SP4 image) by adding an entry to `VBOX_PLATFORMS`; a guest with newer Guest Additions that passes arguments would not need the helper but would keep working with it.
- What stays unverified: the helper and the launcher were exercised on Windows 2000 build 2195 only (RTM, not SP4).
- A typed-input test through `VBoxManage controlvm keyboardputstring` assumes a US keyboard layout; on this guest (French AZERTY for the logged-on user) it typed `?qrie` for `Marie`, so the trial driver sends the scancodes of the physical keys instead.
