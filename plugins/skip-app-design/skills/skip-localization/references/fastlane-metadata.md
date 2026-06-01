# Fastlane storefront metadata — file layout, byte budgets, validation

This file documents the on-disk structure for App Store Connect (`Darwin/fastlane/`) and Google Play (`Android/fastlane/`) storefront metadata, the byte budgets enforced by the stores, and the validation commands Skip ships.

For the language-code mapping per store (BCP-47 → App Store vs. Play codes), see [`language-codes.md`](language-codes.md). For the workflow ("translate the storefront metadata for the next release"), see the main [skip-localization](../SKILL.md) skill.

## Directory trees

```
Darwin/fastlane/metadata/<locale>/
  title.txt                 # 30-byte max — App Store listing title
  subtitle.txt              # 30-byte max
  keywords.txt              # 100-byte max — comma-separated
  description.txt           # 4000-byte max
  release_notes.txt         # 4000-byte max — what changed in this version
  promotional_text.txt      # 170-byte max — editable between submissions
  marketing_url.txt
  privacy_url.txt
  support_url.txt

Darwin/fastlane/metadata/default/
  …same files…              # iOS source-of-truth English (newer fastlane)

Darwin/fastlane/metadata/en-US/
  …same files…              # iOS source-of-truth English (older fastlane)

Darwin/fastlane/screenshots/<locale>/
  1_<locale>.png            # phone screenshots, numbered
  2_<locale>.png
  3_<locale>.png

Android/fastlane/metadata/android/<locale>/
  title.txt                 # 30-byte max — Google Play listing title
  short_description.txt     # 80-byte max
  full_description.txt      # 4000-byte max
  video.txt                 # YouTube URL, optional
  changelogs/<versionCode>.txt   # release notes for a given build
  images/
    icon.png
    featureGraphic.png
    phoneScreenshots/
      1_<locale>.png        # phone screenshots, numbered
      2_<locale>.png
      3_<locale>.png

Android/fastlane/metadata/android/en-US/
  …source-of-truth English…
```

The `default/` vs `en-US/` split on Darwin reflects different fastlane versions — newer setups use `default/`, older ones use `en-US/`. Skip's templates create `en-US/`; both work as the canonical English source.

## Byte budgets, not character budgets

**Every length limit is a byte count of the UTF-8-encoded file**, not a Unicode codepoint count.

- ASCII characters: 1 byte each.
- Latin-with-accents (`é`, `ü`, `ñ`): 2 bytes each.
- CJK ideographs (`待`, `办`, `タ`): 3 bytes each.
- Emoji: 4+ bytes each.

A 30-character Japanese title is 90 bytes — over the 30-byte limit. A 30-character German title with three umlauts is 33 bytes — also over.

Always verify with `wc -c`, never by eyeballing:

```bash
wc -c Darwin/fastlane/metadata/de-DE/title.txt
wc -c Android/fastlane/metadata/android/ja-JP/short_description.txt
```

### Limits summary

| File | Max bytes | Notes |
|---|---|---|
| `Darwin/.../title.txt` | 30 | App Store listing title |
| `Darwin/.../subtitle.txt` | 30 | Below the title in the storefront |
| `Darwin/.../keywords.txt` | 100 | Comma-separated; the commas count |
| `Darwin/.../description.txt` | 4000 | Main marketing description |
| `Darwin/.../release_notes.txt` | 4000 | What's-new for the current version |
| `Darwin/.../promotional_text.txt` | 170 | Editable between submissions |
| `Android/.../title.txt` | 30 | Google Play listing title |
| `Android/.../short_description.txt` | 80 | Shown in search results |
| `Android/.../full_description.txt` | 4000 | Main marketing description |
| `Android/.../changelogs/N.txt` | 500 | Per-versionCode release notes |

If a translation is over budget, tighten it — the limits are hard. App Store Connect and Play Console reject uploads that exceed them; Skip's release tooling catches this earlier.

## Validation

Skip ships two validators that run before any upload:

```bash
skip verify           # validates lengths, required files, image dimensions
skip meta index       # checks that metadata can be converted into an app index
```

`skip verify` is the fast first pass — it catches byte-budget overruns and missing required files. `skip meta index` exercises the deeper schema (image dimensions, JSON structure of `rating.json` / `app_privacy_details.json`) and is what release pipelines run.

A typical pre-release check:

```bash
skip verify && skip meta index
```

Both must pass before `skip export -c release`.

## Updating existing translations

Storefront metadata changes with every release. When asked to "update the translations":

1. Read the canonical English (`Darwin/fastlane/metadata/en-US/` or `default/`, `Android/fastlane/metadata/android/en-US/`).
2. For each existing translated locale, compare against the canonical version — is the translation still accurate for the current English text? If not, update it.
3. Add translations for any locale present in the in-app catalog but missing from `fastlane/metadata/`.
4. Verify byte budgets with `wc -c` on every modified `.txt` file.
5. Run `skip verify` and `skip meta index`.

`release_notes.txt` (iOS) and `changelogs/<versionCode>.txt` (Android) are **version-scoped** — only the current release's notes need translation; old release notes are frozen.

## Screenshots

| Platform | Path | Sizes typically used |
|---|---|---|
| App Store Connect | `Darwin/fastlane/screenshots/<locale>/N_<locale>.png` | iPhone 6.7" (1290×2796), iPhone 6.5" (1242×2688), iPad 12.9" (2048×2732) |
| Google Play | `Android/fastlane/metadata/android/<locale>/images/phoneScreenshots/N_<locale>.png` | 16:9 or 9:16 aspect, min 320 px short edge, max 3840 px long edge |

Each store accepts up to 10 phone screenshots per locale; both require at least 1 per locale you list. The filename prefix `N_` (1_, 2_, ...) controls display order.

Generate these from Maestro flows — see the locale-aware walkthrough pattern in [skip-ui-automation](../../../../skip-testing-deployment/skills/skip-ui-automation/references/locale-switching.md).

## File-by-file checklist

Before considering a fastlane translation pass complete:

- [ ] `title.txt` exists in every locale, ≤ 30 bytes.
- [ ] `subtitle.txt` (iOS) ≤ 30 bytes, `short_description.txt` (Android) ≤ 80 bytes.
- [ ] `keywords.txt` (iOS) ≤ 100 bytes, comma-separated, no trailing comma.
- [ ] `description.txt` / `full_description.txt` ≤ 4000 bytes.
- [ ] `release_notes.txt` (iOS) / `changelogs/<N>.txt` (Android) populated for the current version.
- [ ] At least 1, typically 3–5, `phoneScreenshots/N_<locale>.png` per locale.
- [ ] URL files (`privacy_url.txt`, `support_url.txt`) copied to every locale (URLs rarely localise but the files must exist).
- [ ] `skip verify` exits 0.
- [ ] `skip meta index` exits 0.
