# skip-app-design

Build cross-platform iOS and Android apps with [Skip](https://skip.dev) using SwiftUI, optional frameworks, and Skip Lite transpilation rules.

## Skills

- **skip-project-creation** — Create new Skip apps and libraries with `skip init`, add optional framework dependencies with Package.swift entries and starter template code
- **building-skip-ui** — SwiftUI component support on Android via SkipUI, Compose customization, cross-platform view patterns, navigation, state management
- **skip-lite-transpilation** — Critical rules for Skip Lite (transpiled) mode: integer overflow, type comparison errors, unsupported patterns, Kotlin interop
- **skip-frameworks** — Integrating optional Skip frameworks: Firebase, SQL, Keychain, WebView, AV, Device sensors, Lottie, FFI, and more
- **skip-icons** — Working with icons. The hard rule: never use `Image(systemName:)` or `Label(_, systemImage:)`. Instead, download Material Symbols from `fonts.google.com/icons` in Apple symbolset format, drop them under `Sources/<Module>/Resources/Icons.xcassets/`, and render with `Image("name", bundle: .module)` inside the `Label { Text } icon: { Image }` form. The same SVG renders identically on both platforms.
- **skip-localization** — Translating in-app strings via `Localizable.xcstrings`, iOS `Info.plist` usage descriptions via `InfoPlist.xcstrings`, and App Store / Google Play metadata under `fastlane/`. The `Text("…", bundle: .module, comment: "…")` extraction convention, translation states, byte-budgeted store metadata, validation with `xcstringstool` and `skip verify`.

## When to use

Use this plugin when:
- Creating a new Skip app or library project
- Building SwiftUI views that need to work on both iOS and Android
- Writing Swift code in a Skip Lite (transpiled) project
- Adding optional frameworks like Firebase, SQLite, or secure storage
- Customizing Android rendering with Jetpack Compose
- Adding icons that need to render on both platforms
- Adding or updating translations for in-app strings, Info.plist permission prompts, or store metadata

## License

Apache 2.0
