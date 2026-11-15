# ADR 0018: Continuous integration runs on GitHub-hosted runners only; what they cannot run stays a local run

- **Status**: accepted (owner decision of 2026-11-15 on the runners; the points marked "not verified" are confirmed or refuted by the first run on GitHub, see Consequences)
- **Date**: 2026-11-15
- **Spike / requirement**: S10 (T121, open: see below) / FR-030, T122, T129

## Context

FR-030 asks for a continuous integration on GitHub with no manual step. The first wording of FR-030 and of tasks T121 to T131 planned two kinds of runners: GitHub-hosted runners for what they can run, and a self-hosted runner driving VirtualBox virtual machines and Docker containers for the rest. Both repositories (`github.com/bansan85/loomora` and `github.com/bansan85/loomora-docs`) are public, and GitHub itself says "self-hosted runners should almost never be used for public repositories", because "any user can open pull requests against the repository and compromise the environment" (<https://docs.github.com/en/actions/reference/security/secure-use>, read 2026-11-15). A self-hosted runner on the owner's machine would run the code of pull requests from forks there.

This record gathers what the hosted runners provide, so that the jobs of Phase 9 are written for real images and not for assumed ones. Every statement below was read on the page cited, on **2026-11-15**, or by a read-only call of the GitHub API on the same day; the statements that no page confirms are marked **not verified**. Nothing was run on GitHub: the repositories had no branch pushed yet (`gh api repos/bansan85/loomora/branches` returned an empty list), so no workflow could run.

## Decision

1. **Hosted runners only.** No self-hosted runner, no secret. Both repositories stay public (the hosted runners are free and unlimited for them).
2. **Whatever a hosted runner cannot do is a local run**, recorded in the validation records (verdicts, procedures, evidence under the rules of ADR 0014) and never a CI job: the VirtualBox virtual machines (Windows 2000 to 10, the old browsers such as Firefox 10), the manylinux1 CentOS 5.11 image (it needs `vsyscall=emulate` on the host kernel, a kernel parameter of the machine that runs the container, which is not ours to set on a hosted runner; whether the runner kernel already has it is not verified), and the legacy Apple systems (macOS 10.6, iOS 5). The workflow comments and `docs/building/README.md` say so. This replaces the self-hosted clause of FR-030 and the self-hosted parts of T121, T123, T124, T125, T126 and T131 (T131 becomes the documentation of those local runs).
3. **Runner images used** (labels as in the table below). The Linux jobs run `python3 tools/verify.py quick` on `ubuntu-22.04`, `ubuntu-24.04` and `ubuntu-24.04-arm`; `ubuntu-22.04` leaves the matrix when GitHub retires it (see below).
4. **Checkout.** `actions/checkout` with `submodules: recursive` and `persist-credentials: false` in every job that runs `verify.py`: `third_party/catch2` must be there for ctest to run the Catch2 tests (without it CMake silently leaves them out), and `docs/implementation` must be there for `tools/validate_records.py` (without it `verify.py` prints SKIP and `validate_records.py` exits 2). A job that only builds and never validates may use `submodules: false`: building never needs the docs. The change-detection job uses `submodules: false` and `fetch-depth: 0` (it only reads the list of changed files).
5. **Caching.** None in the first Linux job: the build of the backend and two UIs is small, and a cache that is not measured is a risk (a stale cache hides a configure error). Candidates, to be decided from the measured minutes of the first runs: ccache through `actions/cache` for the Linux builds; the Android NDK r15c, the Android system images and Emscripten for the later jobs (the NDK r27, r29 and r30 are already on the Ubuntu image). Limits of a cache: 10 GB per repository, entries not accessed for 7 days are removed, a run restores the caches of its branch and of the default branch, and a cache made by a pull request is only visible to that pull request (<https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching>, read 2026-11-15).
6. **Workflow hygiene** (T129): third-party actions pinned by full commit SHA with the tag in a comment (GitHub: "Pin references to actions to a commit SHA in your workflow files"); `permissions: contents: read`; `persist-credentials: false`; `concurrency` that cancels outdated runs; `timeout-minutes` on every job; a change that only touches Markdown files or `docs/` skips the build (a `changes` job reads the list of changed files) and runs only the validation of the records. The check to require in a branch protection rule is the `gate` job, not the build jobs: a workflow skipped by a `paths` filter would leave a required check pending, whereas `gate` always runs and passes when no job failed. The repository setting that makes GitHub refuse any action not pinned by SHA (`sha_pinning_required`) is `false` today (read with `gh api repos/bansan85/loomora/actions/permissions`); turning it on is the owner's decision.

## What the hosted runners provide

Source: <https://docs.github.com/en/actions/reference/runners/github-hosted-runners> (read 2026-11-15): "Use of the standard GitHub-hosted runners is free and unlimited on public repositories."

| Label | Architecture | Public repository: vCPU / RAM / disk |
|---|---|---|
| `ubuntu-22.04`, `ubuntu-24.04`, `ubuntu-26.04` (`ubuntu-latest`) | x64 | 4 / 16 GB / 14 GB |
| `ubuntu-22.04-arm`, `ubuntu-24.04-arm`, `ubuntu-26.04-arm` | arm64 | 4 / 16 GB / 14 GB |
| `windows-2022`, `windows-2025` (`windows-latest`), `windows-2025-vs2026` | x64 | 4 / 16 GB / 14 GB |
| `windows-11-arm`, `windows-11-vs2026-arm` | arm64 | 4 / 16 GB / 14 GB |
| `macos-26`, `macos-15`, `macos-latest` | arm64 (M1) | 3 / 7 GB / 14 GB |
| `macos-15-intel`, `macos-26-intel` | Intel | 4 / 14 GB / 14 GB |

Private repositories get half the CPU and memory of the Linux and Windows rows and draw on the included minutes; this does not apply to the public repositories. The page does not state the `-latest` mappings; its links point `ubuntu-latest` to Ubuntu 24.04, `windows-latest` to Windows Server 2025 and `macos-latest` to macOS 26 (the workflows name the exact label and never use `-latest`). The arm64 runners (Linux, Windows, macOS) are in the table of the public repositories with the same "free and unlimited" terms. On the arm64 macOS runners the page lists: no nested virtualization, and community actions that may not support arm64.

**Image contents** (the `runner-images` repository, images read 2026-11-15; `main` branch of <https://github.com/actions/runner-images/blob/main/images/>):

- `ubuntu-24.04` (image 20261111.370.1, `ubuntu/Ubuntu2404-Readme.md`): Docker client and server 28.0.4, CMake 3.31.6, Ninja 1.13.2, GCC 12.4, 13.3 and 14.2, Clang 16, 17 and 18, system Python 3.12.3, git 2.56.1, Node.js 22, Xvfb (apt package 2:21.1.12), Android NDK 27.3.13750724, 29.0.14206865 and **30.0.16248370** (so NDK r30 is on the image; NDK r15c is not). **Not listed** on that page: gtkmm or GTK development packages, `libx11-dev`, KVM. The Linux jobs therefore install the packages of `docs/building/README.md` with `apt-get`.
- `windows-2025` (image 20261111.331.1): Visual Studio Enterprise 2022 17.14, Windows SDK 10.0.26100, .NET SDKs 8.0 to 11.0, CMake 3.31.6, Ninja, MSYS2 in `C:\msys64` (not on PATH; no MinGW version listed), LLVM 20.1.8, Python 3.12.10, Chrome 157, Firefox 160, Docker 29.7.2, Android emulator 37.2.13 with NDK 27, 29 and 30, WSL (WSL 1 enabled, WSL 2 default). **Not listed**: Windows App SDK version.
- `windows-11-arm` (image 20261006.188.1): Visual Studio Enterprise 2022 17.14 with the ARM64, ARM64EC and ARM tools, .NET SDKs 6 to 10, CMake 4.4.4, Ninja, Python 3.13, Git, Node 24. **Not listed** in that readme: Visual Studio 2026 (the `windows-11-vs2026-arm` label exists in the table; its image was not read).
- `macos-26` arm64 (image 20261102.0473.1): Xcode 26.6 (default), 26.5, 26.4.1, 26.3, 26.2, 26.1.1 and 26.0.1; iOS simulator runtimes 26.2, 26.4 and 26.5; CMake 4.4.5, Ninja, Python, Node 24, Chrome 156, Firefox 159. **Not listed**: Emscripten.

**KVM.** The GitHub changelog of 2024-04-02 (<https://github.blog/changelog/2024-04-02-github-actions-hardware-accelerated-android-virtualization-now-available/>, read 2026-11-15) says hardware acceleration for the Android emulator is available on Linux runners with 2 vCPUs ("previously ... only on runners with 4 or more vCPUs") and that the runner user must be given access to `/dev/kvm` with a udev rule (the page shows the rule; emulator actions then use acceleration). It does not say whether the public-repository runners or the arm64 runners are covered: the public x64 Linux runners have 4 vCPUs, so the stated condition is met; **KVM on `ubuntu-24.04-arm` is not verified**, and so is whether an API 16 x86 or an API 24 emulator image boots (T124).

**Docker.** Docker 28.0.4 is on the Ubuntu 24.04 image, so a container of a public image (for example a manylinux image) can run on the Linux runners. Whether a container of the old distributions of the validation (manylinux2010) runs on the kernel of the hosted runner is **not verified**. The Docker on the Windows image is not assumed to run Linux containers (**not verified**).

**Deprecation of `ubuntu-22.04`.** GitHub's issue <https://github.com/actions/runner-images/issues/14254> (read through the API on 2026-11-15) announces that `ubuntu-22.04` and `ubuntu-22.04-arm` begin deprecation on 2026-09-17, with failing brownouts, and are fully unsupported from 2027-04-17: "GitHub Actions maintains the latest two stable versions of any given OS version". The Linux matrix keeps `ubuntu-22.04` until that date because it is the oldest hosted Linux image that builds with the system packages (gtkmm 3.24); after it, the oldest Linux system is a local run (Docker image or machine).

## What is run where

| What | Where | Status |
|---|---|---|
| `verify.py quick` (backend, Xlib and gtkmm UIs, vectors on Xvfb, ctest, check_std, records) | `ubuntu-22.04`, `ubuntu-24.04`, `ubuntu-24.04-arm` (T122, written) | not run on GitHub yet |
| Linux variants (GTK 3.10 floor, GTK 4, i686 Xlib, Xlib core font) | `ubuntu-24.04`, schedule and when their files change (T122, written; the i386 packages and the apt mirror for them are not verified) | not run on GitHub yet |
| Validation of the records only | `ubuntu-24.04` (change with no code in it) | not run on GitHub yet |
| MinGW-w64 cross build of the Windows Legacy UI, runs of the x64 and x86 executables | cross build on Linux (`g++-mingw-w64`, `-win32` thread model through `update-alternatives`, apt, not verified on the image), runs on `windows-2025` (WoW64 for x86; the x64 and x86 emulation on `windows-11-arm` is not verified) (T123, not written) | to do |
| Windows 2000 to 10 ladder | local VirtualBox run | not CI |
| Android: NDK r15c (download, verified checksum) and r30 (on the image), emulator API 16 x86 and API 24 | Linux x64 runner with KVM (T124, not written) | KVM boot not verified |
| Web: Emscripten (not on the images: install and cache), Node 22, headless Chrome and Firefox | Linux or Windows runner (T125, not written); Chrome and Firefox are listed on the Windows 2025 and macOS images, not read for the Ubuntu one | not verified |
| Old browsers (Firefox 10 on Windows 2000) | local VirtualBox run | not CI |
| macOS and iOS simulator, Modern UIs | `macos-26` (Xcode 26.x, iOS 26.2, 26.4 and 26.5 simulators) (T126, not written) | not verified |
| macOS 10.6 and iOS 5 | local run on Apple hardware or supplied image | not CI |
| WinUI 3 x64 and ARM64 | `windows-2025-vs2026` and `windows-11-vs2026-arm` (labels exist; contents not read) (T127, not written) | not verified |
| manylinux2010 and manylinux1 containers | manylinux2010 in Docker on a Linux runner is to be tried; manylinux1 (CentOS 5.11, `vsyscall=emulate`) stays a local run | not verified |

## Checkout of the docs repository

`actions/checkout` (latest release v7.0.1, 2026-07-20; <https://github.com/actions/checkout>, README read 2026-11-15) takes `submodules: false | true | recursive`. The submodule URL of `docs/implementation` is the relative `../loomora-docs`; git resolves a relative URL against the URL of the remote `origin`, which on a runner is `https://github.com/bansan85/loomora`, so it becomes `https://github.com/bansan85/loomora-docs`, the sibling repository, public so with no token. That is git's documented behaviour, but neither the checkout README nor any GitHub page says it for this action: **not verified until the first run**. Consequences kept in mind:

- The commit that the submodule pointer names must exist in `loomora-docs` on GitHub and be reachable from one of its branches: push `loomora-docs` first, then `loomora`.
- A pull request from a fork resolves the same way against the base repository (`../loomora-docs` of the base), not against the fork: not verified.
- If `loomora-docs` ever becomes private, the checkout needs a token that can read it (a fine-grained token or a deploy key stored as a secret), which this decision forbids for now ("no secret").

## Alternatives rejected

- **A self-hosted runner** (the first wording of FR-030): rejected for a public repository, see Context.
- **A self-hosted runner restricted to branches of the repository** (no fork): still runs on the owner's machine code that any collaborator or compromised account can push, with the build machine's data and network; the benefit (the old systems in CI) is kept by the local runs.
- **`ubuntu-latest` and the other `-latest` labels**: they move without notice; a CI that changes its baseline by itself hides a regression of the floors, so the exact label is named.
- **Third-party actions for caching, paths filters or toolchains** (`dorny/paths-filter`, `jwlawson/actions-setup-cmake`, ...): not needed for the first jobs; each one is an extra pinned dependency to audit. The changed files are read with `git diff`.
- **`paths:` filters on the workflow trigger** to skip the build for a documentation change: a skipped workflow leaves a required check pending. A `changes` job and a `gate` job give the same saving without it.

## Consequences

- S10 stays **open** in `validation/spikes.json`. Answered by reading the documentation and the image readmes: the runner labels and their sizes, the arm64 runners being free for public repositories, the contents of the Ubuntu 24.04, Windows 2025, Windows 11 ARM and macOS 26 images, the NDK r30 on the Ubuntu image, Docker on the Linux image, the KVM rule for Linux runners, the cache limits, the retirement of `ubuntu-22.04`. Not answered, to be confirmed by the first run after the owner pushes both repositories: the relative submodule checkout; the real duration of `verify.py quick` on each image and whether the Xlib and gtkmm UIs run under Xvfb on `ubuntu-24.04-arm`; the i386 packages on the hosted Ubuntu 24.04; KVM on the arm64 runner and the boots of the API 16 and API 24 emulators; manylinux2010 in Docker; the Windows 2025 and macOS jobs. Each is answered by a run, never by the documentation alone (T121).
- FR-030 (spec) and the text of T121, T123 to T126 and T131 still describe a self-hosted runner: the owner decides how to reword them. This record is the later decision.
- The first workflow is `.github/workflows/ci.yml` (T122, T129). Its actions are pinned to `actions/checkout` v7.0.1 (`3d3c42e5aac5ba805825da76410c181273ba90b1`) and `actions/upload-artifact` v7.0.2 (`cf430e030ddbb5b0abf93d22962f4752f3646cd9`), both read from `gh api repos/<owner>/<repo>/git/ref/tags/<tag>` on 2026-11-15 (annotated tags would have needed one more dereference; both are plain tags pointing at the commit).
- `tools/validate_records.py` accepts the spike id S10.

## Findings: Android and Web jobs (2026-11-15)

Only what the runs showed. Workflows `.github/workflows/web.yml` and `.github/workflows/android.yml`, branch `feat/ci3`.

- **Web** (`ubuntu-24.04` plus `windows-2025`): run <https://github.com/bansan85/loomora/actions/runs/43714823780> green. Build and the three node launchers 1 min 37 s (emsdk install 47 s on the cache miss, then cached); 15 of 15 vectors per launcher; the headless Chrome 156.0.8078.80 preinstalled on `windows-2025` ran 15 of 15 in 1 min 10 s. Later runs of the same workflow on commits 2bcbaa2 and 9491e7a are green too (<https://github.com/bansan85/loomora/actions/runs/43730165573>). Firefox of a recent version is not run.
- **Android** (`ubuntu-24.04`): run <https://github.com/bansan85/loomora/actions/runs/43730165614> green. The NDK r15c is not on the image: it is downloaded from Google and checked against the published size and SHA-1 and a SHA-256 computed on that download. Legacy build 1 min 14 s, Modern build (NDK r30, AGP) 3 min 1 s.
- **KVM**: `/dev/kvm` exists on the Linux runner but is root-only until a udev rule opens it (`MODE="0666"`, then `udevadm trigger`); with it both emulators ran hardware-accelerated.
- **API 16 x86 boots on the hosted runner**: Android 4.1.2 booted in 31 s and the Legacy UI (built for API 9) passed 15 of 15 vectors in 45.0 s. API 24 x86_64 (Android 7.0) booted in 10 s and the Modern UI passed 15 of 15 in 48.6 s. Each emulator job took about 2 min in total. No image of an API below 16 exists for the emulator, so the lower levels stay built only (`not_tested`).
- **Pitfalls met, in the order they appeared**: (1) `avdmanager` and the emulator look for the SDK from their own location or from `ANDROID_HOME` and `ANDROID_SDK_ROOT`; with the image's SDK in the way the emulator ended with `Broken AVD system path` (it searched `/usr/local/lib/android/sdk/system-images/android-16/default/x86/`); the job uses the cmdline-tools of its own SDK folder and exports both variables to it. (2) The emulator process must stay alive in the same step as `adb` and the vectors (boot and run are one step). (3) `ro.product.cpu.abilist` does not exist before API 21: `tools/run_vectors.py --ui-abi` now reads `ro.product.cpu.abi` and `abi2` when it is empty. (4) `python3-jsonschema` is installed with `apt-get` for `tools/check_probe.py`.
- **Gate**: the emulator jobs are separate and not part of the `gate` job, because booting an emulator is the least reliable step of the CI; they are never `continue-on-error`, so a failing vector still fails the run. The first two runs of the emulator jobs failed (points 1 and 3); the build jobs passed from the first run on. Whether the boot stays reliable over many runs is not known (one green run per emulator).
