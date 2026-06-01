# Switching device locale for screenshot flows

Localised Maestro screenshots need the app to be running in the target locale. iOS and Android expose different mechanisms for changing it.

## iOS — set the simulator's global locale

iOS does not have per-app locale on the Simulator the way Android 13+ does. Switch the whole device, then restart SpringBoard so the new defaults take effect:

```bash
SIM=214FA103-3284-4C45-A567-52A9721EC64A   # your booted iPhone Air UDID

# Switch to French
xcrun simctl spawn $SIM defaults write "Apple Global Domain" AppleLanguages "(fr)"
xcrun simctl spawn $SIM defaults write "Apple Global Domain" AppleLocale "fr_FR"
xcrun simctl spawn $SIM launchctl stop com.apple.SpringBoard
xcrun simctl launch $SIM dev.skip.todo

# Switch back to English when done
xcrun simctl spawn $SIM defaults write "Apple Global Domain" AppleLanguages "(en)"
xcrun simctl spawn $SIM defaults write "Apple Global Domain" AppleLocale "en_US"
xcrun simctl spawn $SIM launchctl stop com.apple.SpringBoard
```

Restarting SpringBoard is much faster than a full shutdown + boot, and it's enough to pick up the new locale at the next app launch. `launchctl stop` exits SpringBoard, which `launchd` immediately respawns; the simulator stays running. (A full `simctl shutdown` + `simctl boot` cycle also works if SpringBoard restart misbehaves, but you almost never need it.)

Common locale settings:

| Language | AppleLanguages | AppleLocale |
|---|---|---|
| English (US) | `(en)` | `en_US` |
| French (France) | `(fr)` | `fr_FR` |
| Simplified Chinese | `(zh-Hans)` | `zh_Hans_CN` |
| Traditional Chinese | `(zh-Hant)` | `zh_Hant_TW` |
| German | `(de)` | `de_DE` |
| Japanese | `(ja)` | `ja_JP` |
| Spanish (Spain) | `(es)` | `es_ES` |
| Portuguese (Brazil) | `(pt-BR)` | `pt_BR` |

## Android — set the *per-app* locale

Android 13+ exposes `cmd locale set-app-locales`, which changes the locale for one app without disturbing the rest of the device:

```bash
SER=emulator-5554

# French for the TODO app
adb -s $SER shell cmd locale set-app-locales dev.skip.todo --locales fr-FR
adb -s $SER shell am force-stop dev.skip.todo
adb -s $SER shell am start -n dev.skip.todo/todo.app.MainActivity

# Simplified Chinese
adb -s $SER shell cmd locale set-app-locales dev.skip.todo --locales zh-Hans-CN

# Back to English (clears the per-app override)
adb -s $SER shell cmd locale set-app-locales dev.skip.todo --locales en-US

# Inspect the current setting
adb -s $SER shell cmd locale get-app-locales dev.skip.todo
```

Setting the system-wide locale with `setprop persist.sys.locale` requires root and is not available on production emulator images — always use the per-app form instead.

The `am force-stop` + `am start -n` cycle is needed to pick up the new locale; without it, the Activity keeps the old `Configuration`.

The `start -n` form takes `<package>/<activity-class>`. Skip-generated apps' activity class is `<package>.MainActivity` by default — you can confirm with `adb shell cmd package resolve-activity --brief <package>` if you've customised it.

## A locale-aware walkthrough flow

Use Maestro's `-e KEY=value` env-substitution to share one flow per language while keeping per-locale screenshot paths:

```yaml
# .maestro/20-walkthrough-en.yaml
appId: dev.skip.todo
---
- launchApp
- assertVisible: "My Tasks"
- takeScreenshot: ".maestro/screenshots/walk-${MAESTRO_OUT_SUFFIX}-1-empty"
- tapOn: { id: "field.newTask" }
- inputText: "Buy fresh bread"
- tapOn: { id: "button.add" }
- takeScreenshot: ".maestro/screenshots/walk-${MAESTRO_OUT_SUFFIX}-2-list"
```

```bash
maestro --device $SIM           test -e MAESTRO_OUT_SUFFIX=ios-en      .maestro/20-walkthrough-en.yaml
maestro --device emulator-5554  test -e MAESTRO_OUT_SUFFIX=android-en  .maestro/20-walkthrough-en.yaml
```

One walkthrough file per language (asserts on the language's strings) plus per-platform suffix gives you `walk-ios-en-1-empty.png` / `walk-android-en-1-empty.png` / `walk-ios-fr-1-empty.png` etc., all in one directory. Those files plug directly into:

- `Darwin/fastlane/screenshots/<locale>/` for App Store Connect.
- `Android/fastlane/metadata/android/<locale>/images/phoneScreenshots/` for Google Play.

For the App Store / Play Store locale-code mapping (`fr` → `fr-FR` on both, `zh-Hans` → `zh-Hans` on App Store but `zh-CN` on Play), see the [skip-localization](../../../../skip-app-design/skills/skip-localization/SKILL.md) skill.

## Typical end-to-end localised screenshot workflow

```bash
# 1. Switch both platforms to French
SIM=214FA103-3284-4C45-A567-52A9721EC64A; SER=emulator-5554
xcrun simctl spawn $SIM defaults write "Apple Global Domain" AppleLanguages "(fr)"
xcrun simctl spawn $SIM defaults write "Apple Global Domain" AppleLocale "fr_FR"
xcrun simctl spawn $SIM launchctl stop com.apple.SpringBoard
adb -s $SER shell cmd locale set-app-locales dev.skip.todo --locales fr-FR

# 2. Relaunch
xcrun simctl launch $SIM dev.skip.todo
adb -s $SER shell am force-stop dev.skip.todo
adb -s $SER shell am start -n dev.skip.todo/dev.skip.todo.MainActivity

# 3. Run the French walkthrough on both, with per-platform suffix
maestro --device $SIM  test -e MAESTRO_OUT_SUFFIX=ios-fr      .maestro/21-walkthrough-fr.yaml
maestro --device $SER  test -e MAESTRO_OUT_SUFFIX=android-fr  .maestro/21-walkthrough-fr.yaml

# 4. Repeat for each supported language, then switch back to English
```

Wrap this in a shell script and you have a one-button regeneration of every fastlane screenshot.
