# ADR 0013: Two verification levels, `quick` by default and `full` only for graphical changes

- **Status**: accepted
- **Date**: 2026-11-14
- **Spike / requirement**: FR-016, FR-018, SC-011 (owner decision of 2026-11-14, tool `tools/verify.py`)

## Context

Every branch has to be verified, by Claude or by an agent. Without a standard each one would improvise its checks, and the graphical ones (a real window, a screenshot, a sanitizer run) are the slow and expensive part: they need Xvfb or WSLg, they produce images, and for a text-only change they prove nothing the probe line does not already prove.

## Decision

`tools/verify.py` has two levels, one line per check, logs under the build directory:

- **`quick`** (the default, the verification of any branch): build of what exists for the host, `ctest`, `tools/run_vectors.py` through each UI built here (the probe line and the reference vectors are the functional proof), `tools/check_std.py` and `tools/validate_records.py`. On Linux and WSL: the core and the Linux UIs, default variant only, run on a private Xvfb display when Xvfb is installed. On a Windows host: the clang and Ninja build of the core and the Win32 UI (as `CMakePresets.json`) and the vectors through that executable. No screenshot, no GUI driving, no sanitizer, no variant. An agent runs it instead of its own checks.
- **`full`** only for a graphical change: a new graphical component, a new graphics platform or toolkit, new font, Unicode, emoji or non-ASCII rendering support, or a change of how text is drawn. It adds, for the Linux UIs, one screenshot per UI and rendering scenario (mixed Unicode, a 1,000-character text) and the Xlib harness under ASan and UBSan, on a private Xvfb display (never WSLg's `:0`; the script gets a private mount namespace where `/tmp/.X11-unix` is read-only). On a Windows host it relays `full` to WSL. It is never run "to check that it still works".
- **Variants** (`--variants`: GTK 3.10 floor, GTK 4, i686 Xlib, Xlib core font) only on request or when the files of that variant change. One default variant per UI during the probe phase.
- Screenshots are candidates: `full` writes them under the build directory and never copies them into the repository (ADR 0014).
- Parallel agents each work in their own worktree with their own build directory (default `$HOME/loomora-build/verify-<hash of the repository path>`).
- Exit status: 0 all passed, 1 a check failed, 2 a tool that `full` needs is missing (the exact `apt` line is printed).

## Rejected alternatives

- **Free-form checks per agent**: results that cannot be compared, and a screenshot taken each time "to see".
- **Always `full`**: slow, needs a display, and the images prove nothing for a change that does not touch rendering.

## Consequences

- `quick` does not exercise the graphical paths: a text-drawing change must run `full` to be accepted.
- The Windows host runs the Win32 UI as a host build only; the MinGW-w64 deliverables and Windows 2000 stay manual (ADR 0012, scenario 2 of `quickstart.md`).
- Nothing in `verify.py` starts Docker, the VirtualBox VM or a trial; those are separate explicit commands.
