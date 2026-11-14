# ADR 0001: C++11 backend behind a clean C API

- **Status**: accepted
- **Date**: 2026-11-12
- **Spike / requirement**: S6 / FR-017, FR-019

## Context

Loomora must run from Windows 2000, macOS 10.6, old Linux distributions, Android 2.3 and iOS 5 up to today. Each platform builds its UI with a different compiler and language (C, C++, Objective-C, Swift, C#, Java, JavaScript).

## Decision

A single backend in strict C++11 (`-std=c++11 -pedantic-errors`), written in idiomatic C++: the C++11 standard library, dynamic allocation, exceptions and RTTI are all allowed (the user lifted the earlier "runtime-free" restriction on 2026-11-12). The only boundary is a handwritten C API (`core/include/loomora.h`, valid as C89 and C++98), and the binding obeys four rules:

1. no C++ type, template or exception in the header;
2. every exported function catches all exceptions and returns a negative error code;
3. memory crosses the boundary only in caller-supplied buffers (no "free this" function);
4. the C++ runtime is linked statically where the platform has none or an old one (Windows, Android); on Linux it is linked dynamically, the system's own (amended by ADR 0007).

## Rejected alternatives

- **C99 backend**: the user decided on C++11 as the backend language.
- **Header-only backend**: every UI compiler would have to parse the C++ source; old compilers make that fragile.
- **Rust**: rejected by the user; its toolchain does not reach the oldest floors.
- **Runtime-free C++ (no STL, no exceptions)**: first adopted, then dropped by the user as needlessly restrictive; only the C binding needs care.

## Consequences

- `tools/check_std.py` checks that the shared build exports only `loomora_*` C symbols.
- Static linking of `libstdc++` (MinGW, NDK `c++_static`) must be confirmed by spikes S1 and S3 (the manylinux2010 spike S4 was removed by ADR 0007).
- Recent compiler features stay behind feature macros with a declared fallback (FR-021).
