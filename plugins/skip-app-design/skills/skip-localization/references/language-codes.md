# Language code mapping — `Localizable.xcstrings` vs. App Store vs. Google Play

The in-app catalog uses BCP-47 codes. The two storefronts use different (incompatible) supersets. This file maps them.

For the workflow ("which directories do I need under `fastlane/metadata/`"), see [`fastlane-metadata.md`](fastlane-metadata.md).

## Why this matters

A common mistake: you translate `Localizable.xcstrings` into `fr` and `zh-Hans`, then create `Darwin/fastlane/metadata/fr/` and `Android/fastlane/metadata/android/zh-Hans/`. Both fastlane runs reject the upload — `fr` isn't a valid App Store locale (it wants `fr-FR`), and `zh-Hans` isn't a valid Play locale (it wants `zh-CN`).

The mapping is one-way (catalog → store-specific), and the two stores disagree on regional suffixes for several languages.

## Mapping table

| Language | `Localizable.xcstrings` (BCP-47) | App Store Connect | Google Play |
|---|---|---|---|
| English (US) | `en` | `en-US` | `en-US` |
| English (UK) | `en-GB` | `en-GB` | `en-GB` |
| French (France) | `fr` | `fr-FR` | `fr-FR` |
| French (Canada) | `fr-CA` | `fr-CA` | `fr-CA` |
| German | `de` | `de-DE` | `de-DE` |
| Spanish (Spain) | `es` | `es-ES` | `es-ES` |
| Spanish (Latin America) | `es-419` | `es-MX` | `es-419` |
| Portuguese (Brazil) | `pt-BR` | `pt-BR` | `pt-BR` |
| Portuguese (Portugal) | `pt-PT` | `pt-PT` | `pt-PT` |
| Italian | `it` | `it` | `it-IT` |
| Dutch | `nl` | `nl-NL` | `nl-NL` |
| Russian | `ru` | `ru` | `ru-RU` |
| Polish | `pl` | `pl` | `pl-PL` |
| Turkish | `tr` | `tr` | `tr-TR` |
| Arabic | `ar` | `ar-SA` | `ar` |
| Hebrew | `he` | `he` | `iw` (Play uses old code) |
| Hindi | `hi` | `hi` | `hi-IN` |
| Thai | `th` | `th` | `th` |
| Indonesian | `id` | `id` | `id` |
| Vietnamese | `vi` | `vi` | `vi` |
| Simplified Chinese | `zh-Hans` | `zh-Hans` | `zh-CN` |
| Traditional Chinese (Taiwan) | `zh-Hant` | `zh-Hant` | `zh-TW` |
| Traditional Chinese (Hong Kong) | `zh-HK` | `zh-Hans` (no HK variant) | `zh-HK` |
| Japanese | `ja` | `ja` | `ja-JP` |
| Korean | `ko` | `ko` | `ko-KR` |

## Notes per store

### App Store Connect

- Reference: https://docs.fastlane.tools/actions/deliver/#available-language-codes
- **Region-specific variants are required where the store distinguishes them.** A bare `de` is rejected where `de-DE` is required; bare `fr` is rejected where `fr-FR` is required.
- A few languages have **no regional suffix** in the App Store code (`ja`, `ko`, `it`, `tr`, `ru`, `pl`, `th`, `vi`) even though Play uses the regional form.
- Chinese: App Store distinguishes script (`zh-Hans` / `zh-Hant`), not region. Hong Kong Traditional must fall back to `zh-Hant`.

### Google Play

- Reference: https://support.google.com/googleplay/android-developer/answer/9844778
- **Almost always region-suffixed.** Bare codes generally only work for languages with one practical region (`th`, `id`, `vi`).
- Chinese: Play uses regional codes (`zh-CN`, `zh-TW`, `zh-HK`), not script. A `zh-Hans` directory will be rejected.
- Hebrew uses the legacy `iw` code on Play, not `he`. (This is a long-standing Android quirk.)
- Latin American Spanish is `es-419` (UN region code), not `es-MX`.

## A worked example

The `skipapp-todo` sample app supports English, French, and Simplified Chinese. Its directory layout:

```
Sources/TodoApp/Resources/Localizable.xcstrings    # keys "fr" and "zh-Hans"

Darwin/fastlane/metadata/en-US/                    # iOS English
Darwin/fastlane/metadata/fr-FR/                    # iOS French
Darwin/fastlane/metadata/zh-Hans/                  # iOS Simplified Chinese (script-based)

Android/fastlane/metadata/android/en-US/           # Play English
Android/fastlane/metadata/android/fr-FR/           # Play French
Android/fastlane/metadata/android/zh-CN/           # Play Simplified Chinese (region-based)
```

Three languages × two stores = six directories on disk, with `fr` mapping the same on both stores but `zh-Hans` and `zh-CN` differing per store. Expand the table above when adding more languages.

## A quick script to generate the right directory names

```bash
# Given a BCP-47 code from Localizable.xcstrings, print the two store codes.
bcp47_to_stores() {
    case "$1" in
        en)        echo "en-US en-US" ;;
        fr)        echo "fr-FR fr-FR" ;;
        de)        echo "de-DE de-DE" ;;
        es)        echo "es-ES es-ES" ;;
        pt-BR)     echo "pt-BR pt-BR" ;;
        ja)        echo "ja ja-JP" ;;
        ko)        echo "ko ko-KR" ;;
        zh-Hans)   echo "zh-Hans zh-CN" ;;
        zh-Hant)   echo "zh-Hant zh-TW" ;;
        ar)        echo "ar-SA ar" ;;
        he)        echo "he iw" ;;
        *)         echo "UNKNOWN UNKNOWN" ;;
    esac
}

# Usage: for each catalog language, create both store dirs
for lang in en fr zh-Hans; do
    read ios play <<< "$(bcp47_to_stores $lang)"
    mkdir -p Darwin/fastlane/metadata/$ios
    mkdir -p Android/fastlane/metadata/android/$play
done
```

Extend this script as you add languages; commit it to the project so the mapping is local to each repo.
