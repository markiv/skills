---
name: skip-localization
description: Translating Skip apps and libraries into other languages. Covers in-app strings via `Localizable.xcstrings` String Catalogs in each module's `Resources/` directory, iOS `Info.plist` usage descriptions via `Darwin/InfoPlist.xcstrings`, and App Store / Google Play metadata under `Darwin/fastlane/metadata/` and `Android/fastlane/metadata/android/`. Includes the `Text("…", bundle: .module, comment: "…")` extraction convention, the `state: needs_review` discipline for agent-produced translations, conservative idiomatic translation guidance, and the catalog-rewrite-from-scratch path for template-to-app conversions. The full xcstrings JSON schema is in references/xcstrings-schema.md; fastlane directory layout, byte budgets, and validation commands in references/fastlane-metadata.md; the BCP-47 ↔ App Store ↔ Play language-code mapping in references/language-codes.md. Use this skill whenever adding or updating translations, expanding the supported-language set, validating xcstrings JSON, or preparing store metadata for a release.
---

# Skip Localization

Skip apps and libraries are translated through three artifacts:

1. **`Sources/<Module>/Resources/Localizable.xcstrings`** — in-app strings in every Swift Package module, extracted from `Text("…", bundle: .module, comment: "…")` and friends.
2. **`Darwin/InfoPlist.xcstrings`** — iOS-only `Info.plist` localisable keys (the various `*UsageDescription` permission prompts).
3. **`Darwin/fastlane/metadata/<locale>/` and `Android/fastlane/metadata/android/<locale>/`** — App Store Connect and Google Play storefront metadata (title, subtitle, descriptions, keywords, release notes, screenshots).

Skip transpiles the in-app `xcstrings` to Android string resources at build time so the same key is translated on both platforms from one source.

For the deep details, this skill links out:

- [`references/xcstrings-schema.md`](references/xcstrings-schema.md) — the full `.xcstrings` JSON shape, translation-state machine, format specifiers / plurals / RTL marks, `xcstringstool` validation, JSON-hygiene pitfalls.
- [`references/fastlane-metadata.md`](references/fastlane-metadata.md) — directory trees for both stores, exact byte budgets per file, screenshot paths, the `skip verify` / `skip meta index` validation commands.
- [`references/language-codes.md`](references/language-codes.md) — BCP-47 (in-app) → App Store → Play code mapping, with a worked example and a generator script.

## Identifying translatable surfaces in a Skip project

Walk a Skip project's translation surface like this:

```bash
# In-app strings — every module that ships user-facing UI
find Sources -name "Localizable.xcstrings"

# iOS Info.plist permission prompts (only present when the app declares them)
ls Darwin/InfoPlist.xcstrings 2>/dev/null

# App Store metadata
ls Darwin/fastlane/metadata/

# Google Play metadata
ls Android/fastlane/metadata/android/
```

Each module gets its **own** `Localizable.xcstrings`. A two-module Skip app typically has at least `Sources/<App>/Resources/Localizable.xcstrings` (UI strings) and `Sources/<App>Model/Resources/Localizable.xcstrings` (data-layer strings). Translate them all.

## Authoring the source: how strings get into the catalog

User-facing strings must be created with the bundle-aware constructor so genstrings / the String Catalog extractor can see them:

```swift
Text("Settings", bundle: .module, comment: "settings sheet title")
Text("\(count) Tabs", bundle: .module, comment: "tabs title with count")

// String values for non-Text contexts
String(localized: "Cancel", bundle: .module, comment: "cancel button on the download dialog")

// Labels with both Text and an icon
Label {
    Text("Block Ads", bundle: .module, comment: "settings toggle label for blocking ads")
} icon: {
    Image("shield", bundle: .module)
}
```

What **doesn't** work:

```swift
Text("Settings")                            // never extracted — no bundle
Text(verbatim: "Settings")                  // explicitly opts out of localization
let title = "Settings"; Text(title)         // extractor can't see the literal
```

`Text(verbatim:)` is reserved for content that genuinely never translates — already-formatted numbers, URLs, usernames. Prefer the locale-aware formatted variants (`Text(count, format: .number)`) over `verbatim:` so locale-aware formatting still happens.

The `Label { Text } icon: { Image }` form is also the canonical icon shape because `Label(_, systemImage:)` mostly doesn't work cross-platform — see [skip-icons](../skip-icons/SKILL.md) for the icon side of the same pattern.

The `comment:` parameter is not optional in practice. Translators rely on it to disambiguate strings that look identical in English ("Open" the verb vs. "Open" the adjective). Write it from the translator's point of view — what they need to know to pick the right gender, formality, or word sense — not from the engineer's.

## Translation conventions

These are the conventions every Skip-project translation should follow.

### Pick a baseline language set, then expand from there

`Localizable.xcstrings` controls which languages each module supports. A reasonable starting set covers high-impact app-store regions across script families — Latin, Cyrillic, Arabic, CJK, Devanagari, Hangul. **The exact list is a project decision.** Inspect the existing `localizations` keys in the canonical module's catalog and treat that as the contract; new translations must cover every code already present, and any expansion to a new language must apply to every module's catalog and the `InfoPlist.xcstrings` uniformly.

When the project explicitly lists supported languages (CLAUDE.md, a translation script, a CI config), honour that list as the floor. Don't drop languages already present; add the missing ones.

### Mark every agent-produced translation `needs_review`

```json
"My Tasks" : {
  "comment" : "main screen navigation title",
  "localizations" : {
    "fr" : {
      "stringUnit" : {
        "state" : "needs_review",
        "value" : "Mes tâches"
      }
    }
  }
}
```

A human reviewer's job is to scan the `needs_review` set, not to discover that you've changed translations they previously approved. The state machine is in [`references/xcstrings-schema.md`](references/xcstrings-schema.md).

### Be conservative and idiomatic

- Translate the meaning, not the word. "Open" as a verb in the browser-tabs sense is different from "Open" as an adjective describing availability — the `comment:` tells you which.
- Prefer the term a native speaker would use in a mobile UI over a literal dictionary translation. "Reload" in a browser context is often the same loanword across European languages even though a dictionary might offer a native verb.
- Keep proper nouns in English unless there is a widely-recognised localised form. Brand names, font names, person names stay as-is. Country names, city names, and standard product categories take their localised form ("Germany" → "Deutschland", but "GitHub" stays "GitHub").
- Match the formality register of the source. Apple's HIG and Material's guidelines both lean informal in most languages; follow that unless the source string is explicitly formal.

### Terse beats verbose

App UI runs on phones with limited screen space. A label that becomes 1.5× longer in translation will truncate, wrap to two lines, or push neighbours offscreen. Prefer the shortest natural phrasing in the target language even when a longer one would feel more complete.

Examples:

- "Clear All Browsing Data" → in German, "Browserdaten löschen" (delete browser data) rather than "Alle Browser-Verlaufsdaten löschen".
- "Open Links in Background" → in Japanese, "リンクをバックグラウンドで開く" rather than a longer descriptive paraphrase.

### Comments are part of the contract

If the source string has **no `comment:`**, look up where it's used in the codebase, write a one-line context comment from the translator's POV, and add it to the catalog before translating. A translator (human or agent) without context produces guesses, not translations.

Finding usages:

```bash
# Search for the literal text — most strings appear only once
grep -rn '"Block Ads"' Sources/

# Or search by the catalog key from the .xcstrings JSON
grep -rn 'bundle: \.module, comment:' Sources/<Module>/ | grep -i "block ads"
```

When you add a comment to an existing entry, keep it terse and audience-appropriate (translator, not engineer). "label for the privacy section's ad-blocker toggle" is useful; "see BrowserTabView.swift line 312" is not.

## InfoPlist localization

`Darwin/InfoPlist.xcstrings` covers iOS-specific keys that appear in system permission dialogs and document type descriptions:

- `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSMicrophoneUsageDescription`, `NSLocationWhenInUseUsageDescription`, `NSContactsUsageDescription`, `NSCalendarsUsageDescription`, `NSRemindersUsageDescription`, `NSBluetoothAlwaysUsageDescription`, and other `*UsageDescription` keys.
- UTType / document-type descriptions for files the app declares it handles.
- `CFBundleDisplayName` if the app's name itself has a localised variant.

The JSON schema is identical to `Localizable.xcstrings` — see [`references/xcstrings-schema.md`](references/xcstrings-schema.md). The source-of-truth English strings live in `Darwin/Info.plist`; the InfoPlist catalog provides translations for whatever the plist exposes. When translating, support every language that the corresponding module's `Localizable.xcstrings` supports.

`InfoPlist.xcstrings` is iOS-only — Android requests permissions through manifest entries that don't carry user-facing description strings, so there's no Android counterpart to translate here.

## When the catalog is full of stale strings (template → app conversion)

`skip init` ships a populated `Localizable.xcstrings` for the **template** UI — Welcome / Home / Settings / "Hello Skipper!" / etc. After you rip out the template and write your real screens, those entries are still in the catalog, complete with the template's es / fr / ja / zh-Hans translations.

Two failure modes:

1. **Translations that don't apply.** Xcode and `xcstringstool` happily report 100% coverage for strings that are no longer referenced anywhere in the source. The String Catalog editor doesn't automatically prune entries when the source string disappears — `Editor → Sync` does, but only when run interactively from Xcode.
2. **Translators see noise.** If you hand the catalog off to a human translator (or another agent) before pruning, they may translate keys you've already deleted, costing time.

The clean path is to **rewrite the catalog from scratch** with only the strings the new code references, and re-add the supported languages programmatically:

```bash
# 1. Find every string my code references
grep -rn 'bundle: \.module, comment:' Sources/<Module>/*.swift > /tmp/strings.txt

# 2. Build the new catalog (one entry per string) with the supported language set
# 3. Mark every translation `"state" : "needs_review"`
# 4. Validate with xcrun xcstringstool print
```

## When the model module has no user-facing strings

A multi-module Skip app (`<App>` + `<AppModel>`) gets one `Localizable.xcstrings` per module. The template puts strings in **both** because its model exposes user-facing strings (item titles, date formatters, etc.). A pure-data model often has none — it's all `Codable` structs, persistence, and computed properties.

If `<AppModel>` has no `Text(_, bundle: .module, ...)` or `String(localized:, bundle:, ...)` calls, set its catalog to the **empty form**:

```json
{
  "sourceLanguage" : "en",
  "strings" : {

  },
  "version" : "1.0"
}
```

— rather than leaving the template's stale entries. An empty catalog still validates with `xcstringstool print`, ships zero extra bytes to either platform, and is unambiguous about what the module does and doesn't translate.

## When the source string changes

When the canonical English changes for an existing key, Xcode (and human reviewers) want to know the existing translations may be stale. Two options:

1. Add a new entry with the new English text as the key. Existing translations stay associated with the old key (which is now unused but historically translated).
2. Edit the existing translations and mark them `"state" : "needs_review"` to flag re-translation.

Option 2 is usually what you want — the strings table is keyed by the English source, so changing the key strands historical translations. Update the values to match the new English meaning and reset state to `needs_review`.

## Workflow for "add translations for all supported languages"

A common task. Steps:

1. **Inventory the catalogs.** `find Sources -name "Localizable.xcstrings"` and note the language set in any module that's already translated. That's the floor.
2. **Decide the supported set.** Either follow the project's documented list or expand the floor with the additional codes the task asks for. The set should be applied uniformly across every module and `InfoPlist.xcstrings`.
3. **For each catalog, walk every string key:**
   - If a `comment:` is missing, search the source for usage and add one before translating.
   - For each supported language not yet present, add a `localizations.<code>.stringUnit` entry with the translation and `"state" : "needs_review"`.
   - Preserve the existing JSON formatting style (space between `:` and value).
4. **Run `xcstringstool print` on every modified catalog** to confirm it parses.
5. **For fastlane metadata:** mirror the supported-language set to `Darwin/fastlane/metadata/<store-locale>/` and `Android/fastlane/metadata/android/<store-locale>/` — translating the BCP-47 catalog codes to the per-store codes ([`references/language-codes.md`](references/language-codes.md)). Translate every `.txt` file under the canonical English locale, respecting the [byte budgets](references/fastlane-metadata.md). Run `skip verify` and `skip meta index`.
6. **Be thorough and exhaustive.** Add translations for **every** supported language for **every** key — not just the obvious ones. An English fallback in production is a translation failure.

## Quick checklist

Before considering a translation pass complete:

- [ ] Every module's `Localizable.xcstrings` has been edited if it had any untranslated strings.
- [ ] `Darwin/InfoPlist.xcstrings` (if present) covers the same language set.
- [ ] Every new translation is marked `"state" : "needs_review"`.
- [ ] Every catalog passes `xcstringstool print`.
- [ ] Fastlane `.txt` files added/updated for every store-supported locale, byte budgets verified with `wc -c`.
- [ ] `skip verify` passes.
- [ ] `skip meta index` passes.
- [ ] Comments added for any source string that lacked one.

## Cross-references

- [skip-icons](../skip-icons/SKILL.md) — the `Label { Text } icon: { Image }` form is also the localization-correct shape; the two patterns share a syntax.
- [building-skip-ui](../building-skip-ui/SKILL.md) — where the bundle-aware `Text` constructor fits into the broader view-code surface.
- [skip-ui-automation](../../../skip-testing-deployment/skills/skip-ui-automation/SKILL.md) — locale-aware Maestro flows that generate the per-language screenshots used by fastlane.
- [skip-deployment](../../../skip-testing-deployment/skills/skip-deployment/SKILL.md) — where `skip verify` and `skip meta index` fit in the release pipeline.
- The xcstrings JSON schema: [`references/xcstrings-schema.md`](references/xcstrings-schema.md).
- Fastlane directory tree, byte budgets, validation: [`references/fastlane-metadata.md`](references/fastlane-metadata.md).
- BCP-47 → App Store → Play language-code mapping: [`references/language-codes.md`](references/language-codes.md).
