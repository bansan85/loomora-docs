# Procedure: iOS (Legacy and Modern UIs)

Replayable procedure for the iOS Modern UI (SwiftUI, deployment target iOS 15) and the iOS Legacy UI (Objective-C, UIKit, no ARC, deployment target iOS 5 in the project, iOS 12 for the arm64 compile of CI). Written on 2026-11-16 from the build targets of `tools/build.py`, the Apple workflow `.github/workflows/apple.yml` (T126) and the records of this directory.

## Status of this document

- **No iOS run has happened yet.** The Apple workflow was written on 2026-11-16 without a Mac and had not run when this page was written. The only iOS system in scope that can be run is the iOS 26 simulator of the `macos-26` hosted runner (fifth session rule, spec Clarifications). Every iOS row below marked `not_tested` stays so until a run proves otherwise.
- **iOS 5 is not run and will not be run in this project** unless the owner supplies a real device or a Mac with Xcode 5.1.1 (spike S2, T067, closed by the owner's decision of 2026-11-16: no old hardware). The iOS 5 simulator is reported as unusable on OS X 10.9 and later (spike S7, T065). The Legacy UI is therefore compiled, not run, on every iOS system that matters for the floor.
- A record is only as strong as its last run. The verdicts below are the source of truth; this page describes how to reproduce them.

## Verdicts

| Platform record | Variant | Verdict | What is done |
|---|---|---|---|
| `ios-26-arm64-simulator` | modern | `validation/verdicts/ios-26-arm64-simulator-modern.json` | `not_tested`: build for the iOS 26 simulator and run the 15 vectors through `xcrun simctl` in the `ios-modern-simulator` job (T126), not run yet |
| `ios-26-arm64-simulator` | legacy | `validation/verdicts/ios-26-arm64-simulator-legacy.json` | `not_tested`: compile only, arm64 at iOS 12 (`ios-legacy` job), never run |
| `ios-26-arm64-simulator` | modern, device floor | (no separate verdict) | `not_tested`: the iPhoneOS SDK build at deployment target iOS 15 (`ios-modern-device` job), unsigned, never installed |

The iOS 15 floor and the iOS 5 floor have no platform record of their own; they are compile-only rows of the records above, and the iOS 5 row waits for hardware (spike S2).

## Rules of this procedure

- Builds of the Apple targets run on the `macos-26` runner only (`runs-on: macos-26`, Apple silicon, Xcode 26.x). Nothing Apple is built on Windows or WSL.
- The iOS simulator runtime is chosen by `tools/ci/ios_simulator.py --major 26`. Its probe line gives the minor version (`ios-26.<minor>-arm64-simulator`); the record file name, its id and the verdicts must then be renamed to that id (the record says so).
- A screenshot is not evidence; ask the owner before adding an image to `docs/implementation`.
- Evidence only when it proves something new (ADR 0014): the probe line and the vector log of a UI that has never run on that system.

## Part A: iOS 26 simulator, Modern UI (CI job `ios-modern-simulator`)

1. Build for the simulator:

   ```
   python tools/build.py ios modern --ios-sdk iphonesimulator 2>&1 | tee build-ios-modern-simulator.log
   ```

   Expected: an arm64 simulator app, the `ios-modern` target of `tools/build.py`, deployment target iOS 15.

2. Boot a simulator of the newest iOS 26 runtime and wait for the end of the boot:

   ```
   python3 tools/ci/ios_simulator.py --major 26
   ```

   The last lines give `udid=<UDID>` and `platform_id=ios-<major.minor>-arm64-simulator`.

3. Run the vectors (install and launch with `xcrun simctl`, read the probe line from the console and from `log show`):

   ```
   python tools/run_vectors.py "$PLATFORM_ID" modern --udid "$UDID" --jobs 1 --evidence-dir <dir>
   ```

   Expected: each of the 15 reference vectors round-trips unchanged and the probe line carries the platform id of step 2. The end-of-life banner must not show: iOS 26 is not end of life (`validation/sources.json`, entry eol-apple-security).

4. Record the verdict: the `ios-26-arm64-simulator-modern.json` status changes only when the job is green on a real run; cite the run URL and the log in the notes.

## Part B: iOS 15 floor, device SDK (CI job `ios-modern-device`)

```
python tools/build.py ios modern --ios-sdk iphoneos 2>&1 | tee build-ios-modern-device.log
```

Compile and link check for arm64 at deployment target iOS 15, unsigned. It proves the build, not the run: the app cannot be installed on a device without a signing identity, and none is in the repository. The row stays `not_tested` for the run.

## Part C: Legacy UI, compile only (CI job `ios-legacy`)

```
python tools/build.py ios legacy 2>&1 | tee build-ios-legacy.log
```

Expected: an arm64 build with `IPHONEOS_DEPLOYMENT_TARGET=12.0` (override of the project's iOS 5.0, fallback `ios-legacy-deployment-override`), the Info.plist naming the executable and the floor. The result is recorded as "compiled for arm64 at iOS 12, iOS 5 `not_tested`". The armv7 and armv7s slices are not built on the runner (Xcode 26 does not build the 32-bit slices).

## Part D: Floors that cannot be run here (owner decision of 2026-11-16)

- **iOS 5 with Xcode 5.1.1** (the toolchain of `cmake/toolchains/ios-xcode5.cmake`, T028): needs a Mac with Xcode 5.1.1 and a device or a simulator. Not available; `not_tested` with the reason of spike S2.
- **armv7 and armv7s slices**: need a 32-bit device of the matching generation. Not available; `not_tested`.

These are not skipped silently: each one has its verdict with its reason in the records.

## Replay

A reader who has a macOS 26 machine with Xcode 26.x and an iOS 26 simulator runtime can run parts A to C in that order. Parts A and B also run from the `apple.yml` workflow (`workflow_dispatch`). Compare the logs with the verdict notes, not with memory.
