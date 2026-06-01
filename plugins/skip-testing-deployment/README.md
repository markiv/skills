# skip-testing-deployment

Test, export, sign, and distribute [Skip](https://skip.dev) apps for the App Store and Google Play.

## Skills

- **skip-testing** — XCTest and Swift Testing parity testing, Robolectric, instrumented tests, test harness configuration
- **skip-ui-automation** — Launching apps on iOS Simulator + Android emulator with `skip app launch`, driving them with Maestro UI tests, and the `.accessibilityIdentifier()` convention that makes flows resilient
- **skip-deployment** — Exporting with `skip export`, iOS and Android signing, ProGuard, app store submission, CI/CD

## When to use

Use this plugin when:
- Writing tests that run on both iOS and Android
- Running parity tests with `swift test`
- Launching the app on both simulators side-by-side with `skip app launch`
- Driving the running app with Maestro flow YAMLs
- Adding `.accessibilityIdentifier(...)` to views so they're stable test targets
- Testing on Android emulator or device
- Building release artifacts with `skip export`
- Signing and submitting to the App Store or Google Play
- Setting up CI/CD for a Skip project

## License

Apache 2.0
