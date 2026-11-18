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

## M.9 Fallback: VMware Workstation Pro with the Unlocker — written 2026-11-16, not run

Why: the QEMU/KVM route of M.5 and M.8 failed twice (black screen, no cause found). ADR 0019 keeps VMware Workstation with the macOS Unlocker as the fallback; the owner chose it on 2026-11-16. Driven by `tools/macvmw.py` on the Windows host. **Nothing below the installation of VMware was run by an agent.**

Facts read on 2026-11-16 (sources: Broadcom pages, `DrDonk/unlocker` README and releases API):

- VMware Workstation Pro is free for all uses since March 2025, no key. Release **26H1u1** (3 September 2026, build 25688693, shown as 26.0.1 on disk). The download needs a Broadcom account (portal login, owner only). Workstation does not officially list macOS guests (Fusion does), hence the Unlocker.
- **Unlocker v4.2.8** (published 2026-05-14, `unlocker428.zip`, 24,748,614 bytes, sha256 `7ec212696981a95f7bbf9b04b7d33e644a8a808c88a39db897b16487be456af4`, matched by the owner's download). It patches `vmware-vmx` and `vmwarebase.dll`, ships the macOS VMware Tools (`iso/darwin.iso`), supports Workstation Pro 16/17/25/26 on Windows and Linux, **cannot add new macOS versions or AMD support**, and is "considered complete" by its author (alternative named by the author: OC4VM, which does not patch VMware; not evaluated here). A VMware update overwrites the patches: run `unlock` again. The README inside the zip is older than the one on the repository (it names Workstation 16/17 only).
- Not verified: that macOS **Tahoe** can be selected or booted (the Unlocker says it cannot add new macOS versions); that macOS boots when a hypervisor is active on the host. This PC has one (`HypervisorPresent` is `True`: WSL2 / Hyper-V), so VMware runs in Windows Hypervisor Platform mode, which Broadcom says can be slower and does not support nested VMs, RTM/HLE or PKU; whether the patched `vmware-vmx` boots macOS in that mode is unknown. If it does not, the options are to turn the Windows hypervisor features off for the session (this stops WSL2) or to test OC4VM.

Steps (D is `LOOMORA_VMW_DIR`, here `C:\t\loomora-macosvm`; `set LOOMORA_VMW_DIR=C:\t\loomora-macosvm` in the shell):

1. Owner, once: install VMware Workstation Pro (silent form documented by Broadcom: `VMware-Workstation-Full-<version>.exe /s /v"/qn EULAS_AGREED=1"`, not used here), download `unlocker428.zip` into D. `python tools/macvmw.py check` must show every PASS (the hypervisor and admin lines are information).
2. `python tools/macvmw.py unpack` verifies the sha256 and extracts to `D\unlocker` (nothing is run).
3. Close VMware, open an **elevated** PowerShell, `python tools/macvmw.py unlock`. Then, still elevated, `python tools/macvmw.py check` prints the patch status of `check.exe`. To undo: `relock`.
4. Recovery image (about 1 GB, owner's order): in WSL, `LOOMORA_VM_DIR=/mnt/c/t/loomora-macosvm/dl python3 tools/macvm.py fetch tahoe --yes-download`, which leaves `D\dl\tahoe\install.img` (checked against its chunklist by the tool).
5. `python tools/macvmw.py vmx tahoe --install D\dl\tahoe\install.img` creates `D\vms\tahoe\` with an 80 GB growable disk (`vmware-vdiskmanager`), `install-flat.img` (a hard link to `install.img`, no extra space on the same volume) and `install.vmdk` (a flat descriptor with a relative extent name; an absolute path made VMware refuse the disk: "file name syntax is incorrect") and `tahoe.vmx` (EFI, `smc.version = "0"`, SATA, e1000e, 8 GB, 4 vCPU). `--guest-os darwin24-64` if VMware shows the guest as "other" with the default `darwin25-64`.
6. Open `tahoe.vmx` in VMware, start it. Whether VMware's EFI boots the recovery disk is **not verified**. If it boots: Disk Utility, erase the 80 GB disk as APFS, quit Disk Utility, Reinstall macOS Tahoe on it (same steps as M.5; the installer downloads the rest from Apple). Then Remote Login, Xcode 26.x from the owner's Apple ID, power off, take a VMware snapshot named `clean`. In the guest: `Install VMware Tools` (the Unlocker's `darwin.iso`).
7. Reaching the guest from the host: VMware NAT, no port forward configured by the tool; use the guest's IP (`ifconfig` in the guest) with `ssh`.

Records: if this runs, every platform record and verdict must say "VMware Workstation 26H1u1 with Unlocker 4.2.8 on a PC, not Apple hardware" (ADR 0019 item 6), and the failures are recorded here like M.8.

### M.9b First run (2026-11-16): the VM is rejected at power-on

Host: Windows 11, VMware Workstation Pro 26.0.1 (build 25688693), Unlocker 4.2.8 applied (`check.exe`: `Patched (1)` on the four files). `HypervisorPresent` is `True`, so VMware runs on the Windows Hypervisor Platform (`WinHvPlatform.dll` loaded, "Hyper-V detected by CPUID").

- VMware accepts `guestOS = "darwin25-64"` (macOS 26): "Powering on guestOS 'darwin25-64'", `darwin.iso` selected as the Tools image.
- `vmrun start` then fails with `Exception 0xc0000005 (access violation)` in `vmware-vmx.exe` (`unrecoverable error: (vmx)`), a few lines after the log prints "Host VT-x Capabilities" with every field at 0 (VMCS revision ID 0, no EPT, no unrestricted guest): the VT-x capability MSRs are not exposed in this mode.
- Control: a 256 MB `otherlinux-64` VM without disk powers on normally on the same host (`poweredOn`), so VMware itself works in WHP mode; the crash is specific to the macOS guest.
- Not proven: that the hypervisor is the cause. It is the only difference found; the test that would settle it is to start the VM with the Windows hypervisor off (`bcdedit /set hypervisorlaunchtype off`, reboot, which stops WSL2 and Docker; `auto` and a reboot restore it). Not done yet: it is the owner's call.

**Outcome (2026-11-16, owner's decision).** The owner refuses to turn the Windows hypervisor off (WSL2 and Docker are needed), so the VMware route stops here: no macOS VM runs on this PC, by QEMU/KVM (M.8) or by VMware (M.9b). Remaining ways, not tried: OC4VM (no patched VMware; whether it works on a WHP host is unknown); the hosted `macos-26` runner for the Modern UIs (T126); Apple hardware or a Mac in the cloud for the Legacy UIs (Xcode 5.1.1, macOS 10.6), which no hosted runner covers. The Legacy Apple trials stay `not_tested` until one of them is available.


## M.10 VirtualBox route: `mavericks` (OS X 10.9.5) on a Windows host — in progress, written 2026-11-17

Why: the owner chose VirtualBox 7.2.22 for the 10.9.5 installer (the VMware route M.9b is stopped). On 2026-11-17 the owner accepted that **Hyper-V is turned off** for VirtualBox to run, which stops WSL2 and Docker Desktop while it is off. This reverses the owner's refusal recorded in M.9b. The steps are scripted in `tools/macvm-virtualbox.ps1` (branch `feat/macos-vm-virtualbox`, not merged), run from an elevated PowerShell on the Windows host:

- `-Mode Disable`: saves the current settings once to `E:\data\vm\osxmavericks\hyperv-state.json`, then turns off `Microsoft-Hyper-V-All`, `VirtualMachinePlatform`, VBS, memory integrity, Fast Startup and the hypervisor launch at boot. A full restart is needed (Restart, not Shut down). WSL distributions are not changed.
- `-Mode Enable`: puts the saved settings back, including WSL versions. A full restart is needed.
- `-Mode Create`: creates the VM in `E:\data\vm\osxmavericks` (`OSXMavericks`, Mac OS X 10.9 Mavericks 64-bit, EFI 64, ICH9, 4 GB RAM, 2 vCPU, 40 GB dynamic VDI, VMSVGA 128 MB, Mac identification keys) and starts it only if the Windows hypervisor is off and the VM is not already running.

Facts found on the host (2026-11-17):

- The installer file `OSXMavericks10.9.5.iso` (6,500,020,736 bytes) is **not** an ISO 9660 image: it has no `CD001` signature and starts with an `EFI PART` GPT header. VirtualBox refuses it as a DVD (`VERR_NOT_SUPPORTED`). `Create` converts it to `OSXMavericks-installer.vdi` (about 6.2 GB) and boots from disk. The original file is kept.
- After `Disable` and a full restart, `HypervisorPresent` was still `True` the first time: the Hyper-V features were enabled and VBS was on. After a second full restart it was `False`.
- Hyper-V must be off: `Create` refuses to start the VM while `HypervisorPresent` is `True`.
- A first `Create` attempt failed on the ISO and its partial VM was removed by the script (the `unregistervm --delete` path). The `Windows 2000` VM was not touched.
- WSL2 needs the hypervisor: `wsl --set-version ... 2` did not return while the hypervisor was off, and the command was stopped. The distributions read as version 2 in the registry afterwards; `wsl -l -v` read version 1 after the restart. The version and the default are to be checked before WSL is used again.

Installation (owner, 2026-11-17):

- The installer shows "This copy of the Install OS X Mavericks application can't be verified". Cause: the installer's signing certificate has expired and the VM clock is in 2026. Fix, in the installer's **Utilities > Terminal**: `date 1015120013` (15 October 2013, 12:00), then quit Terminal and continue. The owner reports that this lets the installation go on.
- Not verified yet: that the installation completes, that the guest boots to the desktop, and that Xcode 5.1.1 is installed. This procedure is not a validation record: no verdict is written from it.

Records: any platform record from this VM must say "VirtualBox 7.2.22 with Mac OS X 10.9.5 on a PC, not Apple hardware" (ADR 0019, item 6), and the date fix is part of the record.

## M.11 Apple Legacy, consolidated record (2026-11-17) — compiled, NOT RUN

Decision (owner, 2026-11-17): the macOS Legacy floor is **10.7 Lion**, not 10.6. Reason: C++11 needs libc++, and the clang of Xcode 4.6.3 refuses `-stdlib=libc++` below 10.7 (`invalid deployment target for -stdlib=libc++ (requires OS X 10.7 or later)`). The iOS Legacy floor stays **iOS 5.0**. Sources read on 2026-11-17: the Qt commit that moved its macOS floor to Lion for libc++ (`qtbase` commit `3d0a60aa`, on gitlab.linphone.org) and the MacPorts wiki page `LibcxxOnOlderSystems` (libc++ on 10.6 only through a separately built libc++). Neither gives the exact clang message, which was observed in the VM.

### The virtual machine (host: Windows 11, VirtualBox 7.2.22)

- Hyper-V, VirtualMachinePlatform, VBS, memory integrity, Fast Startup and the hypervisor launch at boot are off (`create-osx-mavericks.ps1 -Mode Disable`; saved settings in `E:\data\vm\osxmavericks\hyperv-state.json`). Consequence accepted by the owner: WSL is at version 1 and Docker Desktop does not run while VirtualBox is used. `-Mode Enable` puts everything back.
- VM `OSXMavericks` (`E:\data\vm\OSXMavericks\`): Mac OS X 10.9.5, EFI 64, ICH9, PS/2 keyboard, USB tablet (later PS/2 mouse for the Tahoe VM), 4096 MB, 2 vCPU, VMSVGA 128 MB, 40 GB dynamic disk. Mac identity `MacBookPro11,3` with the SMC keys of the tool.
- The installer `OSXMavericks10.9.5.iso` (6 500 020 736 bytes) is **not** ISO 9660: it is a raw GPT disk (`EFI PART`). It is converted to `OSXMavericks-installer.vdi` and booted as a disk.
- Installation: the installer refused with "can't be verified" because the VM clock was in 2026 and the installer certificate has expired. Fix: Utilities > Terminal, `date 1015120013`, then continue.
- SSH into the guest (for the builds): NAT forward host 127.0.0.1:2222 to guest 22 (`VBoxManage controlvm OSXMavericks natpf1`), Remote Login on. The 10.9 sshd (OpenSSH 6.2) needs `KexAlgorithms diffie-hellman-group14-sha1`, `Ciphers aes128-ctr`, `MACs hmac-sha1`, `HostKeyAlgorithms +ssh-rsa`, `PubkeyAcceptedAlgorithms +ssh-rsa`, and an RSA key (ed25519 is not known to OpenSSH 6.2). The host alias `loomora-mavericks` is in the user's SSH config. Snapshot `before-ssh-xcode` taken before the SSH set-up. Passwords are not written in any file.
- Sources copied to the guest: `core/` and `ui/legacy/` from the branch (`git archive`), into `~/loomora`.

### Xcode and SDKs in the guest

| Xcode | Compiler | SDKs | Use |
|---|---|---|---|
| 4.6.3 (`Xcode_4.6.3.dmg`, 1 633 MB), in `/Applications/Xcode.app` | Apple LLVM 4.2 (LLVM 3.2), build 4H1503 | MacOSX10.7, MacOSX10.8, iPhoneOS6.1, plus MacOSX10.6.sdk copied from 4.3.3 | Tried for C++11: refused (see below) |
| 4.3.3 (`Xcode_4.3.3_for_Lion.dmg`, 1 884 MB), mounted, not installed | Apple clang 3.1 (LLVM 3.1) | MacOSX10.6.sdk, MacOSX10.7.sdk | Source of the SDKs, and a C++0X test |
| 4.2 (`installxcode_421_lion.dmg`, 1 724 MB) | pkg installer, not opened | not checked | Not used |
| **5.1.1** (`Xcode_5.1.1.dmg`, 2 156 MB), in `/Applications/Xcode-5.1.1.app` | Apple LLVM 5.1 (LLVM 3.4) | MacOSX10.7 (copied from 4.3.3), MacOSX10.8, MacOSX10.9, iPhoneOS7.1 | Used for macOS 10.7 and iOS 5.0 |

The licence is accepted with `defaults write /Library/Preferences/com.apple.dt.Xcode IDEXcodeVersionForAgreedToGMLicense <version>` (for 4.6.3 and 5.1.1). Xcode 4.6.3 lists the 10.6 SDK once it is copied from 4.3.3 (`xcodebuild -showsdks`). Xcode 5.1.1 does not contain the 10.6 SDK. Compatible Xcode versions per floor:

- macOS 10.7 floor: Xcode 5.1.1 (C++11 and libc++ compile; SDK 10.7 copied in).
- macOS 10.6 floor: not reached. Xcode 4.6.3 refuses libc++ below 10.7; 4.3.3 was not tested with libc++.
- iOS 5.0 floor: Xcode 5.1.1 with the iPhoneOS7.1 SDK (deployment target 5.0; armv7 and armv7s; arm64 is set to 7.0 by Xcode, which is normal). Xcode 4.6.3 has the iPhoneOS6.1 SDK, which also allows 5.0, but without arm64 and with an older C++ compiler.

### C++11 tests (what each compiler accepts on `core/src`)

| Compiler | Target | Result |
|---|---|---|
| Xcode 4.6.3, `-stdlib=libc++` | 10.6 | Refused by the driver: `requires OS X 10.7 or later` |
| Xcode 4.6.3, `-stdlib=libstdc++` | 10.6 | Fails: no `<cstdint>`, no `std::u32string`, no inheriting constructors |
| Xcode 4.6.3, `-stdlib=libc++` | 10.7 | Fails: `inheriting constructors are not supported` (`errors.hpp`) |
| Xcode 4.3.3 (clang 3.1), `-std=c++0x` and `-std=c++11` | 10.7 | Fails: `utf8.cpp:22`, braced initialisation `Lead{...}` inside a conditional expression |
| **Xcode 5.1.1, `-std=c++11 -stdlib=libc++`** | **10.7** | **6 objects, 0 error** |
| **Xcode 5.1.1, `-std=c++11 -stdlib=libc++`** | **iOS 5.0, armv7 and armv7s** | **6 objects each, 0 error** |
| **Xcode 5.1.1, `-std=c++11 -stdlib=libc++`** | **iOS 5.0, arm64** | **6 objects, 0 error** (this slice is minimum iOS 7.0) |

The C++0X approach suggested by a forum post (Xcode 4.2, "C++0X" with libc++) was not reproduced: the 4.x compilers tested fail on the code of `core/`. Xcode 4.2 was not opened (it is an installer package). The working C++11 compiler here is Xcode 5.1.1.

### Results

- **macOS Legacy, 10.7** (`~/build/macos-legacy-10.7-x5`): backend `libloomora.a` (C++11, libc++, x86_64) and the AppKit UI (`ui/legacy/macos-appkit`, Objective-C without ARC, manual retain/release) compiled and linked with `-lc++ -framework Cocoa -framework Foundation`. `LC_VERSION_MIN_MACOSX` = 10.7, sdk 10.7. **Not run**: the guest is 10.9.5, and running a graphical app needs a session.
- **iOS Legacy, 5.0** (`~/build/ios5-app/Loomora.app`): `libloomora.a` fat (armv7, armv7s, arm64) built from `core/` with `-miphoneos-version-min=5.0`; `xcodebuild` of `ui/legacy/ios-uikit/Loomora.xcodeproj` with Xcode 5.1.1, `-sdk iphoneos7.1`, `LOOMORA_LIB_DIR` pointing to that library: **BUILD SUCCEEDED**. `LC_VERSION_MIN_IPHONEOS` = 5.0 for armv7 and armv7s, 7.0 for arm64. Links `/usr/lib/libc++.1.dylib`. **Not run**: an iOS 5 device is needed, and whether iOS 5 ships libc++ is not verified.
- Incidents, for the record: a stuck `xcrun lipo` (Xcode 4.6.3 shim) blocked for 11 minutes; the fat library was rebuilt with the Xcode 5.1.1 `lipo`. The archiver of Xcode 4.6.3 cannot archive arm64 objects: use the archiver of the Xcode that compiled them.

### The CI workflow (`.github/workflows/apple.yml`)

- YAML parses (PyYAML). The scripts named by the jobs exist (`tools/build.py`, `tools/ci/apple-setup.sh`, `tools/ci/ios_simulator.py`, `tools/run_vectors.py`).
- The `macos-legacy` job, `tools/build.py macos legacy` and `tools/report.py` used 10.6 as the floor: changed to **10.7** on branch `feat/apple-legacy-10-7` (commit `ecee802` in the main repository), with the matching comments and step names.
- Not checked: whether Xcode 26 accepts `-mmacosx-version-min=10.7` with libc++ on `macos-26`. The job has not run (no push yet). If Xcode 26 refuses it, the job fails at BUILD, which is the expected first-run failure.
- iOS 5 is not in CI: the hosted `macos-26` runner has no Xcode 5.1.1. The iOS Legacy job builds arm64 at iOS 12 with Xcode 26 (the project's own deployment target is overridden). The iOS 5.0 result above is local only.
- The workflow header still says "written without a Mac, never run"; the push that would run it awaits the owner's approval.

### Still open

- A run of the macOS 10.7 and iOS 5.0 binaries on real hardware (or on Mavericks for the macOS one).
- The C++11 libc++ question on an iOS 5 device.
- The macOS 10.6 floor: no compiler found here accepts libc++ below 10.7 without a separately built libc++.
