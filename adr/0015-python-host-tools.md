# ADR 0015: The host tools are Python 3.8+ scripts using the standard library only

- **Status**: accepted
- **Date**: 2026-11-14 (the choice of Python was made in research R9 on 2026-11-12; this record states it with the tools as they are)
- **Spike / requirement**: FR-016, FR-018, FR-027, research R9

## Context

The project needs programs that are not part of any product: build a deliverable and record it, verify a branch, feed the reference vectors to a UI, check the C++11 and C API rules, validate and report the data records, measure code sharing, and prove that one change of the greeting prefix reaches every UI. They run on the Windows host of the project and in WSL (Ubuntu 24.04), they start other programs (CMake, MinGW-w64, `wsl`, VirtualBox), and they read and write JSON. Nothing of them is shipped, and the backend does not depend on them (ADR 0001), except that CMake runs one of them while building the tests.

## Decision

- **The host tools are Python scripts, Python 3.8 or later, standard library only.** The floor is declared in the tools themselves and in `core/tests/CMakeLists.txt` (`find_package(Python3 3.8 REQUIRED COMPONENTS Interpreter)`). The only third-party package is `jsonschema` (`tools/requirements.txt`), and only for the probe schema: `tools/check_probe.py` imports it when it first validates a line, `tools/run_vectors.py` and `tools/prefix_check.py` use `check_probe.py`, and `tools/validate_records.py` skips the probe evidence check with a warning when the package is missing. Every other tool (`build.py`, `verify.py`, `check_std.py`, `check_imports.py`, `check_ui_logic.py`, `measure_sharing.py`, `report.py`, `diff_verdicts.py`, `gen_vectors.py`) imports the standard library and `tools/common.py` only.
- **What the tools are** (`tools/`):
  - `build.py` builds one deliverable (`<platform> <variant>`), copies it to `out/` and writes its deliverable record into the docs submodule when it is checked out (without the submodule it warns and writes no record). Only the Windows Legacy, Linux Legacy and Linux Modern targets are written; the others stop with "not implemented yet". It runs `check_imports.py` on every Windows executable (ADR 0002, ADR 0012).
  - `verify.py` is the verification of a branch (`quick`, `full`: ADR 0013).
  - `run_vectors.py` launches a built UI in probe mode on every valid reference vector and checks the probe line with `check_probe.py`; it also holds the launchers (local host, WSL, VirtualBox guest). `check_probe.py` validates one probe line against `probe-result.schema.json`.
  - `check_std.py` (strict C++11 with every GCC and Clang found, clean C API exports), `check_imports.py` (imports of a Windows executable against `win2000_imports.json`) and `check_ui_logic.py` (no greeting built outside the backend).
  - `validate_records.py` (platforms, verdicts, deliverables, fallbacks, spikes, sources of `validation/`), `report.py` (writes `validation/report.md`, deterministic) and `diff_verdicts.py` (differences between two sets of verdicts).
  - `measure_sharing.py` (backend, glue and UI lines per platform and variant) and `prefix_check.py` (changes the prefix in a copy of `core/src/greet.cpp` only, builds every UI the machine can build and runs the vectors).
  - `gen_vectors.py` turns `core/tests/vectors.json` into the initializer lists the Catch2 tests include; the CMake build runs it.
  - `common.py` (paths, UTF-8 console output, JSON helpers) and the drivers under `tools/trials/<platform-id>/`, which are tooling and not evidence (CLAUDE.md). Programs that must run on the system under test are not Python: `tools/guest/` holds C and shell (`runw.c`, `build_selftest.sh`).
- **Where they run.** The same scripts run on the Windows host (`python tools/<tool>.py`) and in WSL or Linux (`python3 tools/<tool>.py`). Where a step needs the other side, the script starts it itself: on a Windows host `build.py linux ...` and `run_vectors.py` for Linux UIs re-run themselves inside WSL (`wsl -d <distro> --cd <repository path> -- python3 tools/...`, the distribution from `--wsl-distro`, `$LOOMORA_WSL_DISTRO` or `Ubuntu-24.04`), and `verify.py full` relays itself to WSL. The MinGW-w64 build (`build.py windows legacy`) is meant to be started in WSL (ADR 0012). Builds in WSL go under `$HOME/loomora-build` (`$LOOMORA_BUILD_DIR` overrides it when several clones exist). The Windows 2000 guest runs C programs only; the Python drivers stay on the host and reach the VM with `VBoxManage` (the VM is never started or restored by a tool).
- **Conventions**: one line per check, exit status 0 for success, 1 for a failed check, 2 for a usage or setup error (a missing submodule or tool, with the command that fixes it); `--self-test` or `--selftest` on the tools that check things.

## Rejected alternatives

- **C++ tools** (research R9): slower to write and no benefit for report generation, JSON and driving other programs; they would also need building before the first check.
- **Shell or PowerShell scripts**: two dialects for the two hosts, and no portable JSON handling.
- **Python with several third-party packages** (a test framework, a CLI or HTML library): a virtual environment to install on every host, WSL and CI machine, for no gain. A package is accepted only where the standard library has no equivalent, and `jsonschema` is the one case (the probe schema).
- **A higher Python floor**: the floor of research R9 is kept; raising it would exclude older hosts and nothing requires it.

## Consequences

- A host needs Python 3.8+ and, for `check_probe.py` and so for `run_vectors.py`, `prefix_check.py` and the probe evidence check, `pip install -r tools/requirements.txt` (Windows) or `python3-jsonschema` (Ubuntu). Building the product never needs `jsonschema`; CMake needs only Python 3.8 to generate `vectors.inc`.
- The tools were run on Python 3.13 (Windows host) and Python 3.12 (WSL Ubuntu 24.04), with `jsonschema` 4.26 and 4.10. The 3.8 floor is a rule the code is written to, not something that was run on a 3.8 interpreter.
- A tool that depends on a platform tool says so and fails with a message that names what to install (`build.py` prints the missing MinGW-w64 tools; `verify.py full` prints the `apt` line). It never falls back to a weaker check.
- Python tools are not reviewed as product code, but they are the functional proof of a branch (`verify.py quick`, CLAUDE.md): a change to a tool is verified with its self-test and with `verify.py quick`.
- The JavaScript build tool needed for the Web UIs (spike S5, not run) is a separate decision and is not covered here.
