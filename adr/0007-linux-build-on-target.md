# ADR 0007: Linux binaries are built on the target machine

- **Status**: accepted (amends ADR 0001 rule 4, ADR 0004 finding 13, ADR 0005 and ADR 0006 on the points listed below)
- **Date**: 2026-11-14
- **Spike / requirement**: owner decision of 2026-11-14; removes spike S4 / FR-021, FR-022 (fallbacks), research R7

## Context

The Linux Legacy UI was built both on the distribution of the build machine and in the manylinux2010 containers (CentOS 6, glibc 2.12, GCC 8), with libstdc++ and libgcc linked statically there, to prove that the executable asks for no glibc symbol above 2.12 (spike S4). Several choices of the sources followed from that goal alone: `sscanf` and `strtol` replaced by hand-written parsers and `gettimeofday` kept (to avoid `__isoc23_*` / GLIBC_2.38 and librt), Xft and fontconfig opened with `dlopen` (so that no development file and no library was needed at build time), a static C++ runtime option in both Linux UIs.

The owner decided that a Linux binary is **always compiled on the final (target) machine**, never built on a recent OS and installed on an old one. The glibc baseline of a build host therefore does not matter.

## Decision

- **Removed**: the manylinux2010 build (the Docker code, the image digests and the glibc <= 2.12 check of `tools/build.py`, the `--toolchain` option), its deliverable records (`linux-manylinux2010-x86_64-legacy.json`, `linux-manylinux2010-i686-legacy.json`) and **spike S4** (its `spikes.json` entry, task T069 and its gate in `tasks.md`; the sources `pep-571` and `zig-cross`). History note: S4 removed on 2026-11-14, Linux binaries are built on the target machine, so the glibc baseline of a build host does not matter. The stored evidence (`validation/evidence/xlib-unicode-manylinux/` and the others) and the old verdict texts are kept as they were and are superseded by this ADR.
- **The Linux floor** is no longer "kernel 2.6 / glibc 2.12": the target machine needs a C++11 compiler, CMake and the libX11 development files (libXft and fontconfig ones for the Xft text path); a very old distribution may need a newer compiler installed. The native i686 build (`cmake/toolchains/linux-gcc-i686.cmake`, `gcc -m32`) stays: it is a native build on a machine that has the 32-bit libraries.
- **Xlib UI**: standard `sscanf` and `strtol` again; `std::chrono::steady_clock` in the probe instead of `gettimeofday`; libXft and fontconfig are linked (pkg-config `xft` and `fontconfig`, then a link test with the flags of the build), `xftdl` and `libdl` are gone and the Xft types come from the real headers (`xftfonts.cpp`). When CMake does **not** find Xft, the UI is built with the core font path (a `?` for a character the font lacks, the probe marker `glyph-substitution`), selected by the compile definition `LOOMORA_XLIB_HAVE_XFT` being absent and announced by a CMake status message; that is the only case of the fallback `xlib-core-font-glyphs`, which `tools/build.py` writes into the deliverable record of such a build. `LOOMORA_XLIB_XFT=ON` makes a missing Xft an error, `OFF` builds the core-font variant on purpose; the run-time switches `LOOMORA_XLIB_NO_XFT` and `LOOMORA_XLIB_NO_XIM` stay, so that the core-font path can be tested on a machine that has Xft (the core font is also used if Xft is linked but opens no font). Everything else (1,000-character limit, probe contract, XIM keys, the tofu box) is unchanged.
- **libstdc++ and libgcc are linked dynamically everywhere**: the options `LOOMORA_STATIC_CXX` (Xlib) and `LOOMORA_STATIC_CXX_RUNTIME` (GTK) and the fallbacks `static-cxx-runtime-xlib` and `static-cxx-runtime-linux` are deleted. The post-build check of `tools/build.py` accepts the expected libraries only (Xlib: libX11, with Xft libXft, fontconfig, FreeType, Xrender; GTK: the gtkmm / GTK family; both: libstdc++, libgcc_s, libm, libc, the loader), records them, and has no GLIBC / GLIBCXX ceiling.
- **GTK UI**: the `std::istringstream` readers of the kernel version and of `VmHWM` stay: they are the natural C++ code; only the comments about GLIBC_2.38 were removed.

## Rejected alternatives

- Keep the manylinux2010 build for a "wide compatibility" executable: it contradicts the rule that the binary is built where it runs, and it forced the static runtime and the dlopen of Xft on every build.
- Keep the dlopen of Xft so that a missing Xft costs nothing at build time: on the target machine the development files can be installed, and a build that sees no Xft says so and takes the fallback.

## Consequences

- Amended by this ADR: ADR 0001 rule 4 (static C++ runtime on Windows and Android only), ADR 0004 finding 13 and its `static-cxx-runtime-linux` entry, ADR 0005 and ADR 0006 (no static option; the `istringstream` readers stay for their own sake).
- A Linux deliverable says it was built natively on its machine and was not run elsewhere. Nothing verifies an old distribution until one is reachable (T092).
- On a machine without the 32-bit Xft development files the i686 build takes the core font fallback: `sudo apt install libxft-dev:i386 libfontconfig-dev:i386` gives the Xft path.
- Verified on Ubuntu 24.04 (WSL2 and WSLg): see the verdict notes and `validation/evidence/linux-build-on-target/linux-ubuntu-24.04-x86_64/`.
