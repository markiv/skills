---
name: skip-ui-automation
description: Launching Skip apps on the iOS Simulator and Android emulator simultaneously with `skip app launch`, and driving them with Maestro UI tests. Covers Maestro installation, the canonical edit→launch→eyeball→test inner loop, writing cross-platform flow YAMLs that work on both SwiftUI on iOS and Jetpack Compose on Android, the importance of `.accessibilityIdentifier()` on every interactive view, and the locale-aware screenshot workflow used for fastlane metadata. The platform-by-platform quirk catalog, the iOS/Android locale-switching commands, and the stale-build recoveries are in references/ — link to them rather than re-explaining. Use this skill whenever working on UI tests, manual smoke testing, designing views that need to be drivable by an automated agent, or generating localised App Store / Play screenshots.
---

# Skip UI Automation

Skip projects can be launched on the iOS Simulator and an Android emulator side-by-side from a single command, and the same flow can be exercised on both platforms with [Maestro](https://maestro.dev). This skill covers the inner loop, the `.accessibilityIdentifier()` convention that holds flows together, and the project layout for `.maestro/` flows.

For the platform-specific gotchas, locale-switching commands, and build-state recoveries that come up often enough to deserve their own files, consult:

- [`references/platform-quirks.md`](references/platform-quirks.md) — iOS vs. Android `assertVisible:` differences, `clearState: true` blank-screen issue, `hideKeyboard` failing on iOS, `inputText` ASCII-only on Android, Compose `OutlinedTextField` ID propagation, Toggle thumb hit target, WebView scroll/text opacity, settings sheet dismissal.
- [`references/locale-switching.md`](references/locale-switching.md) — iOS `simctl spawn defaults write` and Android `adb cmd locale set-app-locales` recipes, the `MAESTRO_OUT_SUFFIX` env-substitution pattern for per-language screenshot paths, the typical end-to-end localised screenshot workflow.
- [`references/build-state-recovery.md`](references/build-state-recovery.md) — when to `rm -rf .build`, the stale `Bundle.module` recovery, stale Gradle/Compose-compiler cache after a Skip upgrade, stale Xcode DerivedData, stale Android install, and the nuclear "wipe everything" reset.

## The inner loop

`swift test` and `swift build` validate that the code compiles and the test suites pass. Neither tells you that the *running* app behaves correctly on both platforms. `skip app launch` puts the app in front of you on both simulators in one step, and Maestro replays the same gestures against both — so when you change a SwiftUI view you find out immediately whether it still renders, responds, and looks right under Compose.

This is the canonical inner loop for cross-platform UI work:

```
edit SwiftUI → skip app launch → eyeball both platforms → maestro test .maestro/feature.yaml → fix → repeat
```

### Pre-flight

`skip app launch` does **not** boot the iOS Simulator or start an Android emulator for you. Boot them first:

```bash
xcrun simctl boot <UDID>; open -a Simulator         # iOS
emulator -avd Pixel_6_API_34                         # Android (blocks; another terminal)
# Or, on CI:
skip android emulator launch --background --headless

# Confirm
xcrun simctl list devices booted
adb devices
```

With both running:

```bash
skip app launch                                      # both platforms
skip app launch --verbose                            # full xcodebuild / gradle output
skip app launch --ios                                # iOS only (fastest)
skip app launch --android                            # Android only
skip app launch --configuration release              # release build
```

When you have **both** a USB-connected Android phone and a running emulator, scope the install to the emulator with the standard env var — `skip app launch` has no `--android-serial` flag:

```bash
ANDROID_SERIAL=emulator-5554 skip app launch
```

For iterating on a peer Skip framework checkout, point at the local tree:

```bash
SKIP_DEPENDENCY_ROOT=/path/to/skiptools skip app launch
```

(The convention is documented at the bottom of every Skip project's `Package.swift` — see the template comment block.)

### When to reach for `skip app launch`

- After any change to a SwiftUI view — `swift build` will never catch a Kotlin compile failure or a Compose rendering bug.
- Before running Maestro tests — the test runs against the *installed* build, so a stale install will silently test old behaviour.
- When debugging platform divergence — running both side by side is the cheapest way to see "this looks fine on iOS but breaks on Android."

## Maestro

[Maestro](https://maestro.dev) is a YAML-driven UI test runner that speaks both XCUITest (iOS) and UIAutomator (Android) underneath. A single flow file can target either platform — only the `appId` differs.

### Install

```bash
brew tap mobile-dev-inc/tap
brew install maestro
maestro --version
```

Maestro is a Java app; the Homebrew formula installs a compatible JDK if one isn't present. On Apple Silicon the iOS driver downloads on first use — allow it through Gatekeeper if prompted.

### Project layout

Put Maestro flows in a `.maestro/` directory next to `Package.swift`:

```
my-app/
  Package.swift
  Sources/…
  .maestro/
    config.yaml                   # optional shared config
    helpers/
      seed-empty.yaml             # reusable sub-flows
    screenshots/                  # output of takeScreenshot:
    00-smoke.yaml                 # baseline cross-platform smoke
    10-add-task.yaml              # one flow per feature
    20-walkthrough-en.yaml        # per-language walkthrough flows
    21-walkthrough-fr.yaml
```

Most cross-platform divergence is hidden by the helpers — most flow files differ only in their `appId` line.

### A minimal cross-platform flow

```yaml
# .maestro/00-smoke.yaml
appId: dev.skip.todo
---
- launchApp                       # no clearState — see references/platform-quirks.md
- assertVisible: "My Tasks"
- assertVisible:
    id: "field.newTask"
- assertVisible:
    id: "button.add"
- takeScreenshot: .maestro/screenshots/00-smoke
```

### Running flows

```bash
# Find your device IDs
xcrun simctl list devices booted     # iOS Sim UDIDs
adb devices                          # Android emulator/device serials

# Single device
maestro --device <iOS-UDID>      test .maestro/00-smoke.yaml
maestro --device emulator-5554   test .maestro/00-smoke.yaml

# In parallel
maestro --device <iOS-UDID>      test .maestro/00-smoke.yaml &
maestro --device emulator-5554   test .maestro/00-smoke.yaml &
wait
```

### Reading failures

Each command line prints `COMPLETED` or `FAILED`. On `FAILED`, Maestro writes debug artefacts to `~/.maestro/tests/<timestamp>/`:

- `screenshot-❌-*.png` — the screen at the moment of failure. Open it first; an unexpected dialog, a slow page load, an off-screen element is usually visible.
- `commands-(*.yaml).json` — the full hierarchy Maestro saw, with `text` / `accessibilityText` / `accessibilityIdentifier` on each node.
- `maestro.log` — the runner's own log.

### Screenshots

`takeScreenshot:` resolves relative to the current working directory. When `maestro test` is run from the project root, prefix paths with `.maestro/` so they land inside the project tree:

```yaml
- takeScreenshot: .maestro/screenshots/feature-after-tap
```

A bare `screenshots/foo` would resolve to `<cwd>/screenshots/` — wherever you ran Maestro from — which surprises users who expect "test-relative" paths.

For per-language screenshot paths driven by env-substitution (`${MAESTRO_OUT_SUFFIX}`), see [`references/locale-switching.md`](references/locale-switching.md).

## `.accessibilityIdentifier()` — load-bearing

**Every interactive view that a test or an agent might want to drive must have an `.accessibilityIdentifier(...)`.** This is not optional politeness; it's the difference between a test suite that survives refactors and one that re-breaks every time a label is translated or a layout shifts.

### Why identifiers, not labels

Maestro can locate elements by visible text (`tapOn: "Search Tabs"`) or by ID (`tapOn: { id: "field.tabSearch" }`). Text-based matching is fragile:

- The string changes on translation. A flow that taps `"Done"` in English breaks in German (`Fertig`).
- The string appears in multiple places. A `tapOn: "Cancel"` may hit the wrong button.
- The string is rendered inside a `WebView`. WKWebView content is not in the iOS accessibility tree — Maestro can match a *native* button labelled "Open", but not text *inside a webpage*.
- Compose `OutlinedTextField` and other primitives don't always surface SwiftUI labels.

IDs are stable across translation, refactor, and SwiftUI-vs-Compose rendering differences. They cost one line of code per view.

### How to attach IDs

```swift
Button(action: closeAction) {
    Image("xmark", bundle: .module)
}
.accessibilityIdentifier("button.close")
.accessibilityLabel(Text("Close", bundle: .module, comment: "accessibility label for the close button"))

TextField("URL", text: $url)
    .accessibilityIdentifier("field.url")

Toggle("Block Ads", isOn: $blockAds)
    .accessibilityIdentifier("toggle.blockAds")

NavigationLink {
    DetailView()
} label: {
    Label {
        Text("Settings", bundle: .module, comment: "settings navigation link label")
    } icon: {
        Image("gearshape", bundle: .module)
    }
}
.accessibilityIdentifier("link.settings")
```

For icon-only buttons, **always pair `.accessibilityIdentifier` with `.accessibilityLabel`** — VoiceOver/TalkBack users can't tap a button whose only signal is a glyph.

### Identifier naming convention

Use dotted lowercase strings that encode the element kind:

| Prefix      | Element                                |
|-------------|----------------------------------------|
| `button.`   | Buttons (`button.close`, `button.tabs`)|
| `field.`    | TextFields, TextEditors                |
| `toggle.`   | Toggles                                |
| `picker.`   | Pickers                                |
| `link.`     | NavigationLinks, hyperlinks            |
| `menu.`     | Menu items                             |
| `label.`    | Static labels worth asserting          |

Dotted names are easy to grep for and survive refactors. `button.menu` and `button.tabs.new` are far more searchable than `firstMenuButton` or `tabsAddButton`.

### What to identify

- Every `Button`, `Toggle`, `Picker`, `TextField`, `Slider`, and `NavigationLink` a user can interact with.
- Each item inside a `Menu` (the menu's label and every `Button` inside it).
- Any view whose visibility you'd want to assert on (an empty-state label, a loading spinner).

You don't need to identify every `Text`, `Spacer`, `HStack`, or layout container.

For Compose-specific ID-propagation issues (notably `OutlinedTextField` and iOS Menu items), see [`references/platform-quirks.md`](references/platform-quirks.md).

## Reusable helpers

Put any tap sequence used by more than one flow into `.maestro/helpers/` and invoke with `runFlow`:

```yaml
# .maestro/helpers/seed-empty.yaml
appId: dev.skip.todo
---
- runFlow:
    when:
      notVisible:
        id: "label.emptyState"
    commands:
      - tapOn: { id: "button.menu" }
      - tapOn: "Clear All"
      - tapOn: "Delete All"
```

```yaml
# .maestro/10-add-task.yaml
appId: dev.skip.todo
---
- launchApp
- runFlow: { file: helpers/seed-empty.yaml }
- tapOn: { id: "field.newTask" }
- inputText: "Buy milk"
- tapOn: { id: "button.add" }
- assertVisible: "Buy milk"
```

The `runFlow: when: visible:` (or `notVisible:`) form lets the helper be a no-op when its precondition isn't met, so it can be used both on first launch and after the app has accumulated state. This keeps flows idempotent and re-runnable.

## Putting it together — typical iteration

1. Edit the SwiftUI view, attach `.accessibilityIdentifier("button.foo")` to anything new that's interactive.
2. `skip app launch` — build and install on both Sim and emulator.
3. Eyeball the change on both screens.
4. Write a flow file under `.maestro/feature.yaml`.
5. `maestro --device <iOS-UDID> test .maestro/feature.yaml` and `maestro --device emulator-5554 test .maestro/feature.yaml`, in parallel.
6. On `FAILED`, open the debug screenshot under `~/.maestro/tests/<timestamp>/` first.
7. Capture release-quality screenshots with `takeScreenshot: .maestro/screenshots/<name>` and keep them in the repo alongside the flow.

## Cross-references

- For logic-level (non-UI) tests that run on both platforms via `XCSkipTests`, see [skip-testing](../skip-testing/SKILL.md).
- For the icon side of `.accessibilityIdentifier` — why icon-bearing buttons need both an `.accessibilityLabel(...)` and a `.symbolset`-backed `Image(...)` — see [skip-icons](../../../skip-app-design/skills/skip-icons/SKILL.md).
- For localization of the `Text(_, bundle: .module, comment:)` strings the flow's `assertVisible:` lines match against, see [skip-localization](../../../skip-app-design/skills/skip-localization/SKILL.md).
- For end-to-end how `skip app launch` and `maestro` fit into the build process, see [skip-app-walkthrough](../../../skip-app-design/skills/skip-app-walkthrough/SKILL.md).
