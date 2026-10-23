# BUILD ENVIRONMENT SETUP — Ubuntu 24.04 (Noble Numbat)

This document describes how to set up the build environment for **Liseur** on Ubuntu 24.04.

## Prerequisites

- Ubuntu 24.04 LTS (or newer)
- JDK 17+ (the build uses OpenJDK 21; `apt-get install openjdk-21-jdk-headless` will install it)
- Gradle Wrapper (`gradlew`) is already included in the project

## Android SDK Setup

The build requires the Android SDK. The following steps will install and configure it:

```bash
# 1. Install the Android SDK packages from Ubuntu repositories
sudo apt-get update
sudo apt-get install -y \
    android-sdk \
    android-sdk-platform-tools \
    android-sdk-build-tools \
    android-sdk-common \
    android-sdk-platform-23

# 2. Set up the SDK manager (sdkmanager) in the expected location
#    The default SDK path is /usr/lib/android-sdk
mkdir -p /usr/lib/android-sdk/cmdline-tools/latest

# Copy sdkmanager and support files from the React Native downloads
# (or download from developer.android.com if you prefer)
cp /root/react-native-helloworld/downloads/cmdline-tools/* \
    /usr/lib/android-sdk/cmdline-tools/latest/

# 3. Accept all SDK licenses
yes | ANDROID_SDK_ROOT=/usr/lib/android-sdk \
    /usr/lib/android-sdk/cmdline-tools/latest/bin/sdkmanager --licenses

# 4. Install the required SDK packages (Build Tools 36 and Platform 37)
#    These will be downloaded and installed automatically by Gradle when
#    building, but having them pre-installed speeds up the first build.
ANDROID_SDK_ROOT=/usr/lib/android-sdk \
    /usr/lib/android-sdk/cmdline-tools/latest/bin/sdkmanager \
    "build-tools;36.0.0" "platforms;android-37.0"
```

## Local Properties

Create `local.properties` in the project root with the SDK path:

```
sdk.dir=/usr/lib/android-sdk
```

This file is already created by the setup script above.

## Build Flavors

The project has two build flavors:

| Flavor | Description |
|--------|-------------|
| `play` (default) | Includes Gemini read-aloud, Gemini translation, and ML Kit translator. Google Play only. Code lives under `app/src/play/`. |
| `foss` | F-Droid build. No Gemini/ML Kit features. |

## Common Makefile Targets

| Target | Command | Description |
|--------|---------|-------------|
| `make build` | `./gradlew assemblePlayDebug` | Debug APK with Gemini/ML Kit |
| `make build foss` | `./gradlew assembleFossDebug` | F-Droid debug APK |
| `make release` | `./gradlew assembleFossRelease` | F-Droid release APK |
| `make bundle` | `./gradlew bundlePlayRelease` | Google Play AAB |
| `make test` | Runs JVM unit tests for both flavors |
| `make lint` | Runs Android Lint for both flavors |
| `make check` | Runs tests + lint + `build-all` (both flavors) |
| `make clean` | `./gradlew clean` | Remove build outputs |
| `make dev` | `./gradlew assemblePlayDev` | Debug APK with dev package (side-by-side, safe on phone) |
| `make dev-install` | `adb install -r` beside real app |
| `make dev-run` | Launch dev app |

## Dev Build (side-by-side, safe on phone with real library)

```bash
make dev             # assemblePlayDev (or assembleFossDev with FLAVOR=foss)
make dev-install     # adb install -r beside real app
make dev-run         # launch dev app
make dev-uninstall
```

The dev build uses its own package (`com.chmouel.liseur.dev`) with its own Room database, DataStore, SAF grant and server device key. It cannot reach the real app's data.

## Signing a Release Build (optional)

The release key lives in `pass`:

| Entry | Contents |
| --- | --- |
| `android/liseur.keystore.p12` | the PKCS#12 keystore, base64 |
| `android/liseur.keystore-password` | store and key password |
| `android/liseur.keystore-alias` | key alias (`liseur`) |

To generate `keystore.properties`:

```bash
signing_dir="${XDG_DATA_HOME:-$HOME/.local/share}/liseur/signing"
install -d -m 700 "$signing_dir"
umask 077
pass show android/liseur.keystore.p12 | base64 -d > "$signing_dir/release.p12"
{
  echo "storeFile=$signing_dir/release.p12"
  echo "storePassword=$(pass show android/liseur.keystore-password)"
  echo "keyAlias=$(pass show android/liseur.keystore-alias)"
  echo "keyPassword=$(pass show android/liseur.keystore-password)"
} > keystore.properties
```

Without the key, Gradle will build unsigned APKs (acceptable for F-Droid/GitHub releases but not for Google Play).

## Quick Start Checklist

```bash
# 1. Install SDK packages
sudo apt-get install -y android-sdk android-sdk-platform-tools ...

# 2. Set up sdkmanager licenses
yes | ANDROID_SDK_ROOT=/usr/lib/android-sdk \
    /usr/lib/android-sdk/cmdline-tools/latest/bin/sdkmanager --licenses

# 3. Create local.properties
echo 'sdk.dir=/usr/lib/android-sdk' > local.properties

# 4. Build debug APK (play flavor)
make dev

# 5. Or build foss flavor
FLAVOR=foss make dev
```