# Skip Skills

Agent skills for building cross-platform iOS and Android apps with [Skip](https://skip.dev). Each skill is a `SKILL.md` file with a frontmatter description. The agent loads a skill when its description matches what you're asking about.

- [Skills](#skills)
- [Installation](#installation)
- [Example: a TODO app in 8 prompts](#example-a-todo-app-in-8-prompts)
- [Reference build: `skipapp-todo`](#reference-build-skipapp-todo)
- [Prompts that trigger each skill](#prompts-that-trigger-each-skill)

## Skills

### `skip-app-design`

| Skill | Scope |
|---|---|
| [skip-app-walkthrough](plugins/skip-app-design/skills/skip-app-walkthrough/SKILL.md) | Orchestrator. Sequences the other skills. Start here on a new project. |
| [skip-project-creation](plugins/skip-app-design/skills/skip-project-creation/SKILL.md) | Scaffolding a new app with `skip init`. |
| [building-skip-ui](plugins/skip-app-design/skills/building-skip-ui/SKILL.md) | Writing SwiftUI that transpiles to Jetpack Compose. |
| [skip-lite-transpilation](plugins/skip-app-design/skills/skip-lite-transpilation/SKILL.md) | Swift-to-Kotlin transpilation rules for Lite mode. |
| [skip-fuse](plugins/skip-app-design/skills/skip-fuse/SKILL.md) | Fuse mode: native Swift on Android via the Swift Android SDK. |
| [skip-frameworks](plugins/skip-app-design/skills/skip-frameworks/SKILL.md) | Adding optional Skip frameworks (SQL, Firebase, Keychain, Web, AV, etc.). |
| [skip-icons](plugins/skip-app-design/skills/skip-icons/SKILL.md) | The hard rule against `Image(systemName:)`, and the Material Symbols `.symbolset` workflow that replaces it. |
| [skip-localization](plugins/skip-app-design/skills/skip-localization/SKILL.md) | `Localizable.xcstrings`, `InfoPlist.xcstrings`, and fastlane storefront metadata. |

### `skip-testing-deployment`

| Skill | Scope |
|---|---|
| [skip-testing](plugins/skip-testing-deployment/skills/skip-testing/SKILL.md) | XCTest and Swift Testing with parity testing across iOS and Android. |
| [skip-ui-automation](plugins/skip-testing-deployment/skills/skip-ui-automation/SKILL.md) | `skip app launch` and Maestro flows that target both platforms. |
| [skip-deployment](plugins/skip-testing-deployment/skills/skip-deployment/SKILL.md) | Release builds, signing, store submission, CI. |

## Installation

[Claude Code](https://claude.com/claude-code) is the recommended agent. Cursor, Gemini, and Codex work as well; the skills are plain markdown and don't depend on agent-specific features.

### Claude Code

```
/plugin marketplace add skiptools/skills
/plugin install skip-app-design
/plugin install skip-testing-deployment
```

### Cursor

1. Open Cursor Settings (Cmd+Shift+J / Ctrl+Shift+J).
2. `Rules & Command` > `Project Rules` > `Add Rule` > `Remote Rule (GitHub)`.
3. Enter `https://github.com/skiptools/skills.git`.

Skills load automatically based on the topic you ask about. There is no slash-command entry to invoke them.

### Other agents

Point the agent at `AGENTS.md` or at any `SKILL.md` under `plugins/`.

## Example: a TODO app in 8 prompts

These are the prompts used to build [`skipapp-todo`](#reference-build-skipapp-todo). Run them in order. Each one builds on the state the previous one left.

### 1. Scaffold

> **Prompt:** Create a new Skip Lite TODO app at `/tmp/skipapp-todo` with bundle id `dev.skip.todo`. Use a two-module layout: `TodoApp` for the UI, `TodoAppModel` for the data. Pass `--no-build` so we can inspect the generated tree first.

Loads `skip-app-walkthrough` and `skip-project-creation`. The agent runs `skip init --transpiled-app --appid=dev.skip.todo skipapp-todo TodoApp TodoAppModel --no-build`. The output is a Swift package with `Package.swift`, `Sources/TodoApp/`, `Sources/TodoAppModel/`, `Darwin/`, `Android/`, and a populated template `Localizable.xcstrings`.

### 2. Confirm the template runs

> **Prompt:** Boot an iPhone simulator and an Android emulator. Run `skip app launch`. Take a screenshot from each platform.

Loads `skip-ui-automation`. The first build takes a few minutes because Skip resolves SkipUI, SkipFoundation, and SkipModel from source. Subsequent builds are incremental. The template Welcome view should render on both screens.

### 3. Replace the template UI with a real TODO list

> **Prompt:** Replace the tab-based template with a single-screen `NavigationStack`: a text field at the top to add tasks, a list of tasks below with a tappable circle that toggles complete, swipe-to-delete, an overflow menu with "Clear Completed" and "Clear All", and an empty state. For every icon, use a Material Symbol downloaded from `fonts.google.com/icons` in Apple symbolset format and reference it via `Image("name", bundle: .module)`. Do not use `Image(systemName:)` or `Label(_, systemImage:)` anywhere — those rely on Apple's SF Symbols catalogue and render blank on Android. Every interactive view needs an `.accessibilityIdentifier(...)`.

Loads `building-skip-ui`, `skip-icons`, and `skip-lite-transpilation`. The agent rewrites `ContentView.swift` and `ViewModel.swift`. It downloads `add`, `check_circle`, `radio_button_unchecked`, `delete`, `delete_sweep`, and `task_alt` from `fonts.gstatic.com` in Apple `.symbolset` format and writes each under `Sources/TodoApp/Resources/Icons.xcassets/<name>.symbolset/` with the right `Contents.json`.

The agent also navigates a few Skip-specific constraints. It uses `.primaryAction` instead of `.topBarTrailing` (the latter is unavailable on macOS, which SwiftPM also builds for). It drops `.contentShape(Rectangle())` (not implemented in Skip yet). It exposes `func remove(at: [Int])` on the model instead of `remove(atOffsets:)` (the IndexSet overload is a SwiftUI extension and the model module doesn't import SwiftUI).

Then `swift test` and `skip app launch` to confirm.

### 4. Maestro flows

> **Prompt:** Write Maestro flows for the TODO app: a smoke test, add two tasks, toggle one complete, clear all. Each flow must pass identically on iOS and Android. Save screenshots under `.maestro/screenshots/`. Run all four on both the simulator and the emulator.

Loads `skip-ui-automation`. The flows go in `.maestro/00-smoke.yaml`, `10-add-task.yaml`, `11-toggle-complete.yaml`, `12-clear-all.yaml`. They tap by ID rather than visible text. They use `runFlow: when: notVisible:` to make the seed step idempotent. They avoid `hideKeyboard` (no implementation on iOS) and non-ASCII `inputText` (Maestro on Android rejects it). They reset state in-app rather than via `clearState: true` (which can render a blank Compose screen on Android for some apps). Each flow runs against both `--device <iOS-UDID>` and `--device emulator-5554`.

### 5. Translate the catalog

> **Prompt:** Translate the in-app strings into French and Simplified Chinese. Mark every new translation `state: needs_review`. Wipe the template's stale translations from the catalog; the new UI doesn't reference those keys. The model module has no user-facing strings, so set its catalog to the empty form.

Loads `skip-localization`. The agent rewrites the UI module's catalog with one entry per `Text(_, bundle: .module, comment: …)` in the source. The model module's catalog becomes `{ "sourceLanguage": "en", "strings": {}, "version": "1.0" }`. The agent runs `xcrun xcstringstool print` on both files to confirm they parse.

### 6. Verify the translations in the running app

> **Prompt:** Switch the iOS Simulator to French and the Android emulator's per-app locale to `fr-FR`. Relaunch the app on both and take a screenshot. Repeat for Simplified Chinese (`zh-Hans` on iOS, `zh-Hans-CN` on Android). Switch both back to English when you're done.

Loads the locale-switching reference under `skip-ui-automation`. iOS goes through `xcrun simctl spawn $UDID defaults write "Apple Global Domain" AppleLanguages "(fr)"` followed by `xcrun simctl spawn $UDID launchctl stop com.apple.SpringBoard` to pick up the new defaults. Android uses `adb shell cmd locale set-app-locales dev.skip.todo --locales fr-FR`. A single Maestro walkthrough flow with an `${MAESTRO_OUT_SUFFIX}` env substitution produces per-language per-platform screenshots in one directory.

### 7. Fastlane metadata for both stores

> **Prompt:** Fill in fastlane metadata for the App Store and Google Play. Translate title, subtitle / short_description, description / full_description, keywords, and release notes for all three languages. Use the correct per-store locale codes; the App Store wants `zh-Hans`, Google Play wants `zh-CN`. Add three phone screenshots per language per store, using the walkthrough screenshots. Validate with `skip verify` and `skip meta index`.

Loads `skip-localization` ([`fastlane-metadata.md`](plugins/skip-app-design/skills/skip-localization/references/fastlane-metadata.md) and [`language-codes.md`](plugins/skip-app-design/skills/skip-localization/references/language-codes.md)) and `skip-deployment`. The agent populates `Darwin/fastlane/metadata/{en-US,fr-FR,zh-Hans}/` and `Android/fastlane/metadata/android/{en-US,fr-FR,zh-CN}/` with text files within the byte budgets (30 for titles, 80 for Play short descriptions, 4000 for descriptions). It copies three walkthrough screenshots per language into `Darwin/fastlane/screenshots/<locale>/` and `Android/fastlane/metadata/android/<locale>/images/phoneScreenshots/`. It runs `wc -c` on every `.txt` file and `skip verify && skip meta index`.

### 8. Regression pass

> **Prompt:** Re-run every Maestro flow on both platforms. Re-run `swift test` to confirm the Kotlin transpilation still passes. Confirm `skip verify` and `skip meta index` exit clean.

Loads `skip-testing` and `skip-ui-automation`. All four flows pass on both platforms, all Swift and JUnit tests pass, both metadata validators exit zero. The project is now ready for `skip export -c release`.

### Result

At the end of step 8 you have:

- 4 Maestro flows passing on iOS and Android.
- 16 catalog keys translated into English, French, and Simplified Chinese.
- App Store and Google Play metadata in all three languages, within byte budgets.
- 3 phone screenshots per language per store.
- `swift test`, `skip verify`, and `skip meta index` exiting clean.

## Reference build: `skipapp-todo`

The [`skipapp-todo`](https://github.com/skiptools/skipapp-todo) sample is the build that produced the prompt sequence above. Its README is a development log: every command, file write, and failure recorded in order.

The gotchas surfaced during that build are now in the relevant skill `references/` files: the iOS vs. Android `assertVisible:` regex difference, `clearState: true` on Android, `hideKeyboard` on iOS, non-ASCII `inputText` on Android, per-app locale switching, and the build-state recoveries.

## Prompts that trigger each skill

Single-question prompts route to a skill by topic.

| Prompt | Skill |
|---|---|
| Walk me through building a new Skip app from scratch | `skip-app-walkthrough` |
| Should I use Skip Lite or Skip Fuse for this project? | `skip-fuse` |
| Create a new Skip Lite app with tab-based navigation | `skip-project-creation` |
| Add Firebase authentication to my Skip project | `skip-frameworks` |
| How do I write tests that run on both iOS and Android? | `skip-testing` |
| Launch the app on the iOS simulator and Android emulator together | `skip-ui-automation` |
| Write a Maestro flow that opens the settings sheet | `skip-ui-automation` |
| My Maestro tests break after I translate a string. Why? | `skip-ui-automation` |
| My `assertVisible:` works on iOS but fails on Android. Why? | `skip-ui-automation` ([`platform-quirks.md`](plugins/skip-testing-deployment/skills/skip-ui-automation/references/platform-quirks.md)) |
| Switch the simulator and emulator to French for screenshot generation | `skip-ui-automation` ([`locale-switching.md`](plugins/skip-testing-deployment/skills/skip-ui-automation/references/locale-switching.md)) |
| Translate the in-app strings into German, Japanese, and Brazilian Portuguese | `skip-localization` |
| What's the App Store locale code for Simplified Chinese, and the Play Store one? | `skip-localization` ([`language-codes.md`](plugins/skip-app-design/skills/skip-localization/references/language-codes.md)) |
| Update the App Store and Google Play metadata for the next release | `skip-localization` and `skip-deployment` |
| Validate `Localizable.xcstrings` and make sure no translations are missing | `skip-localization` |
| My `Label` icon shows up on iOS but not Android | `skip-icons` |
| Add a bookmark icon to this button. Make sure it renders on both platforms | `skip-icons` |
| Download a Material Symbols icon and wire it into the app's xcassets | `skip-icons` ([`symbolset-anatomy.md`](plugins/skip-app-design/skills/skip-icons/references/symbolset-anatomy.md)) |
| Kotlin compile is failing in `XCSkipTests.testSkipModule`. How do I debug it? | `skip-testing` and `skip-lite-transpilation` |
| I'm getting `Unresolved reference 'Bundle'` after adding `bundle: .module` | `skip-ui-automation` ([`build-state-recovery.md`](plugins/skip-testing-deployment/skills/skip-ui-automation/references/build-state-recovery.md)) |
| How do I sign the Android build for release? | `skip-deployment` |

If routing falls back to general Skip knowledge, mention the specific artefact name in the prompt (`Localizable.xcstrings`, `.symbolset`, `skip app launch`, `maestro test`, `skip.yml`, `fastlane/metadata/`). That usually corrects it.

## License

Apache 2.0
