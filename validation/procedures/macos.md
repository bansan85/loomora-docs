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

At the OpenCore picker choose the Base System, erase the disk as APFS in Disk Utility, install (it may sit on the Country Selection screen for a while, per the OSX-KVM README), enable Remote Login, then `stop`, `snapshot tahoe clean`, and delete `install.img` and `BaseSystem.dmg`. Xcode 27 needs Apple silicon: do not install it here. Not verified: that Tahoe boots with this OSX-KVM revision; the speed of the iOS simulator without a GPU.

## M.6 Run a build and the vectors (when a VM exists) — not written

Tasks T047 to T052 and T078 to T079 add the `macos-*` and `ios-*` targets to `tools/build.py`; `tools/run_vectors.py` will need a launcher that copies the build to the guest with `scp -P <port>` and runs it over `ssh -p <port>` (the guest has no other route: ports forwarded on `127.0.0.1` only). Neither exists yet. When they do: record the platform (`validation/platforms/`), the build, the vectors run and the verdict, each saying "QEMU/KVM on a PC under WSL2, not Apple hardware".

## M.7 Not covered

Real Apple hardware, Apple silicon, iOS devices, macOS 10.7, 10.13 and 11+ ladder entries (the PoC scope rule asks only for the lowest testable Legacy system and one Modern system), iOS 5 to 12.
