---
name: skip-fuse
description: Building Skip apps in Fuse mode — Swift compiled natively for Android via the Swift Android SDK (Swift 6.3+), with auto-generated JNI bridging between Swift and Kotlin/Java. Covers when to choose Fuse over Lite, project init with `skip init --native-app`, the `mode: 'native'` + `bridging: true` skip.yml configuration, the `import SkipFuse` requirement for `@Observable` models (a build-time warning, not an error), the internal/public visibility rules for views and properties, the ~60 MB Swift runtime binary cost, debugging via `OSLog.Logger` (breakpoints not yet supported), `ProcessInfo.processInfo.dynamicAndroidContext()` for reaching the Android Context, the Kotlin-inline-class string-interpolation workaround for Android SDK properties, `SKIP_BRIDGE=1` for dynamic library bundling, and the `skip android test` (CLI vs APK mode) test command. Use this skill whenever working on a project whose `skip.yml` contains `mode: 'native'`, or when deciding between Lite and Fuse for a new app.
---

# Skip Fuse

Fuse is Skip's "native Swift on Android" mode. Instead of transpiling Swift to Kotlin (Lite), Fuse compiles the same Swift source with the official Swift SDK for Android and generates JNI bridges so the Swift code can call into Kotlin/Java and vice versa. The SwiftUI surface still goes through SkipUI → Compose; only the Swift-language code is compiled rather than transpiled.

This skill covers the Fuse-specific workflow. For the rest of the Skip workflow (UI components, icons, localization, tests, deployment) the Lite skills apply unchanged.

## When to choose Fuse over Lite

| Choose Fuse when… | Choose Lite when… |
|---|---|
| You depend on Swift language features that transpile poorly: variadic generics, complex macros, reflection, key-paths-as-values, `Mirror`. | You're targeting the broadest base of Swift libraries and want the smallest APK. |
| You need C interop without `SkipFFI` boilerplate (Fuse can call C directly via the Swift Android SDK). | You're shipping a small, focused app where 60 MB of Swift runtime is the wrong trade-off. |
| You're consuming Swift packages that weren't written for Skip and want them to "just work". | You want maximum agent / tooling support — most Skip skills, examples, and sample apps target Lite. |
| You need bidirectional Swift↔Kotlin interop in the same project (a Swift Package that also calls Android SDK APIs). | You don't need to call Kotlin or Java APIs directly; SkipFoundation / SkipUI cover what you use. |

Default to **Lite** unless one of the left-column reasons applies. Fuse is more capable but more nascent — fewer skills, sample apps, and CI templates document it.

## Mode detection (always do this first)

Before writing any code, determine the project mode. The Swift you can safely write depends on it.

```bash
# Check each module's skip.yml
grep -l "mode: 'native'" Sources/*/Skip/skip.yml
```

- `mode: 'native'` → Fuse (this skill applies).
- `mode: 'transpiled'` or no mode set → Lite ([skip-lite-transpilation](../skip-lite-transpilation/SKILL.md) applies).
- `mode: 'automatic'` → depends on the resolved dependency graph. If `SkipFuse`/`SkipFuseUI` is pulled in (typically via `SKIP_MODE=fuse` at build time), it's Fuse; otherwise Lite.

## Prerequisites

Before starting a Fuse project, install the Swift SDK for Android and run the Fuse-specific environment checkup:

```bash
skip android sdk install        # one-time: download the Swift SDK for Android
skip checkup --native           # verify the native toolchain (above and beyond plain `skip checkup`)
```

`skip checkup --native` runs everything in `skip checkup` plus the additional checks needed for Fuse compilation. If you skip these, the first `skip app launch` will fail with a confusing toolchain error.

## Creating a Fuse project

```bash
skip init --native-app --appid=com.example.someapp some-app SomeApp
```

The flags that matter:

- `--native-app` — selects Fuse mode (vs. `--transpiled-app` for Lite).
- A Fuse app typically uses a **single module** — no separate `SomeAppModel`. The bridge generator works better with one module per app.
- All the other `skip init` flags from [skip-project-creation](../skip-project-creation/SKILL.md) (`--appid`, `--version`, `--no-build`, `--git-repo`, `--icon`) apply unchanged.

The generated `Sources/SomeApp/Skip/skip.yml` will contain:

```yaml
skip:
  mode: 'native'
  bridging: true
```

`bridging: true` is shorthand for the auto-bridging form (`bridging.auto: true`) — see [Bridging options](#bridging-options) below for the full schema and when you'd want to customise it.

## `@Observable` requires `import SkipFuse` (warning, not error)

Any Swift file that declares an `@Observable` type in a Fuse module needs `import SkipFuse` (or `import SkipFuseUI`, which re-exports it). Without it, the state-tracking glue that synchronises Swift observables with Jetpack Compose's recomposition is not installed, and the Android UI silently stops updating when the model changes.

```swift
import Foundation
import SkipFuse        // installs the @Observable bridge for Compose
import Observation

@Observable final class ViewModel {
    var items: [Item] = []
    var isLoading = false
}
```

This is enforced as a **warning**, not a compile error. The build emits:

> This file contains @Observables, but they will not be able to power your Android UI unless you 'import SkipFuse' or 'import SkipFuseUI'

The code compiles, the iOS build runs correctly, and the Android build runs without crashing — the bug only shows up at runtime as a UI that never updates when the model mutates. Treat the warning as a blocker during code review.

## Visibility rules for views

Views and their SwiftUI properties must use **default (internal) or public visibility**. `private` views and `private` `@State` / `@Binding` properties are not visible to the bridge generator and will not render on Android.

```swift
// BAD on Android — bridge generator can't see private members
private struct DetailRow: View {
    @State private var expanded = false
    var body: some View { ... }
}

// GOOD — default internal visibility
struct DetailRow: View {
    @State var expanded = false
    var body: some View { ... }
}
```

This applies in both Fuse and Lite, but it bites harder in Fuse because the bridge generator runs against the public API surface; in Lite the transpiler reaches private members through its own AST traversal.

## `#if os(Android)` vs. `#if SKIP` in Fuse

In Lite, `#if os(Android)` and `#if SKIP` are equivalent (everything is transpiled to Kotlin). In Fuse they're **different**:

| Conditional | Fuse behaviour |
|---|---|
| `#if os(Android)` | Compiled as **native Swift** for Android. Cannot call Kotlin/Java APIs directly. |
| `#if SKIP` | **Transpiled to Kotlin.** Can call Kotlin/Java API directly. |
| `#if !SKIP` | Native Swift on **both** iOS and Android. The right place for shared logic that should not be transpiled. |
| `#if !os(Android)` | iOS only. |

Rules of thumb:

- Need to call `android.content.Context`, `java.util.UUID`, a Kotlin coroutine? Use `#if SKIP`.
- Need platform-specific Swift that uses Foundation/Swift APIs only (e.g. an Android-specific file path)? `#if os(Android)` is fine and stays native.
- Shared business logic that should run as native Swift on iOS *and* Android? `#if !SKIP` is what you want.

## Bridging options

`bridging: true` in `skip.yml` is shorthand for the auto-bridging form. The full schema:

```yaml
skip:
  mode: 'native'
  bridging:
    auto: true
    options: 'kotlincompat'    # optional
```

| Form | Effect |
|---|---|
| `bridging: true` | Shorthand for `bridging.auto: true`. Auto-generates JNI bridges for **every public API** in the module. The default for `skip init --native-app`. |
| `bridging.auto: true` | Same as above, expanded. |
| `bridging.options: 'kotlincompat'` | Generated Kotlin wrappers use `kotlin.collections.List` / `kotlin.collections.Map` / `java.net.URI` instead of `skip.lib.Array` / `skip.foundation.URL`. Use this when the bridged API is consumed from native Kotlin / Compose code rather than from another transpiled Swift module. |

Without `kotlincompat`, the bridged surface is shaped for Skip Lite consumers (uses the Skip wrapper types). With `kotlincompat`, it's shaped for raw Kotlin consumers (uses standard Kotlin collections). For an app that mixes Lite and Fuse modules, leave it off. For a Fuse library you want to ship to Kotlin-side consumers, turn it on.

Per-declaration control via SKIP comment directives:

| Directive | Effect |
|---|---|
| `// SKIP @bridge` | Bridge this specific declaration even if auto-bridging is off. |
| `// SKIP @bridgeMembers` | Bridge all members of this type matching its visibility (all `public` if the type is `public`). |
| `// SKIP @nobridge` | Exclude this member from auto-bridging. |

Use `@nobridge` to selectively trim what crosses the JNI boundary — for example, a public method that returns a non-bridgeable type, where you only want it visible to native Swift callers.

## AnyDynamicObject — calling Kotlin/Java from Fuse without a wrapper

When a Fuse module needs to use a Kotlin or Java type from the Android SDK or a Kotlin library and there's no transpiled Swift wrapper for it, the documented escape hatch is `AnyDynamicObject`. It's a SkipFuse type that wraps any JVM object and lets native Swift call its methods and properties via Swift's dynamic dispatch features.

Raw form (no `dynamicroot`):

```swift
import SkipFuse

#if os(Android)
let date = try AnyDynamicObject(className: "java.util.Date", 999)

// Call instance methods. Return types and `throws` are mandatory.
let time: Int64 = try date.getTime()!

// Property syntax works for getters and setters.
let t = date.time!
date.time = 1001

// Chain calls freely.
let s = try date.instant?.toString()
#endif
```

Important characteristics:

- **All calls are unchecked at compile time.** You name the method, supply the args, and assert the return type. Wrong method name or wrong return type produces a runtime error, not a build error.
- **All calls are `throws` and `?`-prefixed.** Kotlin methods can throw, and Swift can't know whether they return null.
- **Android-only.** Always wrap in `#if os(Android)`.

### `dynamicroot` for cleaner syntax

Repeatedly typing `className: "java.util.Date"` is tedious. The `dynamicroot` option in `skip.yml` creates a magic namespace prefix for fully-qualified types:

```yaml
skip:
  mode: 'native'
  dynamicroot: 'D'        # any single uppercase letter or short identifier
```

With `dynamicroot: 'D'`:

```swift
import SkipFuse

#if os(Android)
let date = try D.java.util.Date(999)        // no string class name
typealias JDate = D.java.util.Date          // typealias for repeated use
let time: Int64 = try date.getTime()!

// Static methods use Companion()
let parsed = try D.java.util.Date.Companion().parse(dateString)!

// Convert a base AnyDynamicObject to a typed one
let dateAsTyped = baseAnyDynamicObject.as(D.java.util.Date.self)
#endif
```

`dynamicroot` is purely syntactic sugar over `AnyDynamicObject`. There's no runtime difference; the build plugin rewrites `D.foo.bar.Baz(args)` into the equivalent `AnyDynamicObject(className: "foo.bar.Baz", args)`.

### When to use AnyDynamicObject vs. a transpiled wrapper

| Use AnyDynamicObject when… | Write a transpiled wrapper when… |
|---|---|
| You need one or two Kotlin types in a Fuse app. | You're consuming a whole Kotlin library across many call sites. |
| The API surface is small enough to live with unchecked calls. | You want Swift-side type checking. |
| You don't want to maintain a parallel Swift declaration file. | You want IDE autocomplete and refactoring. |

For the deeper story (including how `dynamicroot` reaches into reflection, performance characteristics, and what it does for class hierarchies), see the [modes documentation](https://skip.dev/docs/modes/#anydynamicobject).

## `SKIP_BRIDGE` — the bridge-mode preprocessor symbol

When a transpiled module is built as a bridge target for native Swift consumption, the build plugin defines a `SKIP_BRIDGE` preprocessor symbol. Use `#if !SKIP_BRIDGE` to guard Swift code that would produce duplicate-symbol errors when the module is built in bridge mode:

```swift
#if !SKIP_BRIDGE
#if os(Android)
import android.content.Context
let context = ProcessInfo.processInfo.androidContext        // Lite-style call
#endif
#endif
```

The skipstone plugin emits a warning when it detects Android-specific code in a transpiled module that lacks the `#if !SKIP_BRIDGE` guard. Treat the warning as a blocker — without the guard, the module won't link in mixed Lite/Fuse projects.

Setting `SKIP_BRIDGE=1` as an environment variable when running `swift build` forces every library product in the resolved graph to be dynamic (`.dynamic` rather than `.automatic` / `.static`). This is occasionally needed when a Fuse module consumes a peer Fuse package across the JNI boundary:

```bash
SKIP_BRIDGE=1 swift build
SKIP_BRIDGE=1 swift test
SKIP_BRIDGE=1 skip app launch
```

Use it for the build of a Fuse-library-plus-Fuse-app project; you don't need it for a single-module Fuse app or for pure Lite projects.

## Binary size — the 60 MB Swift runtime

Fuse apps add approximately **60 MB** to the Android APK for the Swift runtime, Foundation, and internationalisation libraries. This is a fixed cost that does not grow with app complexity. It will decrease as the Swift Android SDK matures, but plan for it today:

- Verify your Play Store listing's APK size limit before shipping a Fuse app.
- Profile the APK with `apkanalyzer` (Android Studio) — confirm the 60 MB lives in `lib/` (the Swift stdlib `.so` files) and not in app code.
- Lite apps land at ~10–15 MB for the same UI surface. If your app is content-heavy on the iOS side, the relative overhead is smaller; if it's small, the overhead dominates.

## Debugging Fuse on Android

Two limitations to plan around:

### Breakpoint debugging of native Swift on Android is not yet supported

LLDB-on-Android for the Swift Android SDK is in development but not generally available. Today:

- Use **`OSLog.Logger`** (not `print()`) for runtime tracing. `OSLog` is bridged to Logcat in Fuse, so `logger.info(...)` lands in `adb logcat -s <tag>`. `print()` goes to stdout, which Android does not route to Logcat.
- For crash investigation, run `swift demangle` on the mangled Swift symbols in the crash dump — they look like `$s9SomeApp11ViewModelC4loadyyYaKF` and demangle to `SomeApp.ViewModel.load() async throws -> ()`.

```swift
import OSLog
let logger = Logger(subsystem: "com.example.someapp", category: "ViewModel")

func loadItems() async {
    logger.info("starting load")    // visible in adb logcat
    // print("starting load")       // NOT visible in Logcat, even though it works on iOS
}
```

### Kotlin inline classes in Android SDK APIs

Some Android SDK and Kotlin-library APIs (common in Compose, LiveKit, AndroidX) expose properties whose types are Kotlin **inline classes** (`@JvmInline value class Dp(...)`, `Color`, `TextUnit`, etc.). The JVM-level getter for an inline-class property has a *mangled* name (`getColor-impl` rather than `getColor`), and Fuse's bridge generator can't resolve `.stringValue` or direct casts against these.

The reliable workaround is **string interpolation**:

```swift
#if SKIP
import androidx.compose.ui.unit.Dp

let height: Dp = ...
// BAD — fails to resolve the mangled getter
let s = height.stringValue

// GOOD — string interpolation forces a toString() call
let s = "\(height)"
#endif
```

This trades type safety for compile-time guarantee and is documented in the Skip codebase as the recommended pattern.

## Running tests on Fuse

`swift test` runs the Swift-side tests as usual. For the Android side, Fuse has its own runner:

```bash
skip android test               # Fuse CLI mode — fast, no Android APIs available
skip android test --apk         # Fuse APK mode — full Android APIs, slower
```

| Mode | What runs | When to use |
|---|---|---|
| `swift test` | Swift tests on the host (macOS/Linux). | All pure-Swift logic. |
| `skip android test` | Fuse tests in a CLI environment on the emulator/device. No Android `Context`, no UI. | Fast iteration on cross-platform logic. |
| `skip android test --apk` | Tests run inside the packaged APK on the emulator/device with full Android APIs. | Anything that touches `ProcessInfo.processInfo.dynamicAndroidContext()`, system services, or asset bundles. |

The `--android-serial` flag scopes to a specific device when multiple are connected:

```bash
skip android test --android-serial emulator-5554
skip android test --android-serial auto   # auto-detect, prefer emulators
```

For the broader test-suite story (parity testing across iOS and Android, `XCSkipTests` harness, Robolectric — all of which apply to Lite, not Fuse), see [skip-testing](../../../skip-testing-deployment/skills/skip-testing/SKILL.md).

## Accessing the Android Context

In Fuse, `ProcessInfo.processInfo.dynamicAndroidContext()` returns the Android `Context` as an `AnyDynamicObject` — the bridge-time wrapper that lets native Swift call Kotlin / Java APIs without writing a transpiled wrapper. Use it whenever you need a system service, shared preferences, an asset, or any Android API that requires a context.

```swift
import SkipFuse

#if os(Android)
let context = ProcessInfo.processInfo.dynamicAndroidContext()
if let packageName: String = try? context.getPackageName() {
    // ...
}
#endif
```

The `androidContext` (no parentheses) form is the **Lite** spelling; in Fuse, calling that on `ProcessInfo` won't compile. Use the `dynamicAndroidContext()` method in Fuse and reach the same API surface through `AnyDynamicObject`'s dynamic-dispatch syntax. For the Activity, the parallel is `UIApplication.shared.dynamicAndroidActivity()`.

For more on `AnyDynamicObject` and the `dynamicroot` machinery that backs these accessors, see the [Skip Fuse / Bridging Modes docs](https://skip.dev/docs/modes/#anydynamicobject).

## Infrastructure packages

The Fuse mode depends on a small set of foundational packages (you usually don't pin these yourself — they come in via `SkipFuseUI`):

| Package | Purpose |
|---|---|
| `swift-jni` | C headers and Swift wrappers for the JNI surface. |
| `swift-android-native` | Wrappers for Android NDK APIs (`Logging`, `Context`, `AssetManager`, `Choreographer`, `Looper`). |
| `skip-android-bridge` | Android-specific bridging glue between Swift and Kotlin. |
| `skip-fuse` | Umbrella for `OSLog`, `@Observable`, `AnyDynamicObject` — the Swift side of bridging. |
| `skip-fuse-ui` | Native Swift UI types for Android (the Fuse equivalent of SkipUI for Lite). |

You'll see them appear in `Package.resolved` and the SwiftPM build graph but rarely need to touch them.

## Cross-references

- [skip-project-creation](../skip-project-creation/SKILL.md) — `skip init` flags, project layout decisions, when to use one vs. two modules.
- [skip-lite-transpilation](../skip-lite-transpilation/SKILL.md) — the parallel skill for Lite mode. Rules from there do **not** apply to Fuse files (no integer overflow concerns, no `Double == Int` issue, etc.), but apply unchanged to any `#if SKIP` block within a Fuse module.
- [building-skip-ui](../building-skip-ui/SKILL.md) — view code, which is the same in both modes. The visibility rule (no `private` views/properties) is enforced more strictly in Fuse.
- [skip-frameworks](../skip-frameworks/SKILL.md) — most frameworks work in both modes. Pay attention to per-framework notes on Fuse compatibility.
- [skip-testing](../../../skip-testing-deployment/skills/skip-testing/SKILL.md) — `swift test` on the Swift side; `skip android test` for the Fuse-on-Android side.
