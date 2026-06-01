# SwiftUI APIs not yet implemented in SkipUI

This file catalogs SwiftUI APIs marked `@available(*, unavailable)` in the SkipUI source tree. If you reach for one of these in a Skip Lite or Skip Fuse project, you get the build error:

> This API is not yet available in Skip. Consider placing it within a `#if !SKIP` block. You can file an issue against the owning library at `https://github.com/skiptools`.

The source of truth is `/opt/src/github/skiptools/skip-ui/Sources/SkipUI/SkipUI/` (on disk if the repo is checked out, or `https://github.com/skiptools/skip-ui` upstream). Counts and line numbers below are as of the audit; the list grows and shrinks as SkipUI gains coverage. **When you hit an unavailable API, check the source file before assuming this catalog is current** — Skip is adding implementations continuously.

## Quick lookup

| API surface | File | Approximate count |
|---|---|---|
| View modifiers (containers, layout, hover, focus, dialogs, effects, accessibility, gestures) | `View/AdditionalViewModifiers.swift` | 63+ |
| ScrollView features (edge effects, paging, indicators, anchors) | `Components/ScrollView.swift` | 16+ |
| Toolbar placements / TabContent modifiers / TabBarMinimizeBehavior | `Components/TabView.swift` | 33+ |
| Table styling / cells / columns | `Components/Table.swift` | 23+ |
| Navigation predicates / split-view destinations | `Components/Navigation.swift` | 11+ |
| List(selection:) and related selection APIs | `Components/List.swift` | 14+ |
| Image initialisers (resizable with capInsets / resizingMode, other variants) | `Components/Image.swift` | 5+ |
| Form initialisers | `Components/Form.swift` | 2+ |
| Picker styling overloads | `Components/Picker.swift` | 2+ |
| DatePicker overloads | `Components/DatePicker.swift` | 2+ |

The most-common categories you'll bump into are in the next sections.

## View modifiers — `View/AdditionalViewModifiers.swift`

This single file holds the majority of unavailable APIs. The ones most likely to surface in agent-written code:

### Hit-testing and shape

- `contentShape(_:eoFill:)` — define the hit-test shape of a view.
- `contentShape(_:_:eoFill:)` — content shape with a kind specifier.
- `containerShape(_:)`.

**Replacement.** For the `Button` hit-target use case, an `HStack` with a trailing `Spacer()` produces a row-wide tap target without `.contentShape(Rectangle())`. For arbitrary hit-shapes, no current replacement; wrap in `#if !SKIP`.

### Layout and sizing

- `containerBackground(_:for:)` and `containerBackground(for:alignment:content:)`.
- `containerRelativeFrame(_:alignment:)` and the multi-arg overloads.
- `coordinateSpace(_:)` for named coordinate spaces.
- `layoutPriority(_:)`.
- `layoutDirectionBehavior(_:)`.
- `fixedSize()` (both overloads).

**Replacement.** `.frame(width:height:)` and `.padding()` cover most of what `containerBackground` and `containerRelativeFrame` are used for. For `layoutPriority`, restructure with `Spacer()` or explicit frames.

### Hover and focus (mostly desktop / iPadOS / visionOS)

- `hoverEffect(_:isEnabled:)`, `hoverEffectDisabled(_:)`, `defaultHoverEffect(_:)`.
- `onHover(perform:)`, `onContinuousHover(coordinateSpace:perform:)`.
- `controlSize(_:)`.
- `keyboardShortcut(_:modifiers:)` (3 overloads).

**Replacement.** None — Android has no equivalent for hover state in the desktop sense. Wrap in `#if !SKIP` or `#if os(iOS)` if you only need them on Apple platforms.

### Dialogs, inspectors, popovers

- `dialogSuppressionToggle(_:isSuppressed:)` (5 overloads).
- `inspector(isPresented:content:)` and `inspectorColumnWidth(...)` (3 overloads).
- `popover(item:...)` and `popover(isPresented:...)`.
- `renameAction(_:)` (2 overloads).
- `fileMover(isPresented:file:onCompletion:)` (4 overloads).

**Replacement.** `.sheet`, `.fullScreenCover`, or `.confirmationDialog` cover most popover-shaped patterns. No analogue for inspectors on Android.

### Effects and styling

- `foregroundStyle(_:_:)` and `foregroundStyle(_:_:_:)` (multi-layer styles).
- `tint(_:)` with a `ShapeStyle`.
- `symbolEffectsRemoved(_:)`.
- `matchedGeometryEffect(id:in:...)`.
- `transformEffect(_:)` (CGAffineTransform).
- `projectionEffect(_:)`.
- `transformEnvironment(_:transform:)`.

**Replacement.** Single-argument `foregroundStyle(_:)` and `tint(_:)` (color-only) work. No matched-geometry-effect substitute on Android — restructure animations to avoid it.

### Accessibility (desktop-leaning)

- `help(_:)` (4 overloads).
- `typeSelectEquivalent(_:)` (4 overloads).
- `headerProminence(_:)`.
- `accessibilityInputLabels(_:)` (Compose has no analogue for now).

**Replacement.** Skip implements the core accessibility surface (`accessibilityIdentifier`, `accessibilityLabel`, `accessibilityValue`, `accessibilityHint`) which is what's load-bearing for Maestro tests and TalkBack/VoiceOver labels.

### Gestures and system events

- `defersSystemGestures(on:)`.
- `handlesExternalEvents(...)`.
- `interactionActivityTrackingTag(_:)`.

### Drag and drop, context menus, swipe actions

- `dropDestination(for:action:)` (View and TabContent variants).
- `contextMenu(forSelectionType:menu:primaryAction:)` (selection-based).
- `swipeActions(...)` on TabContent.
- `draggable(_:)` on TabContent.

**Replacement.** `.swipeActions(edge:allowsFullSwipe:content:)` *on List rows* is implemented. The variants on TabContent are not.

### System UI

- `persistentSystemOverlays(_:)`.
- `sectionActions(content:)`.
- `selectionDisabled(_:)`.
- `badge(_:LocalizedStringResource)` — the `Int`, `String`, and `Text` overloads of `badge` are implemented.

## ScrollView — `Components/ScrollView.swift`

- `scrollBounceBehavior(_:axes:)`.
- `scrollClipDisabled(_:)`.
- `scrollEdgeEffectStyle(_:for:)` (2 overloads).
- `scrollIndicatorsFlash(onAppear:)` and `scrollIndicatorsFlash(trigger:)`.
- `scrollTarget(isEnabled:)`.
- `ScrollPosition` initialiser.
- `PinnedScrollableViews.sectionHeaders` and `.sectionFooters`.
- `PagingScrollTargetBehavior` initialiser and `.paging`.
- `ViewAlignedScrollTargetBehavior` initialisers with `anchor`.

**Replacement.** `scrollIndicators(_:axes:)` and basic `ScrollViewReader` are implemented. For paging behaviour, use a `TabView(.page)` or a `LazyHStack`/`LazyVStack` inside a `ScrollView` with manual position tracking.

## TabView — `Components/TabView.swift`

The `TabView` itself works; many of its modifiers and styles do not:

### TabContent (the per-tab item closure) modifiers

The full set of accessibility overloads on TabContent are unavailable (`accessibilityValue`, `accessibilityLabel`, `accessibilityHint` — 4 overloads each, accepting `Text`, `LocalizedStringKey`, and `String` variants). Most projects don't notice because the View-level overloads work — but if you nest accessibility on a TabContent specifically, those are out.

### Customisation and placement

- `customizationBehavior(_:for:)`, `customizationID(_:)`.
- `tabPlacement(_:)`, `defaultVisibility(_:for:)`, `defaultAdaptableTabBarPlacement(_:)`.
- `tabViewBottomAccessory(content:)`.

### Interaction

- `sectionActions(content:)`, `dropDestination(...)`, `springLoadingBehavior(_:)`, `draggable(_:)`, `popover(...)`, `contextMenu(...)`, `swipeActions(...)` on TabContent.
- `accessibilityIdentifier(_:)` on TabContent specifically — use it on the contained view instead.

### TabBarMinimizeBehavior

- `.onScrollDown` and `.onScrollUp` values are unavailable.

**Replacement.** Apply `accessibilityIdentifier` to the root view inside each tab, not to the TabContent wrapper. For tab placement, the default works on both platforms; don't reach for `tabPlacement(_:)`.

## Table — `Components/Table.swift`

Most cell, column, and row styling APIs are unavailable. Tables work for simple read-only data presentation; advanced selection / column resizing / sorting customisation is not.

## Navigation — `Components/Navigation.swift`

`NavigationStack` and `NavigationLink` work. The unavailable ones are:

- `NavigationSplitView` (preferred-compact-column, column-width modifiers).
- Predicate-based destinations (`navigationDestination(for:Predicate)`).
- Multi-column split-view APIs.

Stick to `NavigationStack` + `navigationDestination(for:Hashable.Type, destination:)`.

## List — `Components/List.swift`

- `List(selection:content:)` — selection binding is unavailable.

If you need multi-select, manage it via your own `@State` and a custom tap handler on each row.

## Image — `Components/Image.swift`

- `resizable(capInsets:resizingMode:)` and `resizable(resizingMode:)` — both are unavailable.

Use `resizable()` (no args) followed by `.scaledToFit()` / `.aspectRatio(_:contentMode:)` / `.frame(width:height:)`.

## Form, Picker, DatePicker, etc.

Each has a small number of unavailable overloads — usually the more specialised initialisers. The simple forms (`Form { ... }`, `Picker("Label", selection: $sel) { ... }`, `DatePicker("Label", selection: $date)`) work.

For `Picker`, the unavailable overloads are around custom styles. For `DatePicker`, the unavailable initialisers are around range constraints — see [`patterns.md`](patterns.md) for the workaround using Fuse bridging.

## What to do when you hit an unavailable API

1. **Confirm against the source.** Open the relevant file in `skip-ui/Sources/SkipUI/SkipUI/`. Sometimes the API has been added since this catalog was written; sometimes another overload of the same API is implemented.
2. **Look for a near substitute** in the same file. SkipUI often implements 80% of an API surface but stops short of the most complex overloads.
3. **Wrap in `#if !SKIP`** if the API is iOS-only acceptable for your use case. This keeps the iOS build correct and produces a no-op on Android.
4. **File an issue** at `https://github.com/skiptools/skip-ui` if it's an API a typical Skip user would need. Issues filed with concrete use cases get prioritised.

## See also

- [`components.md`](components.md) — the **supported** components and modifiers, for the inverse view.
- [`compose-customization.md`](compose-customization.md) — when you can't avoid an unavailable API on Android, dropping into raw Compose via `ComposeView` is the fallback.
- [`patterns.md`](patterns.md) — common cross-platform UI patterns that route around unavailable APIs.
