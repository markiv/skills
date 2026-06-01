---
name: skip-frameworks
description: Integrating optional Skip frameworks into a project. Covers Firebase, SQLite, Keychain, WebView, audio/video, device sensors, Lottie animations, FFI, and other frameworks. Includes Package.swift dependencies, skip.yml configuration, and usage examples.
---

# Skip Frameworks

Use this skill when adding optional frameworks to a Skip project.

## Adding a Framework

1. Add package dependency to `Package.swift`
2. Add product to target dependencies
3. Configure `Skip/skip.yml` if Android-specific Gradle dependencies are needed
4. Import the module in Swift code

`Package.swift` is the source of truth for both iOS and Android builds — never use Xcode's "Add Package…" menu, which only updates Xcode's tracking and not Skip's Gradle generation. For the broader project layout (which target gets which product, where view code vs. model code lives, when to split into two modules), see [skip-project-creation](../skip-project-creation/SKILL.md).

## Skip frameworks vs. pure-Swift SwiftPM packages

There are two kinds of dependency a Skip project can pull in, and they behave differently on Android.

**Skip frameworks.** A SwiftPM package that contains a `Sources/<Module>/Skip/skip.yml` file is *Skip-aware*. The build plugin sees the `skip.yml`, generates the corresponding Gradle module, and builds it for Android automatically. All the frameworks listed in this skill (SkipSQL, SkipFirebase, etc.) are Skip frameworks.

**Pure-Swift SwiftPM packages.** A package without a `skip.yml` is not Skip-aware. Skip Lite modules **cannot consume pure-Swift packages on Android** — the transpiler has nothing to transpile against. On iOS the package compiles normally; on Android it's excluded from the build.

If you want a pure-Swift package only on iOS, gate it with the `condition:` argument:

```swift
.target(name: "MyApp", dependencies: [
    .product(
        name: "Lottie",
        package: "lottie-ios",
        condition: .when(platforms: [.macOS, .iOS])  // iOS-only
    ),
])
```

Without the `condition:`, the Lite build fails on Android with "no such Gradle dependency." With it, the Android build excludes the dependency and any code that references `Lottie` must be guarded with `#if !SKIP`.

Skip Fuse modules can consume pure-Swift packages directly because the Swift Android SDK compiles them natively — no transpilation needed. Be aware that the pure-Swift package's own dependencies and platform support must extend to Android, which most don't.

## Framework Reference

Consult the detailed reference files:
- ./references/firebase.md -- Firebase setup, auth, Firestore, messaging, crashlytics
- ./references/data-storage.md -- SkipSQL, SkipKeychain, UserDefaults
- ./references/media-web.md -- SkipWeb, SkipAV, SkipMotion

## Quick Reference

### SkipSQL — SQLite

```swift
// Package.swift
.package(url: "https://source.skip.tools/skip-sql.git", from: "1.0.0"),
.product(name: "SkipSQL", package: "skip-sql"),
```

```swift
import SkipSQL
let db = try SQLContext(path: dbPath)
try db.execute(sql: "CREATE TABLE items (id INTEGER PRIMARY KEY, name TEXT)")
try db.execute(sql: "INSERT INTO items (name) VALUES (?)", parameters: [.text("Widget")])
```

### SkipKeychain — Secure Storage

```swift
// Package.swift
.package(url: "https://source.skip.tools/skip-keychain.git", "0.0.0"..<"2.0.0"),
.product(name: "SkipKeychain", package: "skip-keychain"),
```

```swift
import SkipKeychain
Keychain.shared.set("secret", forKey: "token")
let token = Keychain.shared.string(forKey: "token")
```

### SkipFirebase — Firebase

```swift
// Package.swift (add only what you need)
.package(url: "https://source.skip.tools/skip-firebase.git", "0.0.0"..<"2.0.0"),
.product(name: "SkipFirebaseAuth", package: "skip-firebase"),
.product(name: "SkipFirebaseFirestore", package: "skip-firebase"),
```

Setup: `GoogleService-Info.plist` in `Darwin/`, `google-services.json` in `Android/app/`.

### SkipWeb — WebView

```swift
.package(url: "https://source.skip.tools/skip-web.git", from: "1.0.0"),
.product(name: "SkipWeb", package: "skip-web"),
```

### SkipAV — Audio/Video

```swift
.package(url: "https://source.skip.tools/skip-av.git", from: "1.0.0"),
.product(name: "SkipAV", package: "skip-av"),
```

### SkipDevice — Location, Sensors, Network

```swift
.package(url: "https://source.skip.tools/skip-device.git", "0.0.0"..<"2.0.0"),
.product(name: "SkipDevice", package: "skip-device"),
```

### Other Frameworks

| Framework | Package | Purpose |
|-----------|---------|---------|
| SkipMotion | `skip-motion` | Lottie animations |
| SkipFFI | `skip-ffi` | C/C++ interop via JNA |
| SkipBluetooth | `skip-bluetooth` | Bluetooth LE |
| SkipNFC | `skip-nfc` | NFC reading/writing |
| SkipQRCode | `skip-qrcode` | QR code generation/scanning |
| SkipXML | `skip-xml` | XML parsing |
| SkipZip | `skip-zip` | ZIP compression |
| SkipScript | `skip-script` | JavaScript engine |
| SkipSupabase | `skip-supabase` | Supabase client |
| SkipStripe | `skip-stripe` | Stripe payments |
| SkipSentry | `skip-sentry` | Error tracking |
| SkipPostHog | `skip-posthog` | Analytics |

## `skip.yml` — adding Android Gradle dependencies

When a framework needs a Maven artefact that isn't picked up automatically, declare it in the module's `Sources/<Module>/Skip/skip.yml` via the `build:` block. The syntax mirrors the structure of a Gradle build script.

```yaml
skip:
  mode: 'transpiled'

build:
  contents:
    - block: 'dependencies'
      contents:
        - 'implementation("io.livekit:livekit-android:2.24.0")'
        - 'implementation("com.google.android.gms:play-services-location:21.3.0")'
```

Each entry under the `dependencies` block is a literal Gradle declaration. `implementation(...)`, `api(...)`, `testImplementation(...)` all work.

To add a Maven repository or override the default plugin set, use `settings:` (parallel structure):

```yaml
settings:
  contents:
    - block: 'dependencyResolutionManagement'
      contents:
        - block: 'repositories'
          contents:
            - 'maven("https://maven.example.com/releases")'
```

Useful when a library is hosted outside Maven Central / Google's Maven repo.

### Full `skip.yml` schema

| Key | Purpose |
|---|---|
| `skip.mode` | `'transpiled'` (Lite), `'native'` (Fuse), or `'automatic'` |
| `skip.bridging` | Fuse-only. `true` for auto-bridging; or `{auto: true, options: 'kotlincompat'}` |
| `skip.kotlinPackage` | Override the Kotlin package name derived from the Swift module name |
| `skip.resources` | Custom resource paths and processing mode |
| `skip.dynamicroot` | Fuse-only. Magic namespace prefix for `AnyDynamicObject` (see [skip-fuse](../skip-fuse/SKILL.md)) |
| `build.contents` | Gradle build-script additions (dependencies, plugins, etc.) |
| `settings.contents` | Gradle settings-script additions (repositories, plugin management) |

## Cross-references

- [skip-project-creation](../skip-project-creation/SKILL.md) — `Package.swift` layout, when to split into UI and Model modules, where each framework's setup code typically lands.
- [skip-lite-transpilation](../skip-lite-transpilation/SKILL.md) — rules to follow when the framework's example code goes into a transpiled module.
- [building-skip-ui](../building-skip-ui/SKILL.md) — when a framework ships UI components (SkipWeb, SkipMotion, SkipAV), how to integrate them in a SwiftUI view.
