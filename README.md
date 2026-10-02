# Building a native React Native APK entirely in Termux

This is a practical record of building a **non-Expo**, native React Native Android APK directly on an ARM64 Android device with Termux.

It is intended as a reference for Termux users. It does not contain an application or APK.

## What this setup builds

- A standard React Native CLI project
- A signed release APK
- An APK containing its JavaScript bundle, so it does not need Metro to open
- `arm64-v8a` output for the device running Termux

## 1. Install Termux packages

Use a current Termux installation, then install the native build tools:

```sh
pkg update
pkg install nodejs openjdk-21 git curl unzip zip \
  cmake ninja make aapt2 d8 apksigner ndk-multilib qemu-user-x86-64
```

Verify the essentials:

```sh
node --version
java -version
cmake --version
aapt2 version
qemu-x86_64 --version
```

`qemu-user-x86-64` is important: React Native currently distributes the Linux Hermes compiler as an x86_64 binary, while most Termux phones are ARM64.
Use a JDK supported by your project's Android Gradle Plugin; the verified React Native 0.87.1 build used Termux OpenJDK 21. Package names can differ by Termux repository.

## 2. Create a native React Native project

This uses the React Native Community CLI, not Expo:

```sh
npx --yes @react-native-community/cli@latest init MyApp --version latest --skip-install
cd MyApp
npm install
```

Change the Android application ID in `android/app/build.gradle` before publishing a real app.

## 3. Android SDK and NDK notes

Set up an Android SDK under `$HOME/android-sdk` with the platform and build-tools version required by the project. The example build used:

```text
compileSdkVersion = 37
buildToolsVersion = 37.0.0
ndkVersion = 27.1.12297006
```

Google's downloaded Android NDK host tools are normally x86_64, so they cannot run directly on ARM64 Termux. The workable approach is a Termux-compatible NDK layout:

1. Keep the Google NDK's `build/cmake`, `meta`, and `platforms` content.
2. Point `toolchains/llvm/prebuilt/bin` to Termux's LLVM tools.
3. Point its sysroot headers, C++ headers, libraries, and Clang runtime files to the matching Termux toolchain paths.
4. Set `ndk.dir` in `android/local.properties` to that compatibility layout.

The precise symlinks depend on the installed Termux Clang version and NDK version. Do not copy an x86_64 NDK `prebuilt/linux-x86_64/bin` directory into an ARM64 build unchanged.
The compatibility layout is a prerequisite, not something `npm install` creates. Verify that its Clang and CMake toolchain files exist before building.

Example `android/local.properties`:

```properties
sdk.dir=/data/data/com.termux/files/home/android-sdk
cmake.dir=/data/data/com.termux/files/usr
ndk.dir=/data/data/com.termux/files/home/termux-ndk
```

## 4. Tell Gradle to use Termux AAPT2 and ARM64

Add these values to `android/gradle.properties`:

```properties
hermesEnabled=true
android.aapt2FromMavenOverride=/data/data/com.termux/files/usr/bin/aapt2
reactNativeArchitectures=arm64-v8a
```

`android.aapt2FromMavenOverride` prevents Gradle from trying to execute the x86_64 AAPT2 downloaded from Google's Maven repository.

## 5. Run Hermes through QEMU

Create `scripts/hermesc-qemu.sh` and make it executable:

```sh
#!/data/data/com.termux/files/usr/bin/sh
script_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
exec /data/data/com.termux/files/usr/bin/qemu-x86_64 \
  "$script_dir/../node_modules/hermes-compiler/hermesc/linux64-bin/hermesc" "$@"
```

```sh
chmod 700 scripts/hermesc-qemu.sh
```

Then add this to the `react {}` block in `android/app/build.gradle`:

```groovy
hermesCommand = "$rootDir/../scripts/hermesc-qemu.sh"
```

Without this, a release build can fail with:

```text
OS not recognized. Please set project.react.hermesCommand
```

Avoid turning Hermes off as a shortcut unless the JavaScriptCore native libraries are fully configured: recent React Native builds can otherwise fail at runtime because `libhermestooling.so` is not packaged.

## 6. Fix Prefab package lookup on Termux when needed

In a verified React Native 0.87.1 build, CMake reached native configuration but could not find `ReactAndroidConfig.cmake`. The file existed under Gradle's extracted Prefab tree at `lib/aarch64-linux-android/cmake/ReactAndroid/`; Termux Clang reported `aarch64-none-linux-android24` as the library architecture, so automatic package lookup missed that directory. This is a CMake lookup issue, not a missing npm package.

If your build has this specific failure, create `scripts/termux-prefab.cmake`:

```cmake
if(CMAKE_FIND_ROOT_PATH)
  list(GET CMAKE_FIND_ROOT_PATH 0 _termux_prefab_root)
  set(_termux_prefab_cmake
    "${_termux_prefab_root}/lib/aarch64-linux-android/cmake")
  foreach(_termux_package ReactAndroid fbjni hermes-engine)
    set("${_termux_package}_DIR"
      "${_termux_prefab_cmake}/${_termux_package}")
  endforeach()
endif()
```

In the `defaultConfig { externalNativeBuild { cmake { ... } } }` block of `android/app/build.gradle`, add a Termux-only argument:

```groovy
if (providers.gradleProperty("termuxBuild").orNull == "true") {
    arguments "-DCMAKE_PROJECT_INCLUDE=${file('../../scripts/termux-prefab.cmake').absolutePath}"
}
```

Pass `-PtermuxBuild=true` to Gradle on the phone. The first entry of `CMAKE_FIND_ROOT_PATH` is the extracted Prefab root; it is a list after the NDK toolchain appends its own path. Keep the adjustment conditional so PC builds retain ordinary CMake package resolution. Check the actual Prefab directory and package names if your React Native version differs.

## 7. Build the standalone release APK

From the `android` directory:

```sh
./gradlew assembleRelease --no-daemon -PtermuxBuild=true
```

The APK is written to:

```text
android/app/build/outputs/apk/release/app-release.apk
```

Check that it is signed and includes the offline JS bundle:

```sh
apksigner verify --verbose app/build/outputs/apk/release/app-release.apk
unzip -l app/build/outputs/apk/release/app-release.apk | grep index.android.bundle
```

The React Native 0.87.1 / Android Gradle Plugin 9.2.1 project used to verify this guide built successfully with the Termux Prefab fix. Its APK signature verified, but build success alone does not prove installation or UI behavior on a device.

## 8. Install through Shizuku / rish

First grant Termux storage access:

```sh
termux-setup-storage
```

Copy the APK to shared storage, then use `rish`:

```sh
cp android/app/build/outputs/apk/release/app-release.apk \
  /storage/emulated/0/Download/MyApp-release.apk

rish -c 'cp /storage/emulated/0/Download/MyApp-release.apk /data/local/tmp/MyApp-release.apk && cmd package install -r /data/local/tmp/MyApp-release.apk'
```

Cold-launch and inspect it:

```sh
rish -c 'am force-stop com.example.myapp; am start -W -n com.example.myapp/.MainActivity'
rish -c 'ps -A | grep com.example.myapp'
```

If `rish` reports a timeout, restart Shizuku, authorize Termux, and remove battery optimization for both apps.

## Common Termux-only failures

| Symptom | Likely fix |
| --- | --- |
| AAPT2 will not execute | Use the Termux `aapt2` override in `gradle.properties`. |
| `OS not recognized` from Hermes | Use the QEMU Hermes wrapper above. |
| NDK tools fail with `Exec format error` | Use a Termux-compatible NDK layout, not the Google x86_64 host binaries. |
| CMake cannot find `ReactAndroidConfig.cmake` even though Prefab extracted it | Check the generated Prefab tree and apply the Termux-only CMake lookup adjustment in section 6. |
| Release app crashes with a missing Hermes DSO | Build with Hermes enabled so the Hermes native libraries are packaged. |
| Debug app only opens when Metro is running | Build and install `assembleRelease`; it embeds `index.android.bundle`. |
| `rish` disconnects | Restart Shizuku and exclude both Shizuku and Termux from battery optimization. |

## Security note

Do not commit `android/local.properties`, keystores, access tokens, or device-specific paths. The default React Native `.gitignore` should exclude `local.properties`, build output, and `node_modules`.

## Codex skill package

The reusable Codex skill is in [skill/termux-react-native-apk-build](skill/termux-react-native-apk-build). To use it locally, copy that folder into `~/.codex/skills/`, then start a new Codex session. Other agents can clone this repository and use the same `SKILL.md` package.
