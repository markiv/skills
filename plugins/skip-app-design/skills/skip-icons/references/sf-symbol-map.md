# Hardcoded SF Symbol → Material Icon compatibility map

> **This file is descriptive, not prescriptive.** It documents what SkipUI does internally when you pass a name to `Image(systemName:)` or `Label(_, systemImage:)`. **Do not use it as a "safe to use" list.** Even for the names below, the right pattern is to ship a `.symbolset` resource and call `Image("name", bundle: .module)`. See the main [skip-icons](../SKILL.md) skill for the hard rule and the symbolset workflow.

SkipUI hardcodes a compatibility map from a small set of SF Symbol names to Jetpack Compose Material Icons. If the name you pass to `systemImage:` happens to be on this list, the Android build will render *some* glyph — usually a Material Icon that looks roughly similar to the requested SF Symbol but is not pixel-identical. Every other SF Symbol falls through to a default branch that renders nothing or a generic placeholder on Android.

This list exists for two reasons: to document SkipUI's runtime behaviour, and to help you diagnose what's happening when you encounter `Image(systemName:)` calls in code you're auditing. It is not a recommendation. Three reasons you should still avoid `systemName:` even for these names:

1. **The list is small and hard to remember.** Picking the right name from a 50-entry mental table is error-prone — especially for an agent who can pattern-match to a plausible SF Symbol like `checklist` or `bookmark.fill` that isn't here.
2. **The rendering isn't identical to iOS.** The replacement Material Icon may differ in shape, weight, or visual centre from the requested SF Symbol.
3. **The map is not a stable API.** SkipUI may add or remove entries between releases; a name that works today might not tomorrow.

For every icon, ship a `.symbolset`. The cost is one ~10 KB SVG; the benefit is identical, predictable rendering on both platforms forever.

## The full hardcoded map

| `systemImage:` name                | Renders on Android as           |
|------------------------------------|---------------------------------|
| `house`, `house.fill`              | `Icons.Outlined.Home`, `Icons.Filled.Home` |
| `gearshape`, `gearshape.fill`      | `Icons.Outlined.Settings`, `Icons.Filled.Settings` |
| `magnifyingglass`                  | `Icons.Outlined.Search`         |
| `plus`, `plus.circle.fill`         | `Icons.Outlined.Add`, `Icons.Outlined.AddCircle` |
| `xmark`                            | `Icons.Outlined.Clear`          |
| `checkmark`, `checkmark.circle`, `checkmark.circle.fill` | `Icons.Outlined.Check`, `Icons.Outlined.CheckCircle`, `Icons.Filled.CheckCircle` |
| `arrow.left`, `arrow.forward`, `arrow.forward.square` | `Icons.Outlined.ArrowBack`, `Icons.Outlined.ArrowForward`, `Icons.Outlined.ExitToApp` |
| `arrowtriangle.down.fill`          | `Icons.Outlined.ArrowDropDown`  |
| `arrow.clockwise.circle`           | `Icons.Outlined.Refresh`        |
| `chevron.down`/`up`/`left`/`right` | `Icons.Outlined.KeyboardArrow*` |
| `pencil`                           | `Icons.Outlined.Create`         |
| `trash`, `trash.fill`              | `Icons.Outlined.Delete`, `Icons.Filled.Delete` |
| `calendar`                         | `Icons.Outlined.DateRange`      |
| `envelope`, `envelope.fill`        | `Icons.Outlined.Email`, `Icons.Filled.Email` |
| `face.smiling`                     | `Icons.Outlined.Face`           |
| `heart`, `heart.fill`              | `Icons.Outlined.FavoriteBorder`, `Icons.Outlined.Favorite` |
| `info.circle`, `info.circle.fill`  | `Icons.Outlined.Info`, `Icons.Filled.Info` |
| `list.bullet`                      | `Icons.Outlined.List`           |
| `location`, `location.fill`        | `Icons.Outlined.LocationOn`, `Icons.Filled.LocationOn` |
| `lock`, `lock.fill`                | `Icons.Outlined.Lock`, `Icons.Filled.Lock` |
| `line.3.horizontal`                | `Icons.Outlined.Menu`           |
| `ellipsis`                         | `Icons.Outlined.MoreVert`       |
| `bell`, `bell.fill`                | `Icons.Outlined.Notifications`, `Icons.Filled.Notifications` |
| `person`, `person.fill`, `person.crop.circle`, `person.crop.circle.fill`, `person.crop.square.fill` | various `Icons.Outlined/Filled.Account*`, `Person` |
| `mappin.circle`, `mappin.circle.fill` | `Icons.Outlined.Place`, `Icons.Filled.Place` |
| `phone`, `phone.fill`              | `Icons.Outlined.Call`, `Icons.Filled.Call` |
| `play`, `play.fill`                | `Icons.Outlined.PlayArrow`, `Icons.Filled.PlayArrow` |
| `paperplane`, `paperplane.fill`    | `Icons.Outlined.Send`, `Icons.Filled.Send` |
| `square.and.arrow.up`, `square.and.arrow.up.fill` | `Icons.Outlined.Share`, `Icons.Filled.Share` |
| `cart`, `cart.fill`                | `Icons.Outlined.ShoppingCart`, `Icons.Filled.ShoppingCart` |
| `star.fill`                        | `Icons.Filled.Star`             |
| `hand.thumbsup`, `hand.thumbsup.fill` | `Icons.Outlined.ThumbUp`, `Icons.Filled.ThumbUp` |
| `exclamationmark.triangle`, `exclamationmark.triangle.fill` | `Icons.Outlined.Warning`, `Icons.Filled.Warning` |
| `wrench`, `wrench.fill`            | `Icons.Outlined.Build`, `Icons.Filled.Build` |

## Source of truth

The mapping is implemented as a `switch` in `SkipUI/Components/Image.swift` in the [`skip-ui` repository](https://github.com/skiptools/skip-ui). If you need to confirm the current behaviour for a specific name, grep that file. The list changes between SkipUI releases; this table is a snapshot, not a guarantee.

## Why this list is small (and why the right answer is still `.symbolset`)

The Material Icons set used by Compose has ~150 base icons (with outlined/filled/rounded variants). SF Symbols has thousands. There's no canonical 1-to-1 mapping, so SkipUI ships the obvious matches and leaves the rest to user-provided `.symbolset` resources, where the same SVG renders on both platforms by design.

For the same reason, the right answer for *every* icon — not just the unmapped ones — is `.symbolset`. The mapped names render *something* on Android, but they render a Material Icon, not the SF Symbol you asked for. Visual differences in shape and weight accumulate across an app's UI and produce a noticeably different feel between platforms. Shipping a single Material Symbol on both platforms via `.symbolset` is the only way to get true visual parity.

If you encounter `systemImage:` / `systemName:` calls in an existing codebase, replace them all with `.symbolset` references — see the "Auditing an existing codebase" section in [skip-icons](../SKILL.md).
