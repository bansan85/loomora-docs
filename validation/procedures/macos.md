# Procedure: macOS and iOS (virtual machines)

**Status: prepared on 2026-11-15, not run.** No VM exists, nothing was downloaded, nothing was built or run on macOS. This page is the plan the owner follows (decisions: ADR 0019); each step that has not been tried is marked so. When a step is run, replace "not run" by the result and the date, and write the verdict files.

Hosts and tools: QEMU/KVM in WSL2 (`Ubuntu-24.04`) on the Windows 11 machine of the trials, driven by `tools/macvm.py`. Every macOS system runs in a VM on a PC, not on Apple hardware; every record made from it says so (ADR 0019, item 6).

| VM | System | Xcode | Purpose | Tasks | Boot recipe |
|---|---|---|---|---|---|
| `tahoe` | macOS Tahoe 26 | 26.x | Modern macOS, iOS simulator | T047 to T052, T061 | OSX-KVM, OpenCore, `Skylake-Client` |
| `mavericks` | OS X 10.9 | 5.1.1 | build the Legacy iOS and macOS UIs | T067, T078, T079 | Chameleon, SeaBIOS, `Penryn` (not verified) |
| `snowleopard` | Mac OS X Server 10.6.8 | none | run the 10.6 floor | T067, T089 | Chameleon, SeaBIOS, `Penryn` (Proxmox runbook) |

iOS 5: no VM; `not_tested` unless a simulator or device appears (T090). Modern macOS and the iOS simulator are also built and run by a hosted `macos-26` runner (T126); the `tahoe` VM is optional.

## M.0 Rules of this procedure

- **No agent downloads anything.** Only the owner runs `fetch`, `apt` and any other command that uses the network, after reading `python tools/macvm.py plan` and `check`.
- Builds in WSL go under `$HOME/loomora-build/<name>` (CLAUDE.md); the VM disks go under `LOOMORA_VM_DIR` (M.2), never `/tmp`.
- One VM at a time (the machine gives 20 GB to WSL). Stop a VM with `macvm.py stop` before taking or restoring a snapshot.
- Screenshots are not evidence; ask the owner before adding an image to `docs/implementation` (CLAUDE.md).
- Evidence only when it proves something new (ADR 0014): the probe line and the vector log of a UI that has never run on that system.

## M.1 Host preparation (owner, one time; uses the network for `apt`)

State on 2026-11-15 (`python tools/macvm.py check`): `/dev/kvm` present and `vmx` exposed, AVX2 present; missing: access to `/dev/kvm`, `ignore_msrs=Y`, QEMU, `qemu-img`, `dmg2img`. Commands (from the OSX-KVM README, read 2026-11-15):

```
wsl -d Ubuntu-24.04
sudo apt install qemu-system-x86 qemu-utils dmg2img git      # download size not measured
sudo usermod -aG kvm $USER                                   # then: wsl --shutdown, and reopen
echo 1 | sudo tee /sys/module/kvm/parameters/ignore_msrs     # lost at each WSL restart
echo "options kvm ignore_msrs=Y" | sudo tee /etc/modprobe.d/kvm.conf
```

The persistent `kvm.conf` only takes effect if the WSL kernel loads `kvm` as a module; if it is built in, repeat the `echo 1` line at each start (not verified on this kernel). Check: `python tools/macvm.py check` must end with no FAIL except the disk line when the VM folder is on the distribution disk.

## M.2 Where the VMs live (owner, one time)

`C:` has 1.7 GB free and holds the distribution disk, so the VMs go on `H:` (about 103 GB free) in a VHDX that is native ext4 inside WSL. In an elevated PowerShell:

```
.\tools\macvm-vhdx.ps1 -Path H:\loomora-vm\macvm.vhdx -SizeGB 80                           # dry run: prints the plan
.\tools\macvm-vhdx.ps1 -Path H:\loomora-vm\macvm.vhdx -SizeGB 80 -Create -Format -Mount -Apply
```

then once in WSL: `sudo mkdir -p /mnt/wsl/macvm/vm && sudo chown $USER /mnt/wsl/macvm/vm` and `export LOOMORA_VM_DIR=/mnt/wsl/macvm/vm`. The mount is lost at each reboot or `wsl --shutdown`: `.\tools\macvm-vhdx.ps1 -Path H:\loomora-vm\macvm.vhdx -Mount -Apply`. The script was parsed and its dry run was read; `-Apply` was **not run**. 80 GB is a limit, not a use: the file grows with the data. Leave room on `H:` for the repositories and the Windows 2000 VM.

## M.3 `snowleopard` (Mac OS X Server 10.6.8) — not run

Media supplied by the owner (not downloadable: ADR 0019): the 10.6.x Server install disc as a raw image `install.img`, a Chameleon ISO `chameleon.iso` (the runbook used Chameleon 2.1 r2069), the 10.6.8 Server combo update. Copy them to `$LOOMORA_VM_DIR/snowleopard/`. The runbook's disc was 7,619,510,272 bytes, SHA-1 `1244e5e3afc73a2cba792f57dae54bb7f3a5575a` (10.6.3 Server); compare yours and record both.

```
python tools/macvm.py create snowleopard
python tools/macvm.py cmd snowleopard --install            # read the command first
python tools/macvm.py start snowleopard --install          # a WSLg window opens
```

At the Chameleon prompt choose the install disc and boot with `cpus=1 -v` (two vCPUs gave a spinlock panic on 10.6.3). In the installer: Utilities, Disk Utility, partition the virtual disk (24 GB here, the runbook used 64 GB) as GUID Partition Table with one Mac OS Extended (Journaled) volume, then install; the runbook says it took hours. Boot the installed system through Chameleon with `cpus=1 -v`, finish Setup Assistant, apply the 10.6.8 Server combo update, shut down cleanly, then enable Remote Login (ssh) in the guest. Then `python tools/macvm.py stop snowleopard` and `python tools/macvm.py snapshot snowleopard clean`. Remove `install.img` from the VM folder afterwards to free the space (the owner keeps the original).

Known from the runbook (Proxmox, not this tool): Chameleon must be on an IDE CD-ROM (SATA gave reset errors); the keyboard needs the EHCI controller; the display stays in plain VESA mode; the guest needs the `isa-applesmc` key; the `vmware-svga` display makes remote administration work after the update. Whether the `macvm.py` command reproduces this is not verified.

## M.4 `mavericks` (OS X 10.9 + Xcode 5.1.1) — not run

Same shape as M.3 with `mavericks`, 4 GB RAM, one vCPU, disk 32 GB. Media supplied by the owner: the 10.9 installer as a raw image, a Chameleon ISO, **Xcode 5.1.1** (needs OS X 10.8.4 or later and an Intel 64-bit CPU), and the **10.6 SDK** from an Xcode 3.x or 4.x (Xcode 5.1.1 does not ship it; one forum post reports copying it, which is not official guidance). Open points: that Chameleon boots 10.9 on QEMU (not verified); whether Xcode 5.1.1 can build and run an iOS 5 target (T065, T067). Build steps and verdicts: tasks T078, T079, T090; not written until the VM exists.

## M.5 `tahoe` (macOS Tahoe + Xcode 26.x) — not run, optional

Needs the network: OSX-KVM (about 170 MB), the Apple recovery image and the macOS install itself, then Xcode 26.x and the iOS simulator runtime from the owner's Apple ID (large; not measured). Check disk and allowance first (`python tools/macvm.py plan tahoe`), then, by the owner:

```
python tools/macvm.py fetch tahoe --yes-download
python tools/macvm.py create tahoe
python tools/macvm.py start tahoe --install
```

Run these commands in a Linux shell of the distribution (`cd /mnt/c/t/loomora`, `export LOOMORA_VM_DIR=/mnt/wsl/macvm/vm`), not from the Windows host: the relay of `macvm.py` forwards only `LOOMORA_HOST_DRIVE` (so `LOOMORA_VM_DIR` would not arrive) and QEMU, started in the background, may be killed when the relaying `wsl.exe` ends. The distribution is chosen on the Windows side by `LOOMORA_WSL_DISTRO` (default `Ubuntu-24.04`) and by `-Distro` of `macvm-vhdx.ps1`. In a command such as `wsl -d <distro> -- bash -c '...'` the `$VAR` of the string are expanded on the Windows side: put the commands in a `.sh` file instead.

Installation, as the owner did it (steps as given by the owner on 2026-11-16, not replayed by an agent):

1. `echo 1 | sudo tee /sys/module/kvm/parameters/ignore_msrs` (it goes back to `N` at each `wsl --shutdown`), then `python3 tools/macvm.py start tahoe --install`; a WSLg window opens.
2. At the OpenCore picker choose the Base System (the installer).
3. Disk Utility: select the big partition (the 80 GB QEMU disk), **Erase** it (APFS, the default), then quit Disk Utility.
4. **Reinstall macOS Tahoe**, choose the erased disk. The installer downloads the rest of the system from Apple (it may sit on the Country Selection screen for a while, per the OSX-KVM README). Keep the window open to the end and check the free space of the VHDX first (M.8).
5. Finish Setup Assistant, enable Remote Login, install Xcode 26.x from the owner's Apple ID, shut the guest down, then `python3 tools/macvm.py stop tahoe`, `python3 tools/macvm.py snapshot tahoe clean`, and delete `install.img` and `BaseSystem.dmg`.

Xcode 27 needs Apple silicon: do not install it here. Not verified: that Tahoe boots with this OSX-KVM revision; the speed of the iOS simulator without a GPU.

**Tahoe cannot build the Legacy UIs.** Current Xcode deploys to iOS 12 at the lowest (research.md, iOS section) and no longer builds the 32-bit slices, so the iOS 5 to 7 slices (`armv7`, `armv7s`) and the macOS 10.6 floor of the Legacy UIs need Xcode 5.1.1 in the `mavericks` VM (M.4); the `tahoe` VM is only for the Modern UIs. The exact minimum deployment target of Xcode 26 was not checked.

## M.6 Run a build and the vectors (when a VM exists) — not written

Tasks T047 to T052 and T078 to T079 add the `macos-*` and `ios-*` targets to `tools/build.py`; `tools/run_vectors.py` will need a launcher that copies the build to the guest with `scp -P <port>` and runs it over `ssh -p <port>` (the guest has no other route: ports forwarded on `127.0.0.1` only). Neither exists yet. When they do: record the platform (`validation/platforms/`), the build, the vectors run and the verdict, each saying "QEMU/KVM on a PC under WSL2, not Apple hardware".

## M.7 Not covered

Real Apple hardware, Apple silicon, iOS devices, macOS 10.7, 10.13 and 11+ ladder entries (the PoC scope rule asks only for the lowest testable Legacy system and one Modern system), iOS 5 to 12.

## M.8 Record of the first attempt (2026-11-15 and 2026-11-16, not successful)

Host: Windows 11 Pro, Intel Core i5-8300H (4 cores, AVX2, VT-x), WSL2 with Ubuntu 24.04 (kernel 6.18), QEMU 8.2.2, `/dev/kvm` read/write, `kvm ignore_msrs=Y`. VM folder on a 60 GB VHDX on a Windows drive (`tools/macvm-vhdx.ps1`).

- Recovery image of macOS Tahoe (OSX-KVM `fetch-macOS-v2.py`, identifier 140-93589): 960,530,321 bytes, verified against its chunklist (signature and the 92 chunk hashes, checked with `tools/macvm-verify-dmg.py`), converted to `install.img` (2.67 GB) with `dmg2img`. The image itself is not stored in the repository: it belongs to Apple.
- The installation was started in a QEMU window (WSLg), with an 80 GB sparse qcow2 disk. The guest wrote about 27 GB. The QEMU window closed at about 05:35 on 2026-11-16 (`Broken pipe` on the WSLg display); the cause of the stop and whether the installer had reached Disk Utility are not identified in the records.
- Problems met on the way, all fixed in the tools: `fetch-macOS-v2.py` writes to the current folder whatever `--outdir` says and needs a terminal for its verification (run through `script`); after the format, the VHDX stayed attached but unmounted until `wsl --shutdown` and a new `wsl --mount`.
- The VM disk was erased on 2026-11-16. The owner then planned to delete the VM folder (`C:\loomora-vm`) on this PC. Nothing here is a verdict: the macOS trials are still not run.

### M.8b Second attempt, host preparation and download (2026-11-16, in progress)

Host: the same PC with `Ubuntu-26.04` (kernel 6.18.58.1-microsoft-standard-WSL2, QEMU 10.2.1). The VHDX is `C:\t\loomora-vm\macvm.vhdx` (80 GB, ext4, mounted as `/mnt/wsl/macvm`, 71 GB free after the download); `C:` had 217.7 GB free.

- `macvm.py check`: all PASS except `ignore_msrs` (back to `N` after the `wsl --shutdown`), which only matters at the start of the VM.
- `fetch tahoe --yes-download` worked: `BaseSystem.dmg` is 917 MiB (960,530,321 bytes, the same as the first attempt), so the recovery image is about 1 GB, not the 20 to 30 GB that `macvm.py plan` estimated. `macvm-verify-dmg.py`: 92 chunks, signature and every hash verified. `install.img` is 2.5 GiB.
- `create tahoe`: `disk.qcow2` (80 GB virtual, sparse) and `OVMF_VARS.fd` created.
- **The installation failed (2026-11-16, owner's report).** The QEMU window went black during the installation phase or just after it, with about 45 GB written in the guest, and stayed black. The owner then erased the VM, so `qemu.log` is lost and the cause is **not identified**. Not known either: whether `kvm ignore_msrs` was `Y` at the start (it goes back to `N` at each `wsl --shutdown`, and was `N` when checked afterwards), whether QEMU was still running, and whether the WSLg window or the guest froze. Nothing here is a verdict.
- Next attempt: `macvm.py start` now refuses when `ignore_msrs` is not `Y` or the VM folder has less than 15 GB free (`--force` overrides), writes the command, the host state and QEMU's exit status to `qemu.log` and the guest errors to `qemu-debug.log`; `macvm.py status tahoe` and `macvm.py screenshot tahoe` (QEMU `info status`, a screen dump that says whether every pixel is black) tell a frozen guest from a frozen window; `--headless` avoids the WSLg window (VNC on `127.0.0.1:5901`). None of this was run against a real guest yet.

Lessons for the next attempt, on any PC: check the free space on the drive that holds the VHDX before the installation (the guest writes more than the 40 GB disk it sees), keep the window open to the end, and read `qemu.log` after each stop.
