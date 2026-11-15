# Procedure: where the old systems and tools come from (offline installation routes)

Task T095 asks for the offline installation route of every platform without Internet access. This page records, platform by platform, where the system or the tool came from and how it was brought to the machine. What this project does not know is written as such: nothing below is a verified fact unless it says so.

## Status of this document

- Written on 2026-11-15 from the records of the project and from the facts given by the owner the same day.
- **No verdict is `ko`** (every verdict is `ok_restricted` or `not_tested`), so there is no cause or `attempts[]` to write for a failed platform (first half of T095).
- Sources and licences were NOT verified by the project: no checksum of an image or installer was compared with a vendor value, and no licence was read. Whoever replays the trials must check them.

## Windows 2000 (platform `windows-2000-x86`)

- **System image.** A VirtualBox virtual machine of Windows 2000 Professional taken from <https://archive.org/details/windows2000virtualbox>. The owner obtained it; the project did not download it. Its checksum was not compared with any published value and its licence was not verified by the project.
- **The VM.** VirtualBox 7.2.22, VM named `Windows 2000`, NAT network, one snapshot named `Claude Firefox 10` (owner's statement of 2026-11-15; `procedures/windows.md`, part B.1, describes the two snapshots named `Claude` seen on 2026-11-14, so the list of snapshots is to be re-read with `VBoxManage snapshot "Windows 2000" list` before relying on this sentence). The guest facts (build 2195, no service pack, 168 MB RAM) are in `validation/platforms/windows-2000-x86.json`.
- **Files into the guest.** The guest has no usable modern Internet access (an old system with no current TLS), so every file reaches it from the host: `VBoxManage guestcontrol "Windows 2000" copyto --username name --password "" <host file> "<guest path>"`, with the SHA-256 of the copy checked in the guest; the exact commands are in `procedures/windows.md`, part B. Results come back with `copyfrom`. The Windows Legacy UI is cross-compiled outside the guest, so the guest needs no compiler.
- **Firefox 10.0** (build ID 20120129021758). The installer was supplied by the owner. **Source of the installer (owner, 2026-11-15): <http://download.mozilla.org/?product=firefox-10.0&os=win&lang=en-US> (Mozilla download redirector, English US build).** The file name, SHA-256 and licence are not recorded; to be filled by the owner. The browser is installed in `C:\Program Files\Mozilla Firefox\` and the page is loaded over HTTP from the host through the NAT (`http://10.0.2.2:<port>/`), see `procedures/web.md`.

## Linux Legacy (platforms `linux-centos-5.11-x86_64` and `linux-centos-6.10-x86_64`)

- **Images.** Containers of the PyPA images `quay.io/pypa/manylinux1_x86_64` (CentOS 5.11) and `quay.io/pypa/manylinux2010_x86_64` (CentOS 6.10), pulled from quay.io with Docker Desktop. The pull needs the Internet; an offline replay needs the images saved with `docker save` on a connected machine and loaded with `docker load`.
- **Packages.** The images ship an old compiler (devtoolset); the extra packages (Xft, fontconfig development files, Xvfb, fonts) come from the CentOS vault mirror `archive.kernel.org/centos-vault/` (the repositories of CentOS 5.11 are already set to `http://archive.kernel.org/centos-vault/5.11/`, http only; for CentOS 6.10 the 403 of `vault.centos.org` is worked around with `archive.kernel.org`). The exact commands are in `procedures/linux.md`, parts L.10 (CentOS 6.10) and L.11 (CentOS 5.11). An offline replay needs a local copy of that mirror.

## Android (platforms `android-4.1.2-x86` and `android-7.0-x86_64`)

- The Android SDK, the system images (API 16 x86, API 24 x86_64), the emulator, NDK r15c (Legacy) and NDK r30 (Modern), Gradle and the Compose libraries come from the vendor download pages and package repositories named in `procedures/android.md` (parts A and E, tools and pinned versions). They were installed with Internet access; nothing is repeated here. An offline replay needs the SDK and NDK folders and the Gradle and Maven caches copied from a connected machine.

## Web build tools

- The current Emscripten (`~/emsdk`) and the Emscripten 1.39.20 fastcomp (`~/emsdk-fastcomp`) come from the emsdk tags named in `procedures/web.md` and spike S5; Node and TypeScript as named there. Same remark: installed online, nothing repeated.

## Not covered

- Windows XP, Vista, 7, 8.1, 10, ARM64, macOS, iOS and the other rows of the version ladders were not run, so they have no installation route. macOS and iOS stay pending.
