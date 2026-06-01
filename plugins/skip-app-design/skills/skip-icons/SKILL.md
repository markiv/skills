---
name: skip-icons
description: Working with icons in Skip projects. Why `Image(systemName:)` and `Label(_, systemImage:)` silently fail on Android for the vast majority of SF Symbols (only ~50 names have hardcoded Material Icon equivalents in SkipUI), and the canonical alternative — downloading Material Symbols from `fonts.google.com/icons` in **Apple symbolset format**, dropping the `.symbolset/` bundle under `Sources/<Module>/Resources/Icons.xcassets/`, and rendering with `Image("name", bundle: .module)` inside a `Label { Text(…, bundle: .module) } icon: { Image(…) }`. Also covers app launcher icons via `skip icon` and how to verify icons render on both platforms. The exhaustive SF Symbol → Material Icon compatibility table is in references/sf-symbol-map.md; the full `.symbolset` format spec (download URLs per variant, Contents.json schemas, verification checks) is in references/symbolset-anatomy.md. Use this skill whenever adding, replacing, or auditing icons in a Skip app or library.
---

# Skip Icons

Icons are the single biggest source of "looks fine on iOS, blank on Android" bugs in Skip projects. The cause is almost always reaching for `Image(systemName:)` / `Label(_, systemImage:)` and assuming SF Symbols transpile. They don't, except for a small hardcoded compatibility list.

## The trap — why `systemImage:` mostly doesn't work

SwiftUI's SF Symbols catalog is an Apple-only registry of thousands of glyphs accessed by name. Android has no equivalent registry. SkipUI hardcodes a compatibility map from a small set of SF Symbol names to Jetpack Compose's `Icons.Outlined.*` / `Icons.Filled.*` Material Icons — fewer than 50 names total. Every other SF Symbol passed to `systemImage:` falls through to a default branch that renders nothing (or a generic placeholder) on Android, while still rendering correctly on iOS.

So these compile, run on both platforms, and one of them looks broken:

```swift
Label("Home",     systemImage: "house.fill")             // ✅ works on both — hardcoded
Label("Settings", systemImage: "gearshape.fill")         // ✅ works on both — hardcoded
Label("Search",   systemImage: "magnifyingglass")        // ✅ works on both — hardcoded

Label("Share",    systemImage: "square.and.arrow.up")    // ✅ works (hardcoded)
Label("Bookmark", systemImage: "bookmark.fill")          // ❌ blank on Android — not in list
Label("Web",      systemImage: "globe.americas")         // ❌ blank on Android — not in list
Label("Code",     systemImage: "chevron.left.forward.slash.chevron.right") // ❌ blank on Android
```

The supported set is small enough to enumerate. Treat it as a closed list — assume any SF Symbol not on the list will not render on Android.

→ The full hardcoded SF Symbol → Material Icon table is in [`references/sf-symbol-map.md`](references/sf-symbol-map.md).

## The canonical alternative — Material Symbols in Apple symbolset format

Apple's `.symbolset` directory format is a folder containing an SVG plus a `Contents.json` that Xcode treats as a vector image asset. Google publishes every Material Symbol in that exact format on `fonts.google.com/icons` — and SkipUI's image-asset pipeline transpiles the same bundle into a Compose `painterResource(...)` lookup. So a single `.symbolset` ships to both platforms.

### Pick + download

Browse `https://fonts.google.com/icons`. The Material Symbols catalog has ~3000 glyphs covering every common UI need. Note the **machine name** shown for each icon — `arrow_back`, `delete_sweep`, `home`, `notifications_active`. Underscored, lowercase. You'll use this both as the SVG filename and as the `Image(...)` lookup name.

Use the **Apple** variant — its SVG contains the Apple symbol grid metadata. The raw SVG download doesn't, and looks wrong inside a `.symbolset`. The canonical URL pattern:

```
https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/<name>/default/<name>_symbol.svg
```

Download a batch in one pass:

```bash
mkdir -p Sources/MyApp/Resources/Icons.xcassets
cd Sources/MyApp/Resources/Icons.xcassets
for name in home settings search add delete check_circle; do
    mkdir -p "${name}.symbolset"
    curl -sS -o "${name}.symbolset/${name}.svg" \
        "https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/${name}/default/${name}_symbol.svg"
done
```

A symbolset SVG is typically 5–20 KB. A 1 KB download is almost certainly a 404 HTML stub — verify with `wc -c` and `grep -l Symbols *.symbolset/*.svg`.

For the per-variant URLs (filled, rounded, sharp), the file-format details, and bulk-Contents.json generation, see [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).

### Drop into the xcassets

```
Sources/MyApp/Resources/
  Icons.xcassets/
    Contents.json                # xcassets-level marker — required
    home.symbolset/
      home.svg
      Contents.json              # symbolset-level — required
    arrow_back.symbolset/
      arrow_back.svg
      Contents.json
```

The directory name **must** end in `.symbolset`. Both `Contents.json` files are required. Without the xcassets-level one, SwiftPM ships the SVGs as plain files and `Image("home", bundle: .module)` returns a blank image — without an error. Bulk-generate both with the loop in [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).

### Reference from SwiftUI

```swift
Image("home", bundle: .module)
Image("arrow_back", bundle: .module)

Label {
    Text("Share", bundle: .module, comment: "share menu item")
} icon: {
    Image("share", bundle: .module)
}
```

The string passed to `Image(...)` is the **symbolset directory name without the `.symbolset` suffix** — `Image("home", bundle: .module)` resolves `Icons.xcassets/home.symbolset/home.svg` on both iOS and Android.

## The localization-correct shape

The trailing-closure `Label { Text } icon: { Image }` form is the **canonical shape** because it solves two problems at once:

- **Cross-platform icon** — the `.symbolset` resource ships to both platforms via the same module bundle, so the icon renders identically on iOS and Android.
- **Translatable label** — the String Catalog extractor sees `Text(_, bundle: .module, comment:)` and adds the string to `Localizable.xcstrings` automatically.

```swift
// ✅ Works on both platforms; localizable; predictable
Label {
    Text("Bookmark", bundle: .module, comment: "bookmark menu label")
} icon: {
    Image("bookmark", bundle: .module)
}
```

```swift
// ❌ Blank on Android (bookmark.fill not in compatibility map); also not translatable
Label("Bookmark", systemImage: "bookmark.fill")
```

For the localization side of the pattern, see [skip-localization](../skip-localization/SKILL.md).

## App launcher icons — `skip icon`

The Material Symbols workflow is for **in-app** glyphs. The launcher icon shown on the home screen / app drawer is a separate artefact:

- **iOS** — `AppIcon.appiconset/` under the app's xcassets, with a `Contents.json` listing every required size and scale.
- **Android** — `Android/app/src/main/res/mipmap-*/ic_launcher.png` plus an `adaptive-icon` XML.

The Skip CLI generates both from a single source image. The source is a **positional argument**, not a flag:

```bash
skip icon app_icon.png                                 # use a single PNG for both platforms
skip icon --background "#1A73E8" --inset 0.25 symbol.svg  # SVG overlay on a coloured background
skip icon --background "#5C6BC0-#3B3F54" symbol.svg    # gradient background
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

1. **Pick the glyph** on `https://fonts.google.com/icons`, note the machine name (e.g. `delete_sweep`).
2. **Download the Apple-format SVG** from `fonts.gstatic.com/.../<name>/default/<name>_symbol.svg` into `Sources/<Module>/Resources/Icons.xcassets/<name>.symbolset/<name>.svg`. Verify size > 5 KB and `grep -l Symbols` succeeds.
3. **Write both `Contents.json` files** — the symbolset-level one referencing the SVG, and the xcassets-level marker. Use the bulk script in [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).
4. **Use it in SwiftUI** via `Image("<name>", bundle: .module)`, wrapped in a `Label { Text(_, bundle: .module, comment: …) } icon: { … }` for any button or menu item that has a visible label.
5. **Build with `skip app launch`** and eyeball both platforms. Blank Android icon almost always means the `.symbolset` directory name doesn't match the `Image(...)` string, or the SVG isn't the Apple variant.

## Verifying icons render on both platforms

The cross-platform contract for a `.symbolset` is "shows on iOS, shows on Android, scales/tints with `.font(.system(size:))` and `.foregroundStyle(...)`". Verify by:

- Running `skip app launch` and eyeballing both screens side by side. A glyph that's blank on Android but visible on iOS is a missing symbolset or a `systemImage:` fallback.
- For a UI test suite, the icon-bearing button's `.accessibilityIdentifier(...)` will work regardless of whether the glyph rendered — but a Maestro `takeScreenshot:` of the button's region is what catches a blank icon. Capture screenshots into `.maestro/screenshots/` and review.

If iOS shows a glyph that Android doesn't, the cause is almost always one of:

| Symptom on Android                          | Likely cause                                                              |
|---------------------------------------------|---------------------------------------------------------------------------|
| Blank space where the icon should be        | `Image(systemName:)` for a symbol not in SkipUI's hardcoded map           |
| Generic placeholder glyph                   | `.symbolset` directory exists but `Contents.json` references wrong file   |
| Glyph appears but is the wrong size/aspect  | SVG is the raw Material download, not the Apple symbolset variant         |
| Glyph appears but the wrong colour          | `.foregroundStyle(...)` not applied; or the SVG lacks the symbol grid markers |

## When to use `systemImage:` vs `Image("name", bundle: .module)`

| Situation                                                              | Use                                              |
|------------------------------------------------------------------------|--------------------------------------------------|
| SF Symbol name is on the [hardcoded compatibility list](references/sf-symbol-map.md) | `Label("…", systemImage: "…")` is fine          |
| Any other SF Symbol name                                               | `Image("name", bundle: .module)` + `.symbolset` |
| You want the label to be translatable (any user-facing button label)   | `Label { Text("…", bundle: .module, comment: "…") } icon: { Image(…) }` — always |
| Pure-icon button (no visible label) and the glyph is on the list       | `Image(systemName: "…")` works, but still attach `.accessibilityLabel(…)` |
| Pure-icon button and the glyph is NOT on the list                      | `Image("name", bundle: .module)` + `.accessibilityLabel(…)` |

When in doubt, ship a symbolset. The cost is one ~10 KB SVG file; the benefit is a glyph that renders the same on both platforms forever.

## Cross-references

- The `Label { Text } icon: { Image }` form is also the localization-correct shape — see [skip-localization](../skip-localization/SKILL.md) for the full String Catalog workflow.
- For where icons fit into the broader SwiftUI-on-Skip surface, see [building-skip-ui](../building-skip-ui/SKILL.md).
- For verifying that an icon-bearing button responds correctly to user input, see [skip-ui-automation](../../../skip-testing-deployment/skills/skip-ui-automation/SKILL.md) — the icon button needs an `.accessibilityIdentifier(...)` so Maestro can find it whether the glyph rendered or not.
- The closed list of SF Symbol → Material Icon mappings: [`references/sf-symbol-map.md`](references/sf-symbol-map.md).
- The `.symbolset` directory format, download URL variants, and verification checks: [`references/symbolset-anatomy.md`](references/symbolset-anatomy.md).
