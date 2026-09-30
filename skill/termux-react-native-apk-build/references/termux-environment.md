# Termux native-build reference

Install the core tools:

```sh
pkg update
pkg install nodejs openjdk-17 git curl unzip zip \
  cmake ninja make aapt2 d8 apksigner ndk-multilib qemu-user-x86-64
```

For an ARM64-only React Native build, add to `android/gradle.properties`:

```properties
hermesEnabled=true
android.aapt2FromMavenOverride=/data/data/com.termux/files/usr/bin/aapt2
reactNativeArchitectures=arm64-v8a
```

Set `android/local.properties` to the SDK and compatible NDK layout:

```properties
sdk.dir=/data/data/com.termux/files/home/android-sdk
cmake.dir=/data/data/com.termux/files/usr
ndk.dir=/data/data/com.termux/files/home/termux-ndk
```

The NDK directory must expose the Google NDK CMake metadata but use Termux-native LLVM binaries, headers, libraries, and Clang runtime files. A stock `linux-x86_64` Google NDK host toolchain will not execute on an ARM64 phone.

Create a Hermes wrapper at `scripts/hermesc-qemu.sh`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
script_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
exec /data/data/com.termux/files/usr/bin/qemu-x86_64 \
  "$script_dir/../node_modules/hermes-compiler/hermesc/linux64-bin/hermesc" "$@"
```

Make it executable and configure it inside the `react {}` block of `android/app/build.gradle`:

```sh
chmod 700 scripts/hermesc-qemu.sh
```

```groovy
hermesCommand = "$rootDir/../scripts/hermesc-qemu.sh"
```

Build from `android`:

```sh
./gradlew assembleRelease --no-daemon
apksigner verify --verbose app/build/outputs/apk/release/app-release.apk
unzip -l app/build/outputs/apk/release/app-release.apk | grep index.android.bundle
```

For a Shizuku/rish installation, copy the APK to shared storage then invoke the package manager as shell:

```sh
rish -c 'cp /storage/emulated/0/Download/MyApp-release.apk /data/local/tmp/MyApp-release.apk && cmd package install -r /data/local/tmp/MyApp-release.apk'
```
