---
name: skip-lite-transpilation
description: Swift-to-Kotlin transpilation rules and pitfalls for Skip Lite mode. Covers the high-impact bugs (silent integer overflow, `Double == Int` compile errors, `Foundation.sqrt`, `String.append(Character)`, `Array.remove(atOffsets:)` SwiftUI-only gotcha, variadic init issues, `String(describing:)` and Kotlin map iteration), the `MutableStruct`/`.sref()` value-semantics machinery and the `@nocopy` escape, generics limits and the `@inline(__always)` reified-generics escape, garbage-collection effects (`weak`/`unowned` ignored, `deinit` becomes Kotlin `finalize`), the Kotlish dialect for `#if SKIP` blocks, and the SKIP comment directives (INSERT/REPLACE/DECLARE/NOWARN/SYMBOLFILE/EXTERN/@nocopy/@nodispatch/@bridge). The full long-form rules are in references/swift-kotlin-rules.md. Use when writing or reviewing Swift code in a transpiled Skip project, or when a Kotlin compile error in `XCSkipTests.testSkipModule` points at one of the above patterns.
---

# Skip Lite Transpilation Rules

**CRITICAL**: `swift build` does NOT validate Kotlin transpilation. You MUST run `swift test` to catch transpilation errors.

This skill is for **Skip Lite** (transpiled). If the module's `Skip/skip.yml` contains `mode: 'native'`, you're in Skip Fuse, not Lite — most of the rules here (integer overflow, `Double == Int`, `Foundation.sqrt`, etc.) do not apply. See [skip-fuse](../skip-fuse/SKILL.md) for the Fuse workflow. The rules below still apply to any `#if SKIP` block inside a Fuse module (those blocks are transpiled to Kotlin even in Fuse mode).

## Impact: CRITICAL — Silent Data Corruption

### Integer Overflow

Swift `Int` is 64-bit. Kotlin `Int` is 32-bit (max ~2.1 billion). Intermediate arithmetic silently overflows in Kotlin.

```swift
// BAD: silently wrong on Android
let product = a * b  // if a * b > 2,147,483,647

// GOOD: use Int64 for large values
let product = Int64(a) * Int64(b)
```

## Impact: HIGH — Kotlin Compile Errors

### Double-to-Int Comparison

```swift
// BAD — Kotlin compile error
if someDouble == 0 { }

// GOOD
if someDouble == 0.0 { }
```

### Foundation-Qualified Math

```swift
// BAD — Kotlin compile error
let x = Foundation.sqrt(2.0)

// GOOD
let x = sqrt(2.0)
```

### String.append(Character)

```swift
// BAD — Kotlin compile error
result.append(someCharacter)

// GOOD
result += String(someCharacter)
```

### Array.remove(atOffsets: IndexSet) in non-SwiftUI modules

The `Array.remove(atOffsets:)` method that pairs with SwiftUI's `.onDelete { offsets in ... }` is an extension on `RangeReplaceableCollection` *provided by SwiftUI*, not by Foundation. A pure-data module that doesn't `import SwiftUI` won't have it — the compiler error is `No exact matches in call to instance method 'remove'`.

```swift
// BAD — in a model module that doesn't import SwiftUI
import Foundation
public func remove(at offsets: IndexSet) {
    items.remove(atOffsets: offsets)   // 'remove' not found
}

// GOOD — expose an Int-array API and let the view convert
public func remove(at indices: [Int]) {
    for index in indices.sorted(by: >) where index < items.count {
        items.remove(at: index)
    }
}
```

```swift
// Caller — SwiftUI is imported here, IndexSet → [Int]
.onDelete { offsets in
    viewModel.remove(at: Array(offsets))
}
```

The sort-descending matters: removing at higher indices first keeps the lower ones valid.

## Impact: HIGH — Wrong runtime behaviour, no compile error

These don't fail the Kotlin build, but the runtime behaviour differs from Swift in ways that bite.

### Strings are immutable and not `Collection`

Anywhere Swift mutates a `String` in place (`s.append(...)`, `s.removeLast()`, `s += ...` on a `var`), the transpiled Kotlin must produce a new string. And although Skip adds many `Collection` APIs to `String`, you cannot pass a `String` to code expecting `Collection<Character>` — Kotlin can't add a protocol to an existing type. See [`references/swift-kotlin-rules.md`](references/swift-kotlin-rules.md) for the full pattern and replacement examples.

### `String(describing:)` doesn't transpile

Use string interpolation instead:

```swift
// BAD — Kotlin sees String(describing = value), not a valid constructor
let s = String(describing: someValue)

// GOOD
let s = "\(someValue)"
```

### `weak` / `unowned` are silently ignored

Skip uses Kotlin's GC. `weak` and `unowned` compile but produce strong references on Kotlin. If you use `weak` to detect deallocation (`weakRef.object == nil` as a signal), the check will lie.

### `deinit` becomes Kotlin `finalize` and may not run

Indeterminate timing, may never execute. Don't put critical cleanup (file close, lock release, network disconnect) in `deinit` — use `defer`, `try`/`catch`, or an explicit `close()` method the caller invokes.

### Variadic initialisers don't transpile

```swift
// BAD
init(_ events: HapticEvent...) { self.events = events }

// GOOD
init(_ events: [HapticEvent]) { self.events = events }
```

## Structs and value semantics

Skip wraps mutable structs in a `MutableStruct` protocol and inserts `.sref()` calls automatically to preserve value semantics. You'll see `.sref()` calls all over your transpiled output; that's normal.

If a struct has `var` properties but is **never modified after construction**, you can opt out of the wrapping with the `// SKIP @nocopy` directive to avoid the copy overhead:

```swift
// SKIP @nocopy
struct Config {
    var apiURL: URL
    var timeout: TimeInterval
}
```

Two related rules:

- Prefer `final class` over `struct` for complex model types with many methods. `.sref()` insertion is fine for plain-data structs but can be subtle with method-heavy ones.
- Prefer named methods (`add()`, `multiply()`) over operator overloads for non-trivial types.

## Generics — limits and the `@inline(__always)` escape

Generic type parameters are erased at runtime. `is` and `as?` against generic types ignore the generics:

```swift
let xs: Any = [1, 2, 3]
if xs is [String] {       // True on Kotlin — generics erased
    let strings = xs as! [String]
    // strings.first crashes — they're actually Ints
}
```

Also unsupported:

- Static members of generic types that use the type's generic parameters.
- Inner types of generic outer types.

When you need the type available at runtime, annotate the function with `@inline(__always)` — Skip transpiles it to a Kotlin `inline fun <reified T>`:

```swift
@inline(__always)
func decode<T: Decodable>(_ data: Data) throws -> T {
    return try JSONDecoder().decode(T.self, from: data)
}
```

Use sparingly — every call site inlines the body.

## The Kotlish dialect and SKIP comment directives

Inside `#if SKIP` blocks, Skip relaxes its Swift parsing so you can write Kotlin-ish code that looks like Swift to Xcode but transpiles to real Kotlin. This is **Kotlish**.

When Kotlish can't express what you need (anonymous inner classes, `:` class references, property delegates), fall back to a **SKIP comment directive**:

- `// SKIP INSERT: <kotlin>` — inject literal Kotlin at this point. The Swift compiler ignores it.
- `// SKIP REPLACE: <kotlin>` — replace the next statement with literal Kotlin.
- `// SKIP DECLARE: <kotlin declaration>` — override only the declaration line; transpile the body normally.
- `// SKIP NOWARN` — silence a transpiler warning/error on the next line.
- `// SKIP SYMBOLFILE` — mark the file as declaration-only; the implementation lives in a parallel `.kt`.
- `// SKIP EXTERN` — JNI / native function reference.
- `// SKIP @nocopy` — disable `MutableStruct` wrapping on a struct (see above).
- `// SKIP @nodispatch` — prevent automatic dispatch of an `async` function onto a Kotlin coroutine context.
- `// SKIP @bridge` / `@bridgeMembers` / `@nobridge` — control JNI bridging in Fuse mode (see [skip-fuse](../skip-fuse/SKILL.md)).

The directives and the Kotlish syntax (named parameters use `:` not `=`; closures use `in` not `->`; wildcard imports use `.__` not `.*`) are covered in detail in [`references/swift-kotlin-rules.md`](references/swift-kotlin-rules.md).

## Kotlin interop — calling Kotlin and Java APIs

```swift
#if SKIP
import android.content.Context
let context = ProcessInfo.processInfo.androidContext
let uuid = java.util.UUID.randomUUID().toString()
#endif
```

For iterating over a Kotlin `Map`, use `entry.value` and `entry.key`:

```swift
#if SKIP
for entry in someMap {
    let key = entry.key
    let value = entry.value
}
#endif
```

The Swift `for (k, v) in dict` form does not work for Kotlin Maps.

## Supported features

Closures, generics (with limits — see above), optionals, `async`/`await`, `@Observable`, pattern matching, `guard`/`if let`, `defer`, `Codable`, string interpolation, collection operations (`map`, `filter`, `reduce`).

## Unsupported features

Key paths as values, complex macros, C interop (use SkipFFI), `Mirror` / reflection, variadic generics, `UIViewRepresentable`. The `Array.remove(atOffsets:)` SwiftUI-extension trap is its own case — see above.

## Diagnosing failures

```bash
swift test --filter XCSkipTests    # Run transpilation + Kotlin compilation
swift test --filter XCSkipTests -v # Verbose Gradle errors
```

Kotlin compile errors appear with `.kt` file paths and line numbers; cross-reference with the original Swift source.

## References

- [`references/swift-kotlin-rules.md`](references/swift-kotlin-rules.md) — full long-form rules: strings, structs/`MutableStruct`/`.sref()`, generics, GC behaviour, Kotlish dialect, every SKIP comment directive with examples, `.kotlin()` conversion.
- [`references/supported-features.md`](references/supported-features.md) — feature support table.
