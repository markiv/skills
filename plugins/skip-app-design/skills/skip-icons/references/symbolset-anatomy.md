# `.symbolset` anatomy and download details

This file documents the on-disk format that Skip and Xcode both consume, the exact URLs that produce valid Apple-format SVGs, and how to verify a downloaded file is actually a symbolset rather than a 404 HTML stub.

For the workflow ("download a glyph, drop it in, reference it") see the main [skip-icons](../SKILL.md) skill. This is the spec.

## Apple symbolset format vs. raw Material SVG

`fonts.google.com/icons` offers multiple download formats per icon. Two are relevant:

| Format | Contains | Works in `.symbolset/` |
|---|---|---|
| **Apple** (the one you want) | SVG with the Apple symbol grid metadata (`Symbols`, `Regular-M` layer markers) | ✅ Renders correctly, scales, tints |
| **SVG** (raw download) | Plain SVG icon, no symbol grid | ❌ Renders the wrong size, can't be tinted |

The Apple variant is what Xcode generates when you drag an SF Symbol into an `xcassets` and choose "Symbol Image" — and exactly what SkipUI's image pipeline expects.

## Canonical download URLs

Google's CDN serves Apple-format SVGs at predictable paths:

```
# Outlined variant (most common)
https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/<name>/default/<name>_symbol.svg

# Filled variant
https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsoutlined/<name>/fill1/<name>_fill1_symbol.svg

# Rounded variant
https://fonts.gstatic.com/s/i/short-term/release/materialsymbolsrounded/<name>/default/<name>_symbol.svg

# Sharp variant
https://fonts.gstatic.com/s/i/short-term/release/materialsymbolssharp/<name>/default/<name>_symbol.svg
```

`<name>` is the underscored lowercase machine name shown on the icon's page (`arrow_back`, `delete_sweep`, `notifications_active`, etc.). The `_symbol.svg` suffix is what makes it the Apple format — a URL without the `_symbol` suffix returns the raw Material SVG.

## Verifying the download

A symbolset SVG is typically **5–20 KB**. A response under **1 KB** is almost certainly a 404 HTML stub (Google's CDN returns a small error page for unknown names rather than a real 404 status).

Two checks:

```bash
# Size — should be > 5000
wc -c Sources/MyApp/Resources/Icons.xcassets/home.symbolset/home.svg

# Apple-format marker — the SVG should contain the string "Symbols" as a layer label
grep -l Symbols Sources/MyApp/Resources/Icons.xcassets/*/*.svg
```

If a glyph passes both checks, it will work in a `.symbolset/`. If the size check fails, the name is probably misspelled — go back to `fonts.google.com/icons`, search, and copy the machine name exactly.

## Directory layout

```
Sources/MyApp/Resources/
  Icons.xcassets/
    Contents.json                # xcassets-level marker — required
    home.symbolset/
      home.svg
      Contents.json              # symbolset-level — required
    settings.symbolset/
      settings.svg
      Contents.json
    arrow_back.symbolset/
      arrow_back.svg
      Contents.json
```

Two non-obvious rules:

1. **The directory name *must* end in `.symbolset`.** Xcode and Skip's image pipeline both look for that suffix to decide whether to treat the directory as a vector asset bundle. A directory named `home/` containing `home.svg` and `Contents.json` is not an asset; it's just a folder.
2. **The xcassets-level `Contents.json` is required.** Without it, SwiftPM ships the SVGs as plain bundle resources, not vector assets, and `Image("home", bundle: .module)` returns a blank image on both platforms — without an error.

The SVG filename itself can be anything; the convention is `<name>.svg` matching the directory name, but the `filename` field of the symbolset `Contents.json` is what actually resolves it.

## `Contents.json` schemas

The two `Contents.json` files have different schemas.

### Symbolset-level (`<name>.symbolset/Contents.json`)

```json
{
  "info" : {
    "author" : "xcode",
    "version" : 1
  },
  "symbols" : [
    {
      "filename" : "home.svg",
      "idiom" : "universal"
    }
  ]
}
```

| Field | Notes |
|---|---|
| `info.author` | Always `"xcode"`. |
| `info.version` | Always `1`. |
| `symbols[].filename` | The SVG filename in the same directory. **Must match the on-disk filename exactly** — case-sensitive on case-sensitive filesystems. |
| `symbols[].idiom` | `"universal"` unless you ship per-idiom variants (`"iphone"`, `"ipad"`, `"watch"`). Per-idiom variants do not transpile to Compose; on Android the universal variant is always used. |

### xcassets-level (`Icons.xcassets/Contents.json`)

```json
{
  "info" : {
    "author" : "xcode",
    "version" : 1
  }
}
```

Just the directory marker. No `symbols`, no per-asset config.

## Generating both `Contents.json` files in bulk

After downloading a batch of SVGs, this loop writes the per-symbolset `Contents.json` for everything found:

```bash
cd Sources/MyApp/Resources/Icons.xcassets

# Per-symbolset Contents.json
for d in *.symbolset/; do
    name="$(basename "${d%.symbolset/}")"
    cat > "${d}Contents.json" <<EOF
{
  "info" : { "author" : "xcode", "version" : 1 },
  "symbols" : [ { "filename" : "${name}.svg", "idiom" : "universal" } ]
}
EOF
done

# xcassets-level marker
cat > Contents.json <<EOF
{
  "info" : { "author" : "xcode", "version" : 1 }
}
EOF
```

## Tinting and sizing

A symbolset rendered through `Image(..., bundle: .module)` participates in the SwiftUI / Compose modifier system the same way as a built-in icon:

```swift
Image("home", bundle: .module)
    .resizable()
    .scaledToFit()
    .frame(width: 24, height: 24)
    .foregroundStyle(Color.accentColor)
```

`.foregroundStyle(...)` only tints correctly if the SVG is the Apple variant (the `Symbols` grid markers are what tell Xcode and Skip "this is monochrome, tint me"). A raw Material SVG renders with its original colour and ignores `.foregroundStyle`.

For pixel-perfect rendering on Android, set an explicit `.frame(width:height:)` — Compose can't reproduce SF Symbol's "scales-with-font-size" behaviour automatically.
