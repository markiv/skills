---
name: skip-app-walkthrough
description: Start-to-finish orchestrator for building a new cross-platform Skip app. Sequences the other Skip skills in the order you actually need them — scaffold with `skip init`, write SwiftUI, add Material Symbols icons, validate Kotlin transpilation with `swift test`, drive the running app with `skip app launch` + Maestro, localise via `Localizable.xcstrings`, and ship via `skip export` with fastlane metadata. Use this skill at the start of a new project, or when picking up an existing Skip app and trying to decide which skill applies to which phase. Each phase links to the dedicated skill for the deep dive. Also points at the `skipapp-todo` working example at https://github.com/skiptools/skills as a reference build.
---

# Skip App Walkthrough

This skill is an index. It tells you which Skip skill applies to which phase of building an app, in the order you actually need them. Each phase has a one-paragraph summary and a link to the dedicated skill for the full workflow.

If you're starting a new app, read this top to bottom. If you're picking up an existing one, jump to the phase that matches what you're doing.

## The build, in order

| Phase | What you're doing | Skill |
|---|---|---|
| 1 | Decide between Lite (transpiled) and Fuse (native) modes | [skip-fuse](../skip-fuse/SKILL.md) for the trade-off table |
| 2 | Scaffold the project with `skip init` | [skip-project-creation](../skip-project-creation/SKILL.md) |
| 3 | Write SwiftUI view code | [building-skip-ui](../building-skip-ui/SKILL.md) |
| 4 | Add icons via `.symbolset` resources (never `systemName:` / `systemImage:`) | [skip-icons](../skip-icons/SKILL.md) |
| 5 | Write Swift model code that transpiles cleanly | [skip-lite-transpilation](../skip-lite-transpilation/SKILL.md) (Lite) or [skip-fuse](../skip-fuse/SKILL.md) (Fuse) |
| 6 | Add optional frameworks (SQL, Firebase, Web, AV, …) | [skip-frameworks](../skip-frameworks/SKILL.md) |
| 7 | Write tests (Swift Testing or XCTest) | [skip-testing](../../../skip-testing-deployment/skills/skip-testing/SKILL.md) |
| 8 | Drive the running app with `skip app launch` + Maestro | [skip-ui-automation](../../../skip-testing-deployment/skills/skip-ui-automation/SKILL.md) |
| 9 | Localise the catalog + iOS Info.plist + fastlane metadata | [skip-localization](../skip-localization/SKILL.md) |
| 10 | Export, sign, distribute to the App Store and Google Play | [skip-deployment](../../../skip-testing-deployment/skills/skip-deployment/SKILL.md) |

## Phase 1 — Lite or Fuse?

Default to **Lite**. It has the broadest skill / sample / agent coverage in this marketplace, transpiles cleanly for most Swift code, and produces a ~10–15 MB APK.

Pick **Fuse** if you need Swift language features that Lite doesn't transpile (variadic generics, macros, reflection, key-paths-as-values), if you're consuming Swift packages not written for Skip, or if you want direct C interop without `SkipFFI`. Fuse costs ~60 MB of Swift runtime in the APK.

→ [skip-fuse](../skip-fuse/SKILL.md) for the full trade-off table and mode-detection commands.

## Phase 2 — Scaffold

```bash
# Lite, two-module (UI + Model)
skip init --transpiled-app --appid=com.example.myapp my-app MyApp MyAppModel --no-build

# Fuse, single-module
skip init --native-app --appid=com.example.myapp my-app MyApp --no-build
```

Use `--no-build` so you can look at the generated tree before triggering a 3-minute build. Boot the iOS Simulator and Android emulator **before** running `skip app launch` — the build tools don't start them for you.

→ [skip-project-creation](../skip-project-creation/SKILL.md) for every flag, the generated layout, and how to add optional frameworks to `Package.swift` after init.

## Phase 3 — Write the SwiftUI

Use standard SwiftUI. SkipUI translates it to Compose for you. Two non-obvious constraints to internalise before you write much:

1. **`.accessibilityIdentifier(...)` on every interactive view.** Button, Toggle, Picker, TextField, NavigationLink, every Menu item — give each one a stable dotted-lowercase ID (`button.close`, `field.url`, `toggle.haptics`). Maestro flows in Phase 8 will rely on these; identifiers added retroactively never survive translation.
2. **Some SwiftUI APIs aren't implemented yet.** Common ones to avoid: `.contentShape(Rectangle())`, `ToolbarItem(placement: .topBarTrailing)` (macOS-unavailable, breaks SwiftPM builds), `matchedGeometryEffect`, `UIViewRepresentable`. Use the substitutes documented in the skill.

→ [building-skip-ui](../building-skip-ui/SKILL.md) for the supported-component table, the Compose customisation surface, and Material 3 theming.

## Phase 4 — Icons

**Never use `Image(systemName:)` or `Label(_, systemImage:)` in a Skip project.** SF Symbols are an Apple-only catalogue; most names render as a blank on Android, and even the ~50 names SkipUI maps internally produce a different-looking Material Icon rather than the SF Symbol you asked for.

For every icon: download the Material Symbol from `fonts.google.com/icons` in **Apple symbolset format**, drop it under `Sources/<Module>/Resources/Icons.xcassets/<name>.symbolset/`, and render with `Image("<name>", bundle: .module)`. One symbolset works identically on both platforms.

```bash
# Example — download three glyphs
mkdir -p Sources/MyApp/Resources/Icons.xcassets
cd Sources/MyApp/Resources/Icons.xcassets
for name in home settings search; do
    mkdir -p "${name}.symbolset"
    curl -sS -o "${name}.symbolset/${name}.svg" \
        "https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/${name}/default/${name}_symbol.svg"
done
```

→ [skip-icons](../skip-icons/SKILL.md) for the full workflow including the `Contents.json` templates, how to audit an existing codebase for stray `systemName:` calls, and how to verify a glyph rendered on both platforms.

## Phase 5 — Write Swift that transpiles

`swift build` does **not** validate Kotlin transpilation. `swift test` does. Run `swift test` after every model-level change.

The rules that bite hardest on Lite:

- Swift `Int` is 64-bit, Kotlin `Int` is 32-bit — intermediate arithmetic can silently overflow on Android.
- `if d == 0` between a Double and an Int doesn't compile on Kotlin. Use `if d == 0.0`.
- `Foundation.sqrt(2.0)` doesn't compile on Kotlin. Use unqualified `sqrt(2.0)`.
- `Array.remove(atOffsets: IndexSet)` is provided by SwiftUI's extension, not by Foundation. A model module that doesn't `import SwiftUI` can't call it.

→ [skip-lite-transpilation](../skip-lite-transpilation/SKILL.md) (Lite) for the full rule set. Fuse modules: see [skip-fuse](../skip-fuse/SKILL.md) instead — most Lite rules don't apply, but the `#if SKIP` blocks inside Fuse modules still do.

## Phase 6 — Optional frameworks

Frameworks (SkipSQL, SkipFirebase, SkipKeychain, SkipWeb, SkipAV, SkipMotion, SkipDevice, SkipFFI, etc.) are added to `Package.swift`. Don't use Xcode's "Add Package…" menu — it updates Xcode's tracking but not Skip's Gradle generation.

→ [skip-frameworks](../skip-frameworks/SKILL.md) for the per-framework `Package.swift` snippets, `skip.yml` config, and starter code.

## Phase 7 — Tests

Use Swift Testing for new tests; XCTest still works. The same suite runs natively on Swift and (via the auto-generated `XCSkipTests` harness) on the JVM under Robolectric — `swift test` exercises both.

```bash
swift test                                    # Swift + transpiled Kotlin
swift test --filter XCSkipTests               # only the transpiled side
swift test --filter MyModelTests.testAdd      # one Swift test
```

→ [skip-testing](../../../skip-testing-deployment/skills/skip-testing/SKILL.md) for Swift Testing syntax, parity-test patterns, Robolectric vs. instrumented tests, and how to read a Kotlin-compile failure in `XCSkipTests` output.

## Phase 8 — Drive the running app

`swift test` doesn't tap buttons, type into fields, or take screenshots. For that, pair `skip app launch` with Maestro:

```
edit SwiftUI → skip app launch → eyeball both platforms → maestro test .maestro/feature.yaml → fix → repeat
```

The `.accessibilityIdentifier(...)`s you put on every interactive view in Phase 3 pay off here — Maestro flows reference views by ID, so a flow survives translation and Compose-vs-SwiftUI rendering differences.

→ [skip-ui-automation](../../../skip-testing-deployment/skills/skip-ui-automation/SKILL.md) for the launch + Maestro inner loop, identifier conventions, and the platform-quirks reference.

## Phase 9 — Localise

User-facing strings created with `Text("…", bundle: .module, comment: "…")` are extracted into `Sources/<Module>/Resources/Localizable.xcstrings`. Add translations for each supported language; mark agent-produced translations `"state" : "needs_review"`.

iOS-specific permission prompts go in `Darwin/InfoPlist.xcstrings`. App Store / Google Play storefront metadata (title, subtitle, description, keywords, release notes, screenshots) goes under `Darwin/fastlane/metadata/<locale>/` and `Android/fastlane/metadata/android/<locale>/`, each file constrained to a byte budget (30 bytes for titles, 80 for Play short descriptions, 4000 for full descriptions).

Skip transpiles the in-app catalog to Android string resources at build time, so a single source produces translated UI on both platforms.

→ [skip-localization](../skip-localization/SKILL.md) for the catalog schema, translation state machine, the byte budgets, App Store / Play language-code mismatches, and the `xcrun xcstringstool print` + `skip verify` + `skip meta index` validation chain.

## Phase 10 — Ship

```bash
skip export -c release -d ./artifacts    # both platforms, release build
```

Sign the iOS archive and Android AAB, run `skip verify` and `skip meta index` one last time, then upload.

→ [skip-deployment](../../../skip-testing-deployment/skills/skip-deployment/SKILL.md) for the `skip export` flags, code-signing setup, keystore management, ProGuard rules, and the GitHub Actions CI workflow.

## A complete working example: `skipapp-todo`

The `skipapp-todo` sample app (in the project that ships these skills, at `https://github.com/skiptools/skills`) is a small TODO app built end-to-end with this exact phase ordering. Its README is a build log: every command, prompt, and decision is recorded with the surprises called out inline. Use it as a reference when you want to see "what does Phase X actually look like in practice."

It demonstrates:

- Scaffolding a Skip Lite app with `skip init --transpiled-app`.
- Replacing the template UI with a single-screen TODO list.
- Six Material Symbols glyphs (`add`, `check_circle`, `radio_button_unchecked`, `delete`, `delete_sweep`, `task_alt`) wired through `Image("name", bundle: .module)`.
- Seven Maestro flows that pass identically on iOS and Android.
- French and Simplified Chinese translations (16 catalog keys, fastlane metadata for both stores with 3 phone screenshots per language).
- All validation passing: `swift test`, `maestro test`, `skip verify`, `skip meta index`.

## When to use which skill — quick lookup

If you're stuck on a specific thing rather than walking through phases:

| Symptom | Likely skill |
|---|---|
| "How do I make a new Skip project?" | [skip-project-creation](../skip-project-creation/SKILL.md) |
| "Should I use Lite or Fuse?" | [skip-fuse](../skip-fuse/SKILL.md) (trade-off table at the top) |
| "My icon is blank on Android" | [skip-icons](../skip-icons/SKILL.md) |
| "My Kotlin compile is failing in `XCSkipTests`" | [skip-lite-transpilation](../skip-lite-transpilation/SKILL.md) (Lite) or [skip-fuse](../skip-fuse/SKILL.md) (Fuse) |
| "I need to add Firebase / SQLite / a WebView" | [skip-frameworks](../skip-frameworks/SKILL.md) |
| "Where does this SwiftUI component go in the supported-list?" | [building-skip-ui](../building-skip-ui/SKILL.md) |
| "How do I write a test that runs on both platforms?" | [skip-testing](../../../skip-testing-deployment/skills/skip-testing/SKILL.md) |
| "How do I write a Maestro flow that survives translation?" | [skip-ui-automation](../../../skip-testing-deployment/skills/skip-ui-automation/SKILL.md) |
| "What's the byte budget for the Play short description?" | [skip-localization](../skip-localization/SKILL.md) |
| "How do I sign the Android build for release?" | [skip-deployment](../../../skip-testing-deployment/skills/skip-deployment/SKILL.md) |
