---
name: skip-project-creation
description: Creating new Skip app and library projects. Handles skip init commands for both Skip Lite (transpiled) and Skip Fuse (native) modes, adding optional framework dependencies to Package.swift, and scaffolding template code for frameworks like SkipSQL, SkipFirebase, SkipKeychain, SkipWeb, and others.
---

# Skip Project Creation

Create new Skip apps and libraries using `skip init`, then optionally add framework dependencies with template code.

## Prerequisites

```bash
brew install skiptools/skip/skip
skip checkup
```

You also need:
- **Xcode** with Command Line Tools (for `xcodebuild` and the iOS Simulator).
- **Android Studio** with at least one AVD created (for the SDK and an emulator). The Skip tooling assumes one is ready.
- **Maestro** (`brew install mobile-dev-inc/tap/maestro`) if you intend to write UI flows — see [skip-ui-automation](../../../skip-testing-deployment/skills/skip-ui-automation/SKILL.md).

## Pre-flight: boot simulators yourself before launching

`skip app launch` does **not** boot the iOS Simulator for you, and does **not** start an Android emulator. If neither is running, the build succeeds and the install step silently fails. Start them yourself first:

```bash
# iOS — pick a UDID from `xcrun simctl list devices available`
xcrun simctl boot <UDID>
open -a Simulator

# Android — list AVDs, start one
emulator -list-avds
emulator -avd Pixel_6_API_34   # blocks; run in another terminal

# Confirm
xcrun simctl list devices booted
adb devices
```

When you have both a USB-connected Android device *and* a running emulator, scope the install with `ANDROID_SERIAL=emulator-5554 skip app launch` — there's no `--android-serial` flag on `skip app launch` itself.

## `skip create` vs `skip init`

Skip ships two scaffolding commands. They produce the same kind of project; the difference is how you supply the answers.

- **`skip create`** is interactive. It walks you through prompts for every decision: project type (app vs. library), mode (Lite vs. Fuse), project name, module names, bundle id, whether to create test modules, license, App Fair participation, git repo, fastlane, pre-build, Android SDK install (for Fuse), and whether to open Xcode at the end. Use it when you're not sure what flags you want or you want a guided walk-through.
- **`skip init`** is non-interactive and takes everything as positional args and flags. Use it when scripting, when running from CI, or when you already know the exact shape you want.

```bash
skip create                                                # interactive
skip init --transpiled-app --appid=com.example.myapp my-app MyApp MyAppModel  # flag-driven
```

## Creating a New App

### Skip Lite (Transpiled) App

Swift is transpiled to Kotlin. Best for most projects.

```bash
skip init --transpiled-app --appid=com.example.myapp my-app MyApp MyAppModel
```

- `--transpiled-app` — Skip Lite mode (Swift transpiled to Kotlin)
- `--appid=BUNDLE_ID` — Required bundle identifier
- `my-app` — Project folder name (lowercase, hyphens OK)
- `MyApp` — UI module name (CamelCase)
- `MyAppModel` — Model module name (optional second module for separation)

### Skip Fuse (Native) App

Swift compiles natively on Android. Full Swift language support.

```bash
skip init --native-app --appid=com.example.someapp some-app SomeApp
```

- `--native-app` — Skip Fuse mode (native Swift compilation)
- Fuse apps typically use a single module (no separate model module)

Before running this for the first time, install the Swift Android SDK with `skip android sdk install` and verify with `skip checkup --native` — see [skip-fuse](../skip-fuse/SKILL.md) for the full Fuse setup.

### Flags — full reference

The flags below are supported on both `skip init` and `skip create`. Default values come from `skipstone/Sources/SkipBuild/Commands/PackageCommand.swift`.

**Mode (mutually exclusive, required for app/library):**

| Flag | Purpose |
|---|---|
| `--transpiled-app` | Skip Lite (transpiled) app |
| `--native-app` | Skip Fuse (native compiled) app |
| `--transpiled-model` | Transpiled library / framework module |
| `--native-model` | Native library / framework module |

**Identity and versioning:**

| Flag | Default | Purpose |
|---|---|---|
| `--appid=BUNDLE_ID` | (required) | Bundle identifier for apps |
| `--version=VER` | none | Initial app version (e.g. `1.0.0`) |
| `--ios-min-version=V` | `17.0` | Minimum iOS deployment target |
| `--macos-min-version=V` | derived | Minimum macOS target; defaults to `iOS - 3` |
| `--swift-package-version=V` | `6.1` | `swift-tools-version` in `Package.swift` |

**Build and run:**

| Flag | Default | Purpose |
|---|---|---|
| `--apk` / `--no-apk` | true | Build the Android APK in the initial build |
| `--ipa` / `--no-ipa` | true | Build the iOS IPA in the initial build |
| `--no-build` | — | Skip the initial build entirely |
| `--open-xcode` | — | Open the project in Xcode after creation |
| `--open-gradle` | — | Open the generated Gradle project after creation |

**Icons:**

| Flag | Purpose |
|---|---|
| `--icon=PATH` | Custom icon source (SVG, PDF, PNG); can repeat for per-platform sources |
| `--icon-background=COLOR` | Background colour or gradient (e.g. `#1A73E8`, `#5C6BC0-#3B3F54`) |
| `--icon-foreground=COLOR` | Foreground colour (overlay tint) |
| `--icon-inset=DECIMAL` | Foreground inset fraction (e.g. `0.25`) |
| `--icon-shadow=DECIMAL` | Shadow radius |
| `--no-icon` | Skip icon generation entirely |

**Modules and tests:**

| Flag | Default | Purpose |
|---|---|---|
| `--bridged` / `--no-bridged` | false | For transpiled models, generate bridge-mode declarations for Fuse consumers |
| `--module-tests` / `--no-module-tests` | auto | Create test modules (auto: yes for Lite, no for Fuse) |
| `--chain` / `--no-chain` | true | When multiple module names are given, chain dependencies (each depends on the next) |
| `--test-case-mode=MODE` | `testing` | Test framework for generated tests: `testing` (Swift Testing) or `xctest` |
| `--resource-path=PATH` | `Resources` | Resource folder name |
| `--kotlincompat` / `--no-kotlincompat` | false | For native libraries, use Kotlin-compatible bridging types in the generated wrappers |

**Project housekeeping:**

| Flag | Default | Purpose |
|---|---|---|
| `--git-repo` / `--no-git-repo` | false | Initialise a git repository in the project root |
| `--github` / `--no-github` | false | Create GitHub metadata (`.github/` workflow files, etc.) |
| `--fastlane` / `--no-fastlane` | true | Generate fastlane configuration (`Darwin/fastlane/`, `Android/fastlane/`) |
| `--free` | — | Use a free-software license |
| `--show-tree` / `--no-show-tree` | — | Print a file-system tree of the created project at the end |
| `--validate-package` / `--no-validate-package` | true | Validate the generated `Package.swift` |

**Module dependency syntax:**

A positional module name can include inline dependency declarations using `:`:

```bash
# Module "MainApp" depends on the SkipUI product from skiptools/skip-ui at version 1.0
skip init --transpiled-app --appid=com.example.demo demo MainApp:skiptools/skip-ui@1.0.0/SkipUI
```

Format: `ModuleName:[org/]repo[@version]/DependencyModule`. Defaults to `skiptools` if no org is given. Useful for one-off scaffolds that pull in a specific Skip framework without editing `Package.swift` after creation.

### Creating Libraries

```bash
# Transpiled (Lite) library
skip init --transpiled-model my-lib MyLib

# Native (Fuse) library
skip init --native-model my-lib MyLib

# A bridged Lite library (for Fuse consumers)
skip init --transpiled-model --bridged my-lib MyLib
```

## Adding Dependencies After Creation

After `skip init` creates the project, add optional framework dependencies by modifying `Package.swift` and adding template code.

### Step-by-step

1. **Add the package dependency** to the top-level `dependencies` array in `Package.swift`
2. **Add the product** to the appropriate target's `dependencies` array
3. **Configure `skip.yml`** if the framework needs Android Gradle dependencies
4. **Add template code** to the appropriate source file (UI module or Model module)

### Where to Add Code

- **UI code** (views, navigation): `Sources/<AppModule>/ContentView.swift`
- **Model/data code** (storage, network, auth): `Sources/<ModelModule>/ViewModel.swift`
- **If single module**: Both go in the app module's source files

## Dependency Templates

Consult the detailed dependency templates reference:
- ./references/dependency-templates.md -- Package.swift entries, skip.yml config, and starter code for each framework

### Quick Reference: Package.swift Dependency Syntax

Stable frameworks use `from:` versioning:
```swift
.package(url: "https://source.skip.tools/skip-sql.git", from: "1.0.0"),
```

Pre-stable frameworks use range versioning:
```swift
.package(url: "https://source.skip.tools/skip-firebase.git", "0.0.0"..<"2.0.0"),
```

Product dependencies go in the target that uses them:
```swift
.target(name: "MyApp", dependencies: [
    "MyAppModel",
    .product(name: "SkipUI", package: "skip-ui"),
    .product(name: "SkipSQL", package: "skip-sql"),  // added
], ...),
```

## Example Workflows

### "Create a Skip Lite app called MyApp with SQLite"

```bash
skip init --transpiled-app --appid=com.example.myapp my-app MyApp MyAppModel --no-build
```

Then add SkipSQL to `Package.swift` in the model target's dependencies, and add the database helper code to `Sources/MyAppModel/ViewModel.swift`.

### "Create a Skip Fuse app called ChatApp with Firebase"

```bash
skip init --native-app --appid=com.example.chatapp chat-app ChatApp --no-build
```

Then add SkipFirebase packages to `Package.swift`, place config files, and add initialization code.

### "Create a Skip Lite app with Keychain and Web"

```bash
skip init --transpiled-app --appid=com.example.demo demo-app DemoApp DemoAppModel --no-build
```

Then add SkipKeychain and SkipWeb to `Package.swift` and add template code to the appropriate modules.
