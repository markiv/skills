# Hardcoded SF Symbol → Material Icon compatibility map

SkipUI hardcodes a compatibility map from a small set of SF Symbol names to Jetpack Compose Material Icons. If the name you pass to `systemImage:` is on this list, it renders correctly on both iOS and Android. Every other SF Symbol falls through to a default branch that renders nothing or a generic placeholder on Android, while still rendering correctly on iOS.

**Treat this list as closed.** If your symbol isn't here, ship a `.symbolset` instead — see the main [skip-icons](../SKILL.md) skill for the workflow.

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

The mapping is implemented as a `switch` in `SkipUI/Components/Image.swift` in the [`skip-ui` repository](https://github.com/skiptools/skip-ui). When in doubt about whether a particular SF Symbol name is mapped, grep that file. The list grows occasionally upstream — if a glyph you need has been added, you can fall back to `systemImage:` for that one. Until then, treat anything not enumerated above as Android-unsupported.

## Why this list is small

The Material Icons set used by Compose has ~150 base icons (with outlined/filled/rounded variants). SF Symbols has thousands. There's no canonical 1-to-1 mapping, so SkipUI ships the obvious matches and leaves the rest to user-provided `.symbolset` resources, where the same SVG renders on both platforms by design.

If you find yourself reaching for `systemImage:` more than a handful of times in an app, write a small wrapper or constant table that resolves your symbol names to either the SF Symbol (when mapped) or a local `Image("name", bundle: .module)` (when not) — and use the wrapper everywhere. It removes the cognitive load of remembering which 50 names work.
