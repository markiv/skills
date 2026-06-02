---
name: skip-icons
description: Icons in Skip projects. The hard rule: never use `Image(systemName:)` or `Label(_, systemImage:)`. SF Symbols are an Apple-only catalogue with no Android equivalent; the names you pass to `systemName:` / `systemImage:` will not resolve on Android and most will render as a blank or generic-placeholder glyph. The canonical replacement is to download Material Symbols from `fonts.google.com/icons` in **Apple symbolset format**, drop the `.symbolset/` bundle under `Sources/<Module>/Resources/Icons.xcassets/`, and render with `Image("name", bundle: .module)` inside the localisation-correct `Label { Text(…, bundle: .module) } icon: { Image(…) }` form. The same SVG ships to both platforms and looks identical. Also covers app launcher icons via `skip icon` and how to verify in-app icons render on both platforms. The full `.symbolset` format spec (download URLs per variant, `Contents.json` schemas, verification checks) is in references/symbolset-anatomy.md. references/sf-symbol-map.md explains what SkipUI does with `systemName:` internally — it is **not** a "safe-to-use" list. Use this skill whenever adding, replacing, or auditing icons in a Skip app or library.
---

# Skip Icons

There is one rule for icons in Skip projects:

> **Never use `Image(systemName:)` or `Label(_, systemImage:)`. Always ship a `.symbolset` resource and render it with `Image("name", bundle: .module)`.**

This rule has no exceptions in production code. Every other section of this skill exists to support it.

## Why `systemName:` is an anti-pattern

SwiftUI's SF Symbols catalogue is an Apple-only registry of thousands of glyphs accessed by name. Android has no equivalent registry, and there is no automatic mapping from SF Symbol names to Android Material Icons. So when you write:

```swift
Image(systemName: "checklist")
Label("Tasks", systemImage: "checklist")
```

…the iOS build resolves "checklist" against Apple's SF Symbols catalogue and renders the glyph. The Android build has nothing to resolve "checklist" against and produces a **blank space** (or a generic placeholder) where the icon should be. The Swift code compiles cleanly on both platforms. The transpiled Kotlin compiles cleanly. Tests pass. The bug only shows up at runtime, on Android, when a human or a Maestro screenshot notices the missing glyph.

Almost every `Image(systemName: "...")` call you write will fail this way on Android. Examples that ship in real apps and silently break:

```swift
Label("Bookmark",  systemImage: "bookmark.fill")          // ❌ blank on Android
Label("Tasks",     systemImage: "checklist")              // ❌ blank on Android
Label("Web",       systemImage: "globe.americas")         // ❌ blank on Android
Label("Code",      systemImage: "chevron.left.forward.slash.chevron.right")  // ❌ blank on Android
Label("Reminders", systemImage: "list.bullet.clipboard")  // ❌ blank on Android
Label("Cloud",     systemImage: "icloud.fill")            // ❌ blank on Android
```

SkipUI does ship a small hardcoded compatibility map (around 50 names — `house`, `gearshape`, `magnifyingglass`, `plus`, `xmark`, `trash`, and a handful of others) that translates a few common SF Symbol names to Compose Material Icons at runtime. That map exists, but **you should still not use `systemName:` for those names**, for three reasons:

1. **The list is small and hard to remember.** Picking from the safe set requires you to know which 50 names happen to be hardcoded. Pick wrong (e.g. `checklist`, `bookmark.fill`, `globe.americas`) and the Android build breaks invisibly.
2. **The rendering isn't identical even when it works.** A hardcoded mapping replaces the Apple glyph with a Material Icon that looks similar but isn't pixel-identical. Tinting, sizing, and weight may differ.
3. **The map is not a stable API.** SkipUI may add or remove entries between releases. A code path that worked yesterday may not work tomorrow.

The solution is to never reach for `systemName:` in the first place. Use a `.symbolset` resource for every icon. The same SVG ships to both platforms and renders identically on each.

> The internal SkipUI behaviour for `systemName:` (the ~50-name hardcoded map) is documented in [`references/sf-symbol-map.md`](references/sf-symbol-map.md) for completeness. **That file is descriptive, not prescriptive.** It tells you what SkipUI does internally with a `systemName:` call. It is not a list of names you should be using.

## The replacement — Material Symbols in Apple `.symbolset` format

Apple's `.symbolset` directory format is a folder containing an SVG plus a `Contents.json` that Xcode treats as a vector image asset. Google publishes every Material Symbol in that exact format on `fonts.google.com/icons`. SkipUI's image-asset pipeline ships the same `.symbolset` bundle to both platforms and resolves it the same way. A single `.symbolset` works on iOS and Android.

### Pick + download

Browse [`fonts.google.com/icons`](https://fonts.google.com/icons). The Material Symbols catalogue has ~3000 glyphs covering every common UI need — checklist, bookmark, globe, code, reminders, cloud, all the way out to specialised pictographs. Note the **machine name** shown for each icon (`arrow_back`, `delete_sweep`, `home`, `notifications_active`). Underscored, lowercase. You'll use it as the SVG filename and as the `Image(...)` lookup name.

Use the **Apple** variant — its SVG contains the Apple symbol-grid metadata. The raw SVG download doesn't, and renders wrong inside a `.symbolset`. The canonical URL pattern:

```
https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/<name>/default/<name>_symbol.svg
```

Download a batch in one pass:

```bash
mkdir -p Sources/MyApp/Resources/Icons.xcassets
cd Sources/MyApp/Resources/Icons.xcassets
for name in checklist bookmark globe code reminders cloud add delete check_circle; do
    mkdir -p "${name}.symbolset"
    curl -sS -o "${name}.symbolset/${name}.svg" \
        "https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/${name}/default/${name}_symbol.svg"
done
```

A symbolset SVG is typically 5–20 KB. A 1 KB download is almost certainly a 404 HTML stub — verify with `wc -c` and `grep -l Symbols *.symbolset/*.svg`.

For the per-variant URLs (filled, rounded, sharp), the file-format details, and bulk-`Contents.json` generation, see [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).

### Drop into the xcassets

```
Sources/MyApp/Resources/
  Icons.xcassets/
    Contents.json                # xcassets-level marker — required
    checklist.symbolset/
      checklist.svg
      Contents.json              # symbolset-level — required
    bookmark.symbolset/
      bookmark.svg
      Contents.json
```

The directory name **must** end in `.symbolset`. Both `Contents.json` files are required. Without the xcassets-level one, SwiftPM ships the SVGs as plain files and `Image("checklist", bundle: .module)` returns a blank image — without an error. Bulk-generate both with the loop in [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).

### Reference from SwiftUI

```swift
Image("checklist", bundle: .module)
Image("arrow_back", bundle: .module)

Label {
    Text("Share", bundle: .module, comment: "share menu item")
} icon: {
    Image("share", bundle: .module)
}
```

The string passed to `Image(...)` is the **symbolset directory name without the `.symbolset` suffix** — `Image("checklist", bundle: .module)` resolves `Icons.xcassets/checklist.symbolset/checklist.svg` on both iOS and Android.

## The localisation-correct shape

The trailing-closure `Label { Text } icon: { Image }` form is the **canonical shape** for any labelled icon, because it solves two problems at once:

- **Cross-platform icon** — the `.symbolset` resource ships to both platforms via the same module bundle, so the icon renders identically on iOS and Android.
- **Translatable label** — the String Catalog extractor sees `Text(_, bundle: .module, comment:)` and adds the string to `Localizable.xcstrings` automatically.

```swift
// ✅ Works on both platforms; localisable; predictable.
Label {
    Text("Bookmark", bundle: .module, comment: "bookmark menu label")
} icon: {
    Image("bookmark", bundle: .module)
}
```

```swift
// ❌ Renders blank on Android; also not translatable.
Label("Bookmark", systemImage: "bookmark.fill")
```

For the localisation side of the pattern, see [skip-localization](../skip-localization/SKILL.md).

## Icon-only buttons (no visible label)

Even when there's no user-facing label text, **still use a `.symbolset`**, and attach an `accessibilityLabel` for VoiceOver/TalkBack:

```swift
Button(action: closeAction) {
    Image("close", bundle: .module)
}
.accessibilityIdentifier("button.close")
.accessibilityLabel(Text("Close", bundle: .module, comment: "accessibility label for the close button"))
```

`Image(systemName: "xmark")` happens to be on SkipUI's hardcoded list, so it would render on Android — but for the reasons in the previous section, you should still ship a `.symbolset`. The cost is one ~10 KB SVG; the benefit is a glyph that renders identically and predictably forever.

## App launcher icons — `skip icon`

The Material Symbols workflow above is for **in-app** glyphs. The launcher icon shown on the home screen / app drawer is a separate artefact:

- **iOS** — `AppIcon.appiconset/` under the app's xcassets, with a `Contents.json` listing every required size and scale.
- **Android** — `Android/app/src/main/res/mipmap-*/ic_launcher.png` plus an `adaptive-icon` XML.

The Skip CLI generates both from a single source image. The source is a **positional argument**, not a flag:

```bash
skip icon app_icon.png                                       # one PNG for both platforms
skip icon --background "#1A73E8" --inset 0.25 symbol.svg     # SVG overlay on a coloured background
skip icon --background "#5C6BC0-#3B3F54" symbol.svg          # gradient background
```

SVG, PDF, or PNG inputs are accepted. Re-run whenever the brand changes.

Useful flags:

| Flag | Purpose |
|------|---------|
| `--background <color>` | Background colour or gradient (e.g. `#1A73E8` or `#5C6BC0-#3B3F54`). |
| `--inset <decimal>` | Decimal fraction of the canvas to inset the foreground (e.g. `0.25`). |
| `--shadow <decimal>` | Shadow radius behind the foreground. |
| `--android <path>` | Use a different source image for Android. |
| `--darwin <path>` | Use a different source image for iOS. |
| `--open-preview` | Open the generated icon set in a preview window after generation. |

For the full flag list and additional examples, see the [`skip icon` reference](https://skip.dev/docs/skip-cli/#icon).

## Workflow for "add a new button with an icon"

Five steps end to end:

1. **Pick the glyph** at [`fonts.google.com/icons`](https://fonts.google.com/icons). Note the machine name (e.g. `delete_sweep`).
2. **Download the Apple-format SVG** from `fonts.gstatic.com/.../<name>/default/<name>_symbol.svg` into `Sources/<Module>/Resources/Icons.xcassets/<name>.symbolset/<name>.svg`. Verify size > 5 KB and `grep -l Symbols` succeeds.
3. **Write both `Contents.json` files** — the symbolset-level one referencing the SVG, and the xcassets-level marker. Use the bulk script in [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).
4. **Reference from SwiftUI** via `Image("<name>", bundle: .module)`, wrapped in `Label { Text(_, bundle: .module, comment: …) } icon: { … }` whenever the icon has a visible label.
5. **Build with `skip app launch`** and eyeball both platforms. A blank Android icon almost always means the `.symbolset` directory name doesn't match the `Image(...)` string, or the SVG isn't the Apple variant. (Or someone reached for `systemName:` again — search the diff for `systemImage:` and `systemName:` and replace each occurrence with the symbolset form.)

## Verifying icons render on both platforms

The cross-platform contract for a `.symbolset` is "shows on iOS, shows on Android, scales/tints with `.font(.system(size:))` and `.foregroundStyle(...)`". Verify by:

- Running `skip app launch` and eyeballing both screens side by side. A glyph that's blank on Android but visible on iOS is a missing `.symbolset` or a `systemImage:` / `systemName:` call that slipped in.
- For a UI-test suite, the icon-bearing button's `.accessibilityIdentifier(...)` will work regardless of whether the glyph rendered — but a Maestro `takeScreenshot:` of the button's region is what catches a blank icon. Capture screenshots into `.maestro/screenshots/` and review.

If iOS shows a glyph that Android doesn't:

| Symptom on Android                          | Likely cause                                                              |
|---------------------------------------------|---------------------------------------------------------------------------|
| Blank space where the icon should be        | An `Image(systemName: ...)` / `Label(_, systemImage:)` call. Replace with `.symbolset`. |
| Generic placeholder glyph                   | A `.symbolset` directory exists but `Contents.json` references the wrong file. |
| Glyph appears but is the wrong size/aspect  | The SVG is the raw Material download, not the Apple symbolset variant.    |
| Glyph appears but the wrong colour          | `.foregroundStyle(...)` not applied; or the SVG lacks the symbol-grid markers. |

## Auditing an existing codebase

When picking up a Skip project that was written without the discipline above (often by an agent in a different session, or by a Swift developer not yet aware of Skip's constraints), find every `systemName:` / `systemImage:` and replace it:

```bash
grep -rn "systemImage:\|systemName:" Sources/
```

For each hit, pick a Material Symbol with the same meaning at [`fonts.google.com/icons`](https://fonts.google.com/icons), download it into the right `Icons.xcassets/<name>.symbolset/`, and replace the call site:

```swift
// Before
Label("Tasks", systemImage: "checklist")
Image(systemName: "bookmark.fill")

// After
Label {
    Text("Tasks", bundle: .module, comment: "tasks tab label")
} icon: {
    Image("checklist", bundle: .module)
}
Image("bookmark_fill", bundle: .module)        // or "bookmark" if you want outlined
```

Then `skip app launch` and confirm both platforms now render the glyphs identically.

## Cross-references

- The `Label { Text } icon: { Image }` form is also the localisation-correct shape — see [skip-localization](../skip-localization/SKILL.md) for the full String Catalog workflow.
- For where icons fit into the broader SwiftUI-on-Skip surface, see [building-skip-ui](../building-skip-ui/SKILL.md).
- For verifying that an icon-bearing button responds correctly to user input, see [skip-ui-automation](../../../skip-testing-deployment/skills/skip-ui-automation/SKILL.md) — the icon button needs an `.accessibilityIdentifier(...)` so Maestro can find it whether the glyph rendered or not.
- The internal SkipUI behaviour for `systemName:` calls (the ~50-entry hardcoded map): [`references/sf-symbol-map.md`](references/sf-symbol-map.md). **Descriptive only — not a list to pick from.**
- The `.symbolset` directory format, download URL variants, and verification checks: [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).
