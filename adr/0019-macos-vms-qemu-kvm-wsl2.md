# ADR 0019: The Apple trials run in macOS virtual machines (QEMU/KVM in WSL2) on the owner's machine

- **Status**: accepted (owner decisions of 2026-11-15: run macOS on non-Apple hardware, WSL2 as the host, free software only, no image downloaded without an explicit order); **prepared, not yet run**: no VM exists and nothing was downloaded when this record was written
- **Date**: 2026-11-15
- **Spike / requirement**: S7 (T065), S2 (T067) / FR-016, T047 to T052, T061, T078 to T090, T109, T126

## Context

The tasks of the Apple families need macOS: the Modern macOS and iOS UIs (Xcode 26, T047 to T052, T061), the Legacy UIs (Xcode 5.1.1 for iOS 5 to 7 slices and the macOS 10.6 floor, T078 to T090) and the 10.6 run of the self-test (T067). The project owner has no Apple hardware. The hosted `macos-*` runners of ADR 0018 build and run the Modern UIs without any local disk, but cannot run macOS 10.6 or Xcode 5.1.1.

Apple's licence allows macOS only on Apple-branded hardware (every source read on 2026-11-15 says so; none quotes the licence text, which is the owner's to read). **The owner decided on 2026-11-15, for private non-commercial use, to run macOS on a non-Apple PC knowing this**, and the records say so (below). This record does not argue the licence.

Constraints of the owner's machine, measured on 2026-11-15 (read-only): Windows 11 Pro 26H2, Intel Core i5-8300H (Coffee Lake, 4 cores, 8 threads, AVX2), 23.8 GB RAM, `.wslconfig` with `memory=20GB`, WSL 3.0.5.0; Ubuntu-24.04 in WSL2 with `/dev/kvm` present and `vmx` exposed. The distribution disk is an 82 GB `ext4.vhdx` on `C:`, which has **1.7 GB free**; `H:` has about 103 GB free. Inside WSL `df` shows 875 GB free (the virtual size), which is not the real limit. The owner's data allowance is limited and the owner asked that no image be downloaded by an agent.

## Decision

1. **Host: QEMU/KVM inside WSL2**, with the boot recipes of OSX-KVM (<https://github.com/kholia/OSX-KVM>) for modern macOS. Free software only. Hypervisor setup is the owner's (`apt`, `usermod`, `ignore_msrs`: `procedures/macos.md`).
2. **Three VMs, one running at a time** (host RAM 20 GB to WSL):
   - `tahoe`: macOS Tahoe 26 (the last macOS that runs on Intel) with Xcode 26.x, for the Modern macOS and the iOS simulator. Optional, because a hosted `macos-26` runner covers it (T126); Xcode 27 needs Apple silicon and cannot run in this VM.
   - `mavericks`: OS X 10.9 with Xcode 5.1.1 (which needs OS X 10.8.4 or later), to build the Legacy iOS and macOS UIs.
   - `snowleopard`: Mac OS X Server 10.6.8, to run the macOS 10.6 floor.
   iOS 5 has no VM: it stays `not_tested` unless a simulator or device appears (T090 already allows it).
3. **Where the disks live**: not in the distribution disk on `C:`. `tools/macvm-vhdx.ps1` creates an expandable ext4 VHDX on another drive and mounts it as `/mnt/wsl/macvm` (`LOOMORA_VM_DIR=/mnt/wsl/macvm/vm`). Not under `/mnt/<drive>` (DrvFs is slow for qcow2) and not in `/tmp`. The disks are sparse qcow2 files, so the virtual sizes (80, 32 and 24 GB) are limits, not the space used.
4. **One tool**: `tools/macvm.py` (Python, ADR 0015) with `check`, `plan`, `fetch`, `create`, `cmd`, `start`, `stop`, `snapshot`, `restore`, `ssh` and `--selftest`. Only `fetch tahoe --yes-download` uses the network, and it refuses without the flag. The guest ports are forwarded on `127.0.0.1` only (ssh 2222 to 2224, VNC `:1` to `:3` for headless). Unlike the VirtualBox launcher of `run_vectors.py`, which never starts or snapshots a VM, this tool owns its VMs: it starts and stops them and takes the snapshot `clean` of a stopped VM.
5. **No download by an agent.** The Apple recovery image, OSX-KVM, the `apt` packages, Xcode and every other medium are fetched by the owner (or by an explicit order of the owner), after `macvm.py plan` and `check` show the room. Media that cannot be downloaded (10.9 and Snow Leopard Server installers, Chameleon, Xcode 5.1.1, the 10.6 SDK from Xcode 3.x or 4.x) are supplied by the owner and described in `procedures/offline-install.md`, like the Windows 2000 image.
6. **Records are honest about the host**: every platform record and verdict of a macOS VM states that it ran in QEMU/KVM on a PC (Intel i5-8300H) under WSL2, not on Apple hardware, with the emulated CPU (`Skylake-Client` or `Penryn`). Both repositories are public.
7. **Evidence rules unchanged** (ADR 0014): a screenshot of a macOS desktop is not evidence unless the owner authorizes it; the probe lines and vector logs are.

## Rejected alternatives

- **VMware Workstation Pro with the macOS Unlocker** (free for personal use, runs directly on Windows, tested by its author with Workstation 16 to 26): not chosen by the owner, who prefers WSL2. Still the fallback if QEMU/KVM does not boot a guest.
- **VirtualBox** (already used for Windows 2000): Oracle calls macOS guests experimental and unsupported, and VirtualBox beside Hyper-V is slow (Oracle's known limitations).
- **QEMU natively on Windows with WHPX**: no report of OSX-KVM running there; QEMU documents WHPX as slower than KVM.
- **Only hosted macOS runners**: cannot run macOS 10.6 or Xcode 5.1.1; kept for the Modern UIs (T126).

## Consequences and what is not verified

- **Nothing was booted.** The QEMU command lines of `macvm.py` were assembled from `OpenCore-Boot.sh` of OSX-KVM and from a Proxmox runbook for 10.6 (<https://lightyearlabs.ca/blog/osx-server/snow-leopard-runbook.md>), both read on 2026-11-15, and only the generation of the commands is self-tested.
- Not verified: that macOS Tahoe boots with OSX-KVM (the menu of `fetch-macOS-v2.py` lists it, with no test claim, and the script header only names 10.13 to 10.15 as tested); that `OpenCore/OpenCore.qcow2` is where the script expects it (the repository listing did not enumerate it); that Mavericks boots with Chameleon on QEMU (the runbook covers 10.6.8 only; Snow Leopard is before the UEFI that OpenCore needs, and Parallels dropped it); that Xcode 5.1.1 has an iOS 5 simulator (not found); the download and disk sizes of every medium (estimates in `macvm.py plan`, not measured); the speed of the iOS simulator without GPU acceleration.
- **Host state on 2026-11-15** (`python tools/macvm.py check`): `/dev/kvm` present but not readable by the user (not in group `kvm`), `kvm ignore_msrs` is `N` (macOS guests reboot-loop without `Y`), QEMU, `qemu-img` and `dmg2img` not installed, 1.7 GB free on `C:`. WSL interop (`powershell.exe` from WSL) fails on this machine (`accept4 failed 110`), so the Windows drive of the distribution is passed in by the Windows side of the tool.
- The VMs use about 10 to 70 GB each once installed (estimates). `H:` has room for the Legacy pair and, if the owner wants it, Tahoe, but not for a second copy of anything.
- Open questions for the owner, not decided here: whether to build the Tahoe VM at all given the disk and the data allowance (the hosted runner covers Modern); where the Apple developer account media come from for Xcode 5.1.1 and the 10.6 SDK (spike S7, T065, still open).
