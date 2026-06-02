---
name: building-skip-ui
description: Building SwiftUI views in Skip projects that work on both iOS and Android. Covers supported components, navigation, state management, Jetpack Compose customization, Material 3 theming, the hard rule against `Image(systemName:)` / `Label(_, systemImage:)` (use `.symbolset` resources via `Image("name", bundle: .module)` instead), and the localization-correct `Label { Text } icon: { Image }` shape.
---

# Building Skip UI

SkipUI converts SwiftUI into Jetpack Compose for Android. Use standard SwiftUI patterns — SkipUI handles the translation. Not all SwiftUI APIs are available; this skill covers what works and how to customize the Android rendering.

## Project Setup

A Skip app uses standard SwiftUI with the SkipUI framework:

```swift
// Package.swift dependencies
.package(url: "https://source.skip.tools/skip.git", from: "1.2.0"),
.package(url: "https://source.skip.tools/skip-ui.git", from: "1.0.0"),
.package(url: "https://source.skip.tools/skip-foundation.git", from: "1.0.0"),
```

Create projects with:
```bash
skip init --appid=com.example.myapp my-app MyApp          # Lite
skip init --native-app --appid=com.example.myapp my-app MyApp  # Fuse
```

## Supported Components

Consult the reference files for detailed tables:
- [`references/components.md`](references/components.md) — full component support table
- [`references/compose-customization.md`](references/compose-customization.md) — Compose and Material 3 customisation
- [`references/patterns.md`](references/patterns.md) — common cross-platform UI patterns
- [`references/unimplemented-apis.md`](references/unimplemented-apis.md) — the catalog of SwiftUI APIs marked `@available(*, unavailable)` in SkipUI, organised by file and category. Consult this when a build error says "This API is not yet available in Skip."

### Layout (Full Support)
`VStack`, `HStack`, `ZStack`, `Group`, `Spacer`, `GeometryReader`, `ScrollView`, `LazyVStack`, `LazyHStack`, `LazyVGrid`, `LazyHGrid`, `Grid`, `GridRow`, `Form`, `Section`

### Controls (Full Support)
`Button`, `Text`, `Label`, `TextField`, `SecureField`, `TextEditor`, `Toggle`, `Slider`, `Stepper`, `Picker`, `ProgressView`, `Link`, `Menu`, `ShareLink`

### Navigation (Full Support)
`NavigationStack`, `NavigationLink`, `NavigationSplitView`, `TabView`, `.sheet`, `.fullScreenCover`, `.alert`, `.confirmationDialog`, `.toolbar`, `.searchable`

### Media (Full Support)
`Image`, `AsyncImage`, `Color`, `LinearGradient`, `Circle`, `Capsule`, `Rectangle`, `RoundedRectangle`, `Path`

### State Management (Full Support)
`@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`, `@Environment`, `@Bindable`, `@Observable` (via SkipModel)

### Limited or Unavailable
- `DatePicker`: Medium support (date range constraints need Fuse bridging)
- `@AppStorage`: Optional values not supported
- `Canvas`: Basic drawing only
- `UIViewRepresentable`: Not available — use `#if os(iOS)` guards
- `matchedGeometryEffect`: Not available
- `@FetchRequest`: Not available — use SkipSQL or SkipFirebase
- Custom `Shape` with complex `AnimatableData`: Not available
- `.contentShape(Rectangle())`: Not implemented yet — emits *"This API is not yet available in Skip"* at transpile time. Drop the modifier or wrap in `#if !SKIP`. The Button hit-target it usually expands is large enough on its own when the label is an HStack containing a `Spacer()`.
- `ToolbarItem(placement: .topBarTrailing) { ... }`: `topBarTrailing` is `unavailable` on macOS. Skip's `Package.swift` declares `macOS(.v14)` alongside `iOS(.v17)`, so any Skip target that builds for macOS will fail with "topBarTrailing is unavailable in macOS". Use `.primaryAction` instead — it has the same visual placement on iOS, is available on macOS, and transpiles to a Compose `TopAppBar` action slot:
   ```swift
   .toolbar {
       ToolbarItem(placement: .primaryAction) {   // not .topBarTrailing
           Menu { ... } label: { Image("more_vert", bundle: .module) }
       }
   }
   ```
   (`Image(systemName: "ellipsis")` would happen to render on Android because `ellipsis` is one of the ~50 SF Symbol names SkipUI maps internally — but you should still ship a `.symbolset`. See [skip-icons](../skip-icons/SKILL.md) for why.)
- `Array.remove(atOffsets: IndexSet)`: this is provided by **SwiftUI's** extension on `RangeReplaceableCollection`, not by Foundation. A pure-model module that doesn't `import SwiftUI` will not have it. Either import SwiftUI in the model, or expose a plain `func remove(at: [Int])` and let the view convert `Array(offsets)`:
   ```swift
   // In TodoAppModel/ViewModel.swift — no SwiftUI import needed
   public func remove(at indices: [Int]) {
       for index in indices.sorted(by: >) where index < items.count {
           items.remove(at: index)
       }
   }
   ```
   ```swift
   // In TodoApp/ContentView.swift — SwiftUI is imported, IndexSet → [Int]
   .onDelete { offsets in
       viewModel.remove(at: Array(offsets))
   }
   ```

## Cross-Platform View Pattern

```swift
import SwiftUI

struct ContentView: View {
    @State private var count = 0

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                Text("Count: \(count)")
                    .font(.title)
                Button("Increment") {
                    count += 1
                }
                .buttonStyle(.borderedProminent)
            }
            .navigationTitle("My App")
        }
    }
}
```

## Observable Models

```swift
import Foundation
import Observation

@Observable final class ViewModel {
    var items: [String] = []
    var isLoading = false

    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        // Fetch data...
    }
}
```

## Platform-Specific Code

```swift
var body: some View {
    Text("Hello")
        #if os(Android)
        .font(.system(size: 18))
        #endif
}
```

## Compose Customization

Embed raw Jetpack Compose in `#if SKIP` blocks:

```swift
#if SKIP
ComposeView { context in
    androidx.compose.material3.FloatingActionButton(
        onClick: { handleTap() }
    ) {
        androidx.compose.material3.Icon(
            imageVector: androidx.compose.material.icons.Icons.Default.Add,
            contentDescription: "Add"
        )
    }
}
#endif
```

Apply Compose modifiers to SwiftUI views:

```swift
Text("Custom")
    #if SKIP
    .composeModifier { modifier in modifier.testTag("customText") }
    #endif
```

Customize Material 3 theming:

```swift
ContentView()
    .material3ColorScheme(ColorScheme(primary: Color.blue, onPrimary: Color.white))
```

## Icons

**The hard rule for Skip projects: never use `Image(systemName:)` or `Label(_, systemImage:)`. Always ship a `.symbolset` resource and call `Image("name", bundle: .module)`.**

SF Symbols are an Apple-only catalogue with no Android equivalent. Almost every `systemName:` call you write will render a blank on Android — the iOS build resolves the name against Apple's catalogue, the Android build has nothing to resolve it against. Even for the ~50 names SkipUI hardcodes a Material Icon substitute for, the rendering isn't pixel-identical to the SF Symbol you asked for, and the hardcoded map can change between SkipUI releases. The reliable answer in all cases is a single `.symbolset` that ships to both platforms and renders identically.

Download Material Symbols from `https://fonts.google.com/icons` in **Apple symbolset format** (not raw SVG), drop into `Sources/<Module>/Resources/Icons.xcassets/<name>.symbolset/`, and reference with `Image("name", bundle: .module)`:

```swift
// ❌ Will render blank on Android.
Label("Bookmark", systemImage: "bookmark.fill")
Image(systemName: "checklist")

// ✅ Works on both platforms; localisation-correct; predictable.
Label {
    Text("Bookmark", bundle: .module, comment: "bookmark menu label")
} icon: {
    Image("bookmark", bundle: .module)
}
```

The trailing-closure `Label { Text } icon: { Image }` form is the canonical shape — it makes the label translatable (the String Catalog extractor sees the `Text(_, bundle: .module, comment: …)`) and the icon cross-platform (the `.symbolset` resource ships to both platforms via the same module bundle).

For the full workflow — picking glyphs, downloading the Apple-format SVG, the `.symbolset` directory layout with `Contents.json`, app launcher icons via `skip icon`, and Android-side rendering quirks — see the [skip-icons](../skip-icons/SKILL.md) skill.

## Localization

User-facing strings should be created with the bundle-aware `Text` constructor so they're extracted into the module's `Resources/Localizable.xcstrings` String Catalog and translated for both platforms:

```swift
Text("Settings", bundle: .module, comment: "settings sheet title")
Label {
    Text("Block Ads", bundle: .module, comment: "settings toggle for blocking ads")
} icon: {
    Image("shield", bundle: .module)
}
```

Skip transpiles the catalog to Android string resources at build time, so a single source produces translated UI on both iOS and Android. For the full translation workflow — catalog schema, translation states, `xcstringstool` validation, byte-budgeted store metadata, and the App Store / Google Play locale mapping — see the [skip-localization](../skip-localization/SKILL.md) skill.

## References

Consult these resources for detailed guidance:
- ./references/components.md -- Complete component and modifier support tables
- ./references/compose-customization.md -- ComposeView, composeModifier, Material 3
- ./references/patterns.md -- Tab navigation, lists, async loading, forms
