# ADR 0011: C++ standard per target: backend C++11, Legacy UIs C++11, Modern UIs up to C++17

- **Status**: accepted
- **Date**: 2026-11-14
- **Spike / requirement**: S6 / FR-019, FR-021, FR-022

## Context

Every UI links the same backend library, and the oldest floors only have compilers that stop at C++11 (MinGW-w64 GCC for Windows 2000, Android NDK r16b, Apple Clang 5.1). Some Modern toolkits need more: gtkmm 4.10, glibmm 2.68 and libsigc++ 3 do not compile under `-std=c++11`. A global `CMAKE_CXX_STANDARD` would give the UI standard to the backend, or the backend standard to a UI that cannot use it.

## Decision

- **Backend** (`core/`, the library `libloomora`, its tests, the selftest): always **C++11**, with `CXX_STANDARD 11`, `CXX_STANDARD_REQUIRED ON`, `CXX_EXTENSIONS OFF`, whatever the UI around it (`core/CMakeLists.txt`). `tools/check_std.py` also compiles it with GCC and Clang `-std=c++11 -pedantic-errors -Wall -Wextra -Werror`. A feature newer than C++11 stays behind a macro of `core/src/internal/config.hpp` with a fallback (ADR 0004; for example `LOOMORA_HAS_CXX17_NODISCARD`).
- **Legacy UIs**: written in **C++11**.
- **Modern UIs**: up to **C++17**, the lowest standard their toolkit accepts: `ui/modern/linux-gtk3` is C++11 with gtkmm-3.0 and C++17 with gtkmm-4.0.
- The standard is **a property of each target**, never a global variable, so a UI standard cannot reach the backend. `tools/build.py linux modern` reads `compile_commands.json` and fails unless the backend sources are compiled with `-std=c++11` and the UI sources with one standard among C++11, C++14 and C++17.
- MSVC cannot select C++11 (its lowest is `/std:c++14`): it is never the conformance reference, it only proves the same source compiles (research R2).

## Rejected alternatives

- **One standard for the whole repository (C++17)**: it would make the backend fail on the floors that only have C++11 compilers.
- **C++14 or C++17 for the gtkmm-3.0 UI**: nothing in it needs more than C++11, and it would exclude compilers that accept C++11 only (ADR 0006).
- **A C++11 Modern UI with gtkmm 4**: its headers do not compile as C++11.

## Consequences

- The `cpp_std` field of a deliverable record is the backend standard, always `c++11`; the UI standard of a Linux Modern build is in its `ui_toolkit` and `notes` fields.
- On a C++11 build `LOOMORA_NODISCARD` is empty: that fallback is taken by every platform recorded so far (`validation/fallbacks.json`, `cxx17-nodiscard`).
