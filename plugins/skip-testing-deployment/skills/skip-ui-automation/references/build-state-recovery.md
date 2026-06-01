# Recovering from stale `skip app launch` build state

A handful of `skip app launch` failures are caused by stale build state rather than a real source error. This file documents the recoveries — try them before changing code.

## Stale `Bundle.module` resolution

**Symptom.** The first build after introducing `bundle: .module` references (in `Text(..., bundle: .module, ...)`, `Image("name", bundle: .module)`, etc.) fails with:

```
PackageSupport.kt:6:30 Unresolved reference 'Bundle'.
```

The transpiler generates a `PackageSupport.kt` with an extension on `skip.foundation.Bundle.Companion.module`, but the Kotlin classpath sees it before SkipFoundation is wired in by the build plugin.

**Anti-fix — do not do this.** Adding `SkipFoundation` as a direct dependency in `Package.swift` looks like the obvious patch:

```swift
.target(name: "MyApp", dependencies: [
    "MyAppModel",
    .product(name: "SkipFoundation", package: "skip-foundation"),  // ← do NOT add this
    .product(name: "SkipUI", package: "skip-ui"),
], ...)
```

…but produces a *worse* error on the next build:

```
ContentView.kt: error: Cannot access 'val module: Bundle': it is internal in 'skip.foundation.Bundle.Companion'.
```

SkipFoundation already defines `Bundle.module` as `internal`; the auto-generated `PackageSupport.kt` extension on the same `Companion` collides, and Kotlin can't disambiguate.

**Actual fix — clear the build cache.**

```bash
rm -rf .build
skip app launch
```

Skip's plugin generates the right `PackageSupport.kt` when the Kotlin build hasn't seen the prior failed state. Revert any `SkipFoundation` direct-dependency change before the clean rebuild.

If you still see the error on a clean tree, file an issue at `https://github.com/skiptools/skip` with the full Gradle log — it's a real bug, not a stale-state recovery.

## Stale Gradle / Compose-compiler classpath after a Skip upgrade

**Symptom.** After upgrading the Skip CLI (`brew upgrade skip`) or a `skip-ui` / `skip-foundation` version pin, `skip app launch` fails with one of:

- `Compose Compiler version mismatch`
- `Unresolved reference 'rememberSaveableStateHolder'`
- `Inconsistent JVM-target compatibility detected`

**Recovery.**

```bash
rm -rf .build
rm -rf Android/.gradle Android/app/build
skip app launch
```

The first delete drops Skip's transpiler output; the second clears Gradle's incremental cache, which holds resolved Compose-compiler bytecode that was correct under the old SkipUI but not the new one.

If the error references a specific Compose API the new SkipUI removed, you'll need to update the source — but always do a clean rebuild *first* to rule out staleness.

## Stale Xcode DerivedData

**Symptom.** `skip app launch --ios` succeeds but the running app shows the *previous* version's UI. Strings update, but layout / view-hierarchy changes don't take effect.

**Recovery.**

```bash
# Wipe Skip's own DerivedData
rm -rf .build/Darwin/DerivedData

# Or, for the global Xcode cache (slower next build, but exhaustive)
rm -rf ~/Library/Developer/Xcode/DerivedData/<your-scheme>-*
```

Skip's build phase uses Xcode's incremental build, which can get out of sync when the SwiftPM target structure changes (renamed module, added/removed target). The local `.build/Darwin/DerivedData/` is usually enough; the global cache only matters if you're switching between branches that differ in target topology.

## Stale Android install

**Symptom.** `skip app launch` reports `BUILD SUCCESSFUL`, the install step prints `Success`, but the running app shows the old UI. iOS picks up the new build; Android doesn't.

**Recovery.**

```bash
adb -s $SER uninstall dev.skip.todo   # or your app's appId
skip app launch
```

Android's package manager occasionally short-circuits an install when the version code didn't bump — `force-stop` is usually enough but a full uninstall is the reliable form. Bump `CURRENT_PROJECT_VERSION` in `Skip.env` to make this rarer.

## A "wipe everything" reset

When you've made enough changes that you're not sure which cache is stale, this is the nuclear reset:

```bash
rm -rf .build                                   # Skip transpiler output
rm -rf Android/.gradle Android/app/build        # Gradle incremental cache
rm -rf ~/Library/Developer/Xcode/DerivedData/<scheme>-* 2>/dev/null
xcrun simctl uninstall $SIM dev.skip.todo
adb -s $SER uninstall dev.skip.todo
skip app launch
```

Expect a ~3-minute cold rebuild. Use this only when iteration debugging has failed — never as a routine step, since it defeats the point of incremental builds.
