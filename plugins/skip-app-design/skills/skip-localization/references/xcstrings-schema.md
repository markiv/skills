# `.xcstrings` String Catalog — JSON schema reference

A String Catalog is a plain-JSON file that Xcode can edit visually but is also safe to edit programmatically. This file documents the schema, the translation-state machine, the format-specifier rules, and the validation toolchain.

For the workflow ("translate a project's catalogs into French and Chinese"), see the main [skip-localization](../SKILL.md) skill. This is the spec.

## Top-level structure

```json
{
  "sourceLanguage" : "en",
  "strings" : {
    "Settings" : {
      "comment" : "settings sheet title",
      "localizations" : {
        "de" : {
          "stringUnit" : {
            "state" : "translated",
            "value" : "Einstellungen"
          }
        },
        "fr" : {
          "stringUnit" : {
            "state" : "needs_review",
            "value" : "Réglages"
          }
        }
      }
    },
    "%lld Tabs" : {
      "comment" : "tabs title showing the number of open tabs",
      "localizations" : {
        "de" : {
          "stringUnit" : {
            "state" : "translated",
            "value" : "%lld Tabs"
          }
        }
      }
    }
  },
  "version" : "1.0"
}
```

| Field | Notes |
|---|---|
| `sourceLanguage` | The catalog's source-of-truth language. Almost always `"en"` for Skip projects. |
| `strings` | A map keyed by the **English source string** (or whatever `sourceLanguage` is). Each value is a "string entry". |
| `version` | Always `"1.0"` for current Xcode releases. |

A string entry contains:

| Field | Notes |
|---|---|
| `comment` | Translator-facing context. Recommended for *every* entry; absent for strings that haven't been translated yet. |
| `localizations` | A map of locale codes (BCP-47) to `stringUnit` blocks. |
| `extractionState` | Optional; Xcode writes `"manual"` for hand-added entries, omits for genstrings-extracted ones. |

A `stringUnit` contains:

| Field | Notes |
|---|---|
| `state` | One of `new`, `needs_review`, `translated`. See the state machine below. |
| `value` | The translated string. |

## Translation state machine

| `state`         | Meaning                                                         | When to use                                                        |
|-----------------|-----------------------------------------------------------------|--------------------------------------------------------------------|
| `new`           | Source string changed; existing translation may be stale        | Auto-assigned by Xcode when the source moves                       |
| `needs_review`  | Translation exists but a human should look at it before shipping | **Default for any agent-produced translation**                     |
| `translated`    | Reviewed and approved                                            | Only after a human translator signs off                            |
| (entry missing) | Use the source string as the value                              | Acceptable when source and target language match exactly (rare)    |

**Mark every agent-produced translation `"state" : "needs_review"`.** It tells the human reviewer where to look and keeps Xcode's translation-progress indicator honest.

## Format specifiers

Skip / iOS use printf-style specifiers. The rules:

- **Pass specifiers through verbatim.** Never translate `%lld`, `%@`, `%lf`, `%d` — only the surrounding words.
- **Count must match.** Every translation of `"%lld Tabs"` must include exactly one `%lld`. Mismatched counts compile and ship, then crash at runtime with `EXC_BAD_ACCESS` or a Kotlin format-string exception.
- **Order with positional specifiers.** For multi-argument strings like `"Version %@ (%@)"`, use positional `%1$@`, `%2$@` in translations so word order can change without breaking the substitution: `"Versão %1$@ (%2$@)"`.

```json
"Version %@ (%@)" : {
  "comment" : "Settings label showing the current version of the app",
  "localizations" : {
    "fr" : {
      "stringUnit" : {
        "state" : "translated",
        "value" : "Version %1$@ (%2$@)"
      }
    },
    "ja" : {
      "stringUnit" : {
        "state" : "translated",
        "value" : "バージョン %1$@ (%2$@)"
      }
    }
  }
}
```

## Plural variations

When a string varies by count, use `variations.plural.{one,other,zero,...}` instead of a single `stringUnit`:

```json
"%lld Tab" : {
  "comment" : "tab count label, with proper pluralisation",
  "localizations" : {
    "en" : {
      "variations" : {
        "plural" : {
          "one" : {
            "stringUnit" : {
              "state" : "translated",
              "value" : "%lld Tab"
            }
          },
          "other" : {
            "stringUnit" : {
              "state" : "translated",
              "value" : "%lld Tabs"
            }
          }
        }
      }
    }
  }
}
```

Languages with non-binary plural categories (Russian, Polish, Arabic) require additional `zero` / `few` / `many` / `two` keys per CLDR. If the catalog has no `variations`, treat the entry as a single literal and use `"%lld <Word>"` style; only introduce variations when the source actually depends on a count.

## RTL marks (Arabic / Hebrew)

Arabic and Hebrew translations may include the right-to-left mark (U+200F, the invisible character before format specifiers) to keep numerals and punctuation grouped correctly:

```json
"%lld Tabs" : {
  "localizations" : {
    "ar" : {
      "stringUnit" : {
        "state" : "translated",
        "value" : "%lld‏علامة تبويب"
      }
    }
  }
}
```

Leave existing RTL marks intact when editing translations. When adding new Arabic / Hebrew strings, look at neighbouring entries in the catalog for the convention used.

## Validating with `xcstringstool`

After editing any `.xcstrings` file, validate it with the Xcode-bundled tool:

```bash
xcrun xcstringstool print Sources/<Module>/Resources/Localizable.xcstrings
xcrun xcstringstool print Darwin/InfoPlist.xcstrings

# Or directly (if `xcrun` isn't on PATH):
/Applications/Xcode.app/Contents/Developer/usr/bin/xcstringstool print PATH/Localizable.xcstrings
```

`xcstringstool print` parses the JSON, validates the schema, and prints a human-readable summary by language. A non-zero exit is a malformed catalog — usually one of:

- A misplaced comma.
- An unclosed brace.
- An unknown `state` value (must be exactly `new` / `needs_review` / `translated`).
- A `stringUnit` missing either `state` or `value`.

The tool does **not** catch:

- Mismatched format-specifier counts between source and translations.
- Wrong format specifier types (`%@` where the source has `%lld`).
- Smart quotes in `value` fields where the surrounding code expects ASCII.

For those, the build catches them at compile time on iOS, but Skip's Kotlin transpiler can pass through subtly wrong format strings that crash at runtime on Android. Manually inspect any string with format specifiers after editing.

## JSON formatting hygiene

Xcode writes the JSON with a **space between key and colon-value separator**: `"value" : "Einstellungen"`, not `"value": "Einstellungen"`. Preserve this style when editing programmatically — diffs stay clean and Xcode won't reformat the whole file on its next interactive open.

Common pitfalls:

- **Trailing commas** — JSON forbids them; `xcstringstool print` will fail.
- **Smart quotes** — pasting from a translation tool can introduce U+201C / U+201D. Catalogs use straight ASCII quotes for the JSON itself; smart quotes are fine inside `value` strings if the target language uses them, but the JSON delimiters must stay ASCII.
- **Trailing whitespace** in `value` fields — invisible in most editors but breaks exact-match assertions in Maestro flows.
- **UTF-8 BOM** — some Windows tools prepend a BOM (`\xEF\xBB\xBF`) when saving. `xcstringstool print` accepts it; Xcode's catalog editor reformats it away on next open.
