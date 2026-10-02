---
name: termux-react-native-apk-build
description: Build, diagnose, and install a native React Native Android APK entirely on an ARM64 Termux device. Use when a user asks to create or release a non-Expo React Native Android app from Termux; do not use for Expo or conventional desktop Android builds.
---

# Termux React Native APK Build

Build a standalone Android APK from a standard React Native CLI project on the user's Termux device.

## Outcome

Produce an `arm64-v8a` APK that installs and opens without Metro. Keep the work native React Native; do not substitute Expo unless the user asks for it.

## Workflow

1. Inspect the existing project and preserve unrelated user changes.
2. Verify Termux has Node.js, JDK, CMake/Ninja, Android packaging tools, and an Android SDK/NDK path appropriate to the project.
3. Configure Gradle to use Termux-native build executables and build only the device ABI.
4. Ensure Hermes can compile on ARM64 Termux. React Native's Linux `hermesc` is usually x86_64, so run it through `qemu-x86_64` rather than disabling Hermes by default.
5. Build `assembleRelease`, verify that the release APK contains `assets/index.android.bundle`, and verify its signature.
6. If requested and authorized, install through Shizuku/rish, cold-launch the app, and inspect crash logs.

## Essential constraints

- Google Android SDK/NDK host binaries are usually x86_64. Do not attempt to execute them directly on ARM64 Termux.
- Point Gradle's AAPT2 override to Termux's native `aapt2` binary.
- If CMake cannot find ReactAndroid despite an extracted Prefab config, inspect `CMAKE_FIND_ROOT_PATH` and the compiler's library architecture. A Termux-only package-dir adjustment may be needed; do not apply one blindly to desktop builds.
- Do not claim a release build is standalone until `index.android.bundle` is present in the APK.
- Keep `android/local.properties`, keystores, tokens, `node_modules`, and build output out of Git.
- Treat Shizuku/rish as optional device-install plumbing. If it is disconnected, build and preserve the APK, report the specific connection issue, and do not falsely report installation.

Read [the Termux environment reference](references/termux-environment.md) before changing Android SDK/NDK, Gradle, or Hermes settings. Use the repository's top-level README as the human-facing full guide.
