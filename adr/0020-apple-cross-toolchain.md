# ADR 0020: Apple cross toolchain per target

- **Status**: accepted
- **Date**: 2026-11-16
- **Spike / requirement**: S2, S7, S11 / FR-017, FR-021

## Context

The Apple family has four deliverables: Modern macOS (Swift, SwiftUI), Modern iOS (Swift, SwiftUI), Legacy macOS (Objective-C, AppKit, no ARC) and Legacy iOS (Objective-C, UIKit, no ARC). Their floors are macOS 10.6, iOS 5 (Legacy) and macOS 11, iOS 15 (Modern). Each floor needs a toolchain that can produce a binary for it, and each run needs a system of that age, which the owner does not have.

Three toolchain routes exist:

- **Xcode 26 on the `macos-26` hosted runner** (Apple silicon, macOS 26.x, SDK 26.x). Native for Modern; for the Legacy floors it can only compile with `-mmacosx-version-min=10.6` (x86_64) or `IPHONEOS_DEPLOYMENT_TARGET=12.0` (arm64). Spike S11 answered this by a real run (2026-11-16): the compile and the minimum-version stamp work, the availability check works, libc++ is linked as a dylib.
- **osxcross with the 10.6 SDK** from a Linux host (`cmake/toolchains/osxcross-macos10.6.cmake`, T028). Needs the MacOSX10.6.sdk, which Apple distributes only inside old Xcode packages (spike S7).
- **Xcode 5.1.1** for iOS 5 (`cmake/toolchains/ios-xcode5.cmake`, T028). Needs a Mac that can run it (OS X 10.8.4 or later, Intel for the Mac build tools of that era) and a device or simulator of iOS 5 (spike S2). The iOS 5 simulator is reported unusable on OS X 10.9 and later (spike S7).

The owner has no old Mac, no old Xcode installation and no iOS 5 device (decision of 2026-11-16, tasks T067, T089, T090). Hosted runners cannot run Xcode 5.1.1 or a 10.6 system.

## Decision

Each Apple target is built with the newest toolchain that can produce its floor, on the `macos-26` runner:

| Target | Toolchain | Floor compiled | Floor run |
|---|---|---|---|
| Modern macOS | Xcode 26 (native) | macOS 11, Universal 2 | macOS 26 (CI) |
| Modern iOS | Xcode 26, `iphonesimulator` and `iphoneos` | iOS 15 | iOS 26 simulator (CI) |
| Legacy macOS | Xcode 26 Clang, `-mmacosx-version-min=10.6`, `x86_64`, `-Werror=partial-availability` | 10.6 | not run (compiled only) |
| Legacy iOS | Xcode 26, `arm64`, `IPHONEOS_DEPLOYMENT_TARGET=12.0` | iOS 12 (override of 5.0) | not run (compiled only) |

The two old-toolchain files stay in `cmake/toolchains/` as the route to run the floors on real hardware, used only when the owner supplies the SDK and the machine:

- `osxcross-macos10.6.cmake` builds i386 and x86_64 for 10.6 from Linux, when `LOOMORA_MACOS_SDK_PATH` names a 10.6 SDK;
- `ios-xcode5.cmake` builds armv7, armv7s and arm64 for iOS 5 on a Mac with Xcode 5.1.1.

Neither is used by the CI.

## Rejected alternatives

- **osxcross in CI.** The 10.6 SDK is not in the repository and cannot be redistributed; the runner has none.
- **Xcode 5.1.1 in CI.** Not installable on the current hosted image, and the iOS 5 simulator does not run on the runner's OS.
- **A macOS 10.6 or Xcode 5 virtual machine as the CI path.** ADR 0019 keeps VMs local and optional; the owner cannot supply the media.
- **Dropping the Legacy Apple rows.** Rejected: the compile row is a real check (SDK availability, minimum-version stamp, floor API use); it is labelled "compiled, not run", never "run".

## Consequences

- Legacy macOS and Legacy iOS are **compiled, never run**. Their verdicts stay `not_tested` with the reason (`validation/verdicts/macos-26-arm64-legacy.json`, `ios-26-arm64-simulator-legacy.json`).
- The fallbacks `macos-legacy-x86_64-only`, `macos-legacy-libcxx-dylib` and `ios-legacy-deployment-override` (`validation/fallbacks.json`) describe what the compile-only rows rely on. The libc++ dylib of a real 10.6 system is not verified (spike S11).
- Spike S2 (old Mac) and spike S7 (legal source of the SDK and Xcode 5.1.1) are recorded with the owner's decision; nothing here claims a run on 10.6 or iOS 5.
- A future run on real hardware changes only the verdict files and the toolchain files named above; the CI route does not change.
