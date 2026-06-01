# Maestro platform quirks for Skip apps

Cross-platform Maestro flows for Skip apps run against SwiftUI on iOS and Jetpack Compose on Android. The two engines disagree often enough that this file is a reference for the recurring failures.

Skip the entries that don't apply to your flow.

## `assertVisible:` text matching — iOS vs Android

The same text assertion can pass on iOS and fail on Android (or vice versa). The two platforms use different matching semantics:

- **iOS** — Maestro reads the `accessibilityText` of each element and looks for the pattern as a **substring**. `assertVisible: "Skipper"` matches an element whose accessibility text is `"Hello Skipper!"`.
- **Android** — Maestro reads the Compose `text` and does an **anchored regex** match. `assertVisible: "Skipper"` does **not** match `"Hello Skipper!"` — the entire text must match the regex.

So a flow that asserts on a short substring like `"Skipper"` works on iOS but fails on Android. A flow that asserts on the full literal `"Hello Skipper!"` may also fail on Android because `!` is a regex meta-char in some Maestro versions and the trailing punctuation has to be in the pattern.

The reliable cross-platform form is **regex with `.*` on both sides**:

```yaml
- assertVisible: ".*Skipper.*"
```

`.*Skipper.*` is a substring-like match on iOS (the literal-substring matcher accepts a regex that contains `Skipper`) and a proper anchored regex on Android (the `.*` anchors absorb the surrounding "Hello " and "!").

When the visible text is a fixed tab label or button label with no surrounding text (e.g. "Settings", "Home"), bare `assertVisible: "Settings"` works on both. The `.*X.*` form is only needed when the element contains more than the substring.

## `launchApp: { clearState: true }` can render blank on Android

```yaml
- launchApp:
    clearState: true
```

…can produce a **blank Compose screen** on Android (the activity launches but no UI renders), while iOS is unaffected. The behaviour is not universal: `skipapp-calculatrix-maestro` uses `clearState: true` across all five of its published flows without issue. Apps that have hit it in practice were ones with `@AppStorage`-backed state or Codable persistence whose first launch after a state wipe re-reads defaults / file paths that no longer exist.

If you can't reproduce the blank screen, `clearState: true` is fine. If you can, drop it and do the reset in-app instead. The `runFlow: when:` form keeps the flow idempotent without needing a real state wipe:

```yaml
- launchApp
- runFlow:
    when:
      notVisible:
        id: "label.emptyState"
    commands:
      - tapOn:
          id: "button.menu"
      - tapOn: "Clear All"
      - tapOn: "Delete All"
- assertVisible:
    id: "label.emptyState"
```

This pattern is also faster than a re-install and matches what a real user would do.

## `hideKeyboard` doesn't work on iOS

```yaml
- hideKeyboard
```

…on iOS returns *"Couldn't hide the keyboard. This can happen if the app uses a custom input or doesn't expose a standard dismiss action."* SwiftUI's keyboard isn't an `inputView` Maestro can find a dismiss for.

The documented workaround is to **tap a non-interactive element** — typically the navigation title, an empty area of the background, or a static label. Pick one that has a stable `accessibilityIdentifier` or whose text you control:

```yaml
- tapOn: "My Tasks"          # navigation title — dismisses the keyboard on iOS
```

On Android, `hideKeyboard` works as documented. The cross-platform-safe form is to dismiss with the title-tap in both places — it's a no-op on Android in most layouts.

## Maestro `inputText` only accepts ASCII on Android

```yaml
- inputText: "Appeler le médecin"
```

…fails on Android with: *"Unicode character input is not supported: Appeler le médecin. Please use ASCII characters."* The same line works on iOS (the simulator's keyboard handles UTF-8 input via the pasteboard).

Tracking issue: https://github.com/mobile-dev-inc/maestro/issues/146

Workarounds:

- **Pick an ASCII seed** for any text the test types. The UI *chrome* (titles, buttons, placeholders) still renders in the target language because it comes from the catalog; only typed values are limited. `"Appeler le docteur"` instead of `"Appeler le médecin"` is enough for a localised-screenshot flow.
- **Paste instead of type**, on a per-device basis: `adb -s <serial> shell input text "Appeler le médecin"` does support UTF-8 on Android, but is outside Maestro's flow grammar. Useful for one-off setup, not for portable flows.
- **Limit non-ASCII text to existing strings**, e.g. assert that a hard-coded localised string is visible — `assertVisible: "Mes tâches"` works on both platforms because the matcher accepts UTF-8; only `inputText` is restricted.

## Compose accessibility-identifier propagation

Skip transpiles `.accessibilityIdentifier(…)` to Compose's `Modifier.testTag(…)`. Most primitives propagate the tag through correctly. Two known gotchas:

- **Compose `OutlinedTextField`** historically does not surface the SwiftUI accessibility identifier on the inner field. On Android, fall back to the placeholder text: `tapOn: "Search Tabs"`. Document it in the test if you have to.
- **Buttons whose label is just a Text** sometimes drop the ID on iOS Menu items. If `tapOn: { id: "menu.foo" }` is flaky, also accept `tapOn: "Foo"` as a fallback.

## iOS — URL bar / TextField focus

SwiftUI's `@FocusState` doesn't always flush by the time `UITextField.textDidBeginEditingNotification` fires, so a notification observer guarded on `isURLBarFocused` will silently drop the first tap. Filter by `accessibilityIdentifier` instead of `@FocusState` if you need to select-all-on-focus.

## Android — URL / text-field tap zone

Composables sometimes report their hit zone as the whole row. Tapping the centre of a `field.url` may register as a tap on the parent. The reliable workaround is `tapOn: { id: "field.url", point: "5%,95%" }` — bottom-left corner of the field.

## SwiftUI Toggle hit target

The switch *thumb* is the actual hit target — tapping the row label does not flip the toggle. Use `tapOn: { id: "toggle.foo", point: "90%,50%" }` to land on the thumb.

## Settings sheet dismissal

- **iOS**: AppFairSettings (and other library sheets) often don't render a `Done` button that Maestro can find. Dismiss with `launchApp: { clearState: false }` after writing to UserDefaults — the next launch picks up the change.
- **Android**: `hideKeyboard` followed by a single `back`. Two back presses kill the app entirely.

## `extendedWaitUntil` vs. `assertVisible`

Page loads (especially over real networks) take longer than Maestro's default `assertVisible` timeout. For anything that waits on a navigation, use `extendedWaitUntil: { visible: "domain.com", timeout: 30000 }`.

## WebView content is opaque to Maestro on iOS

Maestro can read the URL bar's domain label (a native SwiftUI `Text`), but it cannot read the heading inside the loaded HTML on iOS — that text lives in WKWebView, not the accessibility tree. Assert against the URL bar (`visible: "example.com"`) or against an in-page text that's mirrored to a native overlay; do not rely on page body text.

## iOS Maestro can't scroll a WKWebView

`swipe: { direction: UP }` will scroll a SwiftUI `ScrollView` or a Compose `LazyColumn`, but it will not synthesize a real scroll event inside a WKWebView on iOS Simulator. Android Compose builds with `setOnScrollChangeListener` do receive synthesized scrolls. If you need to verify scroll-driven behaviour, do it on Android and rely on visual review for iOS.
