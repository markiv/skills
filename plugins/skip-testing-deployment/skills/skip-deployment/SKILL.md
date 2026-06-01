---
name: skip-deployment
description: Exporting, signing, and distributing Skip apps. Covers skip export command, iOS archive and App Store submission, Android APK/AAB signing, ProGuard, app icons, Skip.env metadata, and CI/CD with GitHub Actions.
---

# Skip Deployment

Build, sign, and distribute Skip apps for the App Store and Google Play.

## Building for Distribution

```bash
skip export                     # Both platforms, debug
skip export -c release          # Both platforms, release
skip export --android           # Android only
skip export --ios               # iOS only
skip export -c release -d ./out # Custom output directory
```

## App Metadata

### Skip.env — the shared config file

`Skip.env` lives at the project root and is the single source of truth for app-level metadata. It's an xcconfig-style file consumed by **both** the Xcode build (via `Darwin/ProjectName.xcconfig`, which `include`s it) and the Gradle build (via `Android/settings.gradle.kts`, which parses it).

Skip's CLI commands (`skip app launch`, `skip export`, `skip meta index`) all read `Skip.env` to know what to build.

Recognised keys:

```properties
# Identity
PRODUCT_NAME=MyApp                          # app display name; matches Swift module name
PRODUCT_BUNDLE_IDENTIFIER=com.example.myapp # iOS bundle ID

# Versioning
MARKETING_VERSION=1.0.0                     # semantic version (CFBundleShortVersionString)
CURRENT_PROJECT_VERSION=1                   # build number (CFBundleVersion / versionCode)

# Android-specific
ANDROID_PACKAGE_NAME=com.example.myapp      # Gradle package
ANDROID_APPLICATION_ID=com.example.myapp    # Play Console app ID (often same as package)

# Store linking (used by `skip meta index`)
APPLE_APP_STORE_ID=1234567890               # numeric App Store ID
GOOGLE_PLAY_STORE_ID=com.example.myapp      # Play Store app ID
```

Edit `Skip.env` in one place; both platform builds pick up the change.

### `SKIP_ACTION` — control the Android build during iOS iteration

The `Darwin/ProjectName.xcconfig` file recognises a `SKIP_ACTION` knob that controls whether the Skip build plugin builds and/or launches the Android side when you build the iOS scheme from Xcode.

| Value | Behaviour |
|---|---|
| `launch` (default) | Build the Android APK and install + launch it on the running emulator alongside the iOS launch. |
| `build` | Build the Android APK but don't launch it. |
| `none` | Skip the Android build entirely. |

Set it in `Darwin/ProjectName.xcconfig`:

```
SKIP_ACTION = launch
```

`SKIP_ACTION = none` is the right setting for short bursts of iOS-only iteration when you're tuning UI on the Apple side and don't want every save to re-run the Gradle build. Don't leave it on for long — Android-specific regressions accumulate silently.

### App Icons

```bash
skip icon icon-1024.png                          # source is a positional argument
skip icon --background "#1A73E8" --inset 0.25 logo.svg
```

See [skip-icons](../../../skip-app-design/skills/skip-icons/SKILL.md) for the full flag list and the in-app `.symbolset` workflow.

## iOS Distribution

Archive via Xcode: **Product > Archive**, then **Distribute App** in Organizer.

Or via command line:
```bash
xcodebuild archive -workspace Project.xcworkspace -scheme MyApp -configuration Release -archivePath ./build/MyApp.xcarchive
xcodebuild -exportArchive -archivePath ./build/MyApp.xcarchive -exportOptionsPlist ExportOptions.plist -exportPath ./build/
```

## Android Distribution

`skip export` produces both a debug and a release APK under `.build/skip-export/`:

- `.build/skip-export/AppName-debug.apk` — debug build, signed with a temporary key. Installs on emulators and devices for testing; you must uninstall before re-installing a newer version.
- `.build/skip-export/AppName.apk` — release build, unsigned by default.

### Signing the release APK

Skip-generated projects ship with a `build.gradle.kts` that already looks for a keystore at `Android/app/keystore.jks` and credentials at `Android/app/keystore.properties`. You do **not** edit `build.gradle.kts` by hand — drop the two files in and `skip export` signs the release APK automatically.

#### Step 1: create the keystore

From the project root:

```bash
keytool -genkeypair -v -keystore Android/app/keystore.jks \
    -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

Remember the passwords; you'll need them in the next step. (For complete signing-key management, see [Android's "Sign your app" docs](https://developer.android.com/studio/publish/app-signing).)

#### Step 2: write the credentials file

Create `Android/app/keystore.properties`:

```properties
keyAlias=app
storeFile=keystore.jks
storePassword=YOUR_KEYSTORE_PASSWORD
keyPassword=YOUR_KEY_PASSWORD
```

Add both files to `.gitignore` — they're sensitive.

#### Step 3: re-export

```bash
skip export -c release
```

The release APK at `.build/skip-export/AppName.apk` is now signed.

### Verifying the signature

```bash
~/Library/Android/sdk/build-tools/<version>/apksigner verify --verbose .build/skip-export/AppName.apk
```

`apksigner` is part of the Android build-tools that Skip's installer pulls down.

### Building an AAB for Play Store upload

`.apk` is for direct distribution and emulator install. The Play Store needs `.aab`:

```bash
gradle -p Android bundle
# Output: .build/Android/app/outputs/bundle/release/app-release.aab
```

The APK signature is the app's own signature; the Play Store also requires an **upload key** which is separate. For automated bundle signing and Play Store upload, fastlane is the documented route.

### Fastlane configuration for Play Store uploads

`skip init --fastlane` (or the `--fastlane` flag on existing app creation) generates a fastlane configuration under `Android/fastlane/` with the standard files (`Appfile`, `Fastfile`, `README.md`) plus the `metadata/android/<locale>/` tree.

To enable automated Play Store uploads, you also need a Google Play service-account credentials file:

1. Open [Google Play Console](https://play.google.com/console).
2. Navigate to **Settings → API access**.
3. Create a service account with the required Play permissions.
4. Download the JSON credentials file.
5. Save it to **`Android/fastlane/apikey.json`**.
6. Add `apikey.json` to `.gitignore` — it's sensitive.

With `apikey.json` in place, the standard fastlane `Fastfile` lanes (`fastlane release`, `fastlane beta`) can sign the AAB with the upload key and publish to the Play Store track of your choice.

The iOS side uses fastlane's `deliver` action for App Store Connect uploads. Credentials there come from an `apikey_appstore.json` file (App Store Connect API key) and an `Appfile` referencing your team ID.

For a complete CI workflow that wires both stores together, see [`references/cicd.md`](references/cicd.md).

## CI/CD

Consult the reference for a complete GitHub Actions workflow:
- ./references/cicd.md -- GitHub Actions workflow, environment setup, artifact upload

### Minimal GitHub Actions

```yaml
name: Build and Test
on: [push, pull_request]
jobs:
  build:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - name: Install Skip
        run: brew install skiptools/skip/skip
      - name: Test
        run: swift test
      - name: Export
        run: skip export -c release -d ./artifacts
      - uses: actions/upload-artifact@v4
        with:
          name: app-artifacts
          path: ./artifacts/
```

## Store metadata is translated separately

App Store Connect and Google Play storefront metadata — title, description, keywords, release notes, screenshots — lives under `Darwin/fastlane/metadata/<locale>/` and `Android/fastlane/metadata/android/<locale>/`. Each file has a hard byte-budget limit (titles 30 bytes, Play short_description 80 bytes, descriptions 4000 bytes), and the store-specific locale codes (`de-DE`, `pt-BR`, `zh-Hans`/`zh-CN`, etc.) don't exactly match the BCP-47 codes used in `Localizable.xcstrings`. Run `skip verify` and `skip meta index` to catch overruns and schema issues before submission.

For the full localization workflow — in-app `xcstrings` translation, `InfoPlist.xcstrings`, fastlane metadata, byte budgets, language-code mapping per store — see the [skip-localization](../../../skip-app-design/skills/skip-localization/SKILL.md) skill.

## Validation before exporting

Run the test suite *before* `skip export` — a release build that ships a Kotlin transpilation regression looks identical to a passing build until users open it on Android. The contract is:

```bash
swift test           # both Swift and transpiled Kotlin/JUnit
skip verify          # fastlane metadata schema + byte budgets
skip meta index      # full storefront metadata index
skip export -c release -d ./artifacts
```

See [skip-testing](../skip-testing/SKILL.md) for what `swift test` actually exercises (Swift side + auto-generated `XCSkipTests` harness that runs the same tests under Robolectric), and [skip-ui-automation](../skip-ui-automation/SKILL.md) for `maestro test` runs against a release-configuration install (`skip app launch --configuration release` then `maestro --device <UDID> test .maestro/*.yaml`).
