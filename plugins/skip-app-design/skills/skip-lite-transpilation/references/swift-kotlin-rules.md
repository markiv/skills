# Swift-to-Kotlin transpilation rules — exhaustive reference

The main [skip-lite-transpilation](../SKILL.md) skill covers the rules that fail loudest (integer overflow, `Double == Int`, `Foundation.sqrt`, etc.). This file covers the rest. Everything here is sourced from `skip.dev/docs/swiftsupport/` and `skip.dev/docs/platformcustomization/`.

## Strings

Swift strings transpile to Kotlin `String`, which has several behaviour gaps you need to plan around.

**Strings are immutable.** You cannot mutate a Kotlin `String` in place. Anywhere Swift code calls `s.append(...)`, `s.removeFirst()`, `s += otherString` on a `var`, the transpiled Kotlin must produce a new string.

```swift
// BAD — mutation patterns that don't transpile cleanly
var s = "hello"
s.append(" world")     // Kotlin String has no mutating append
s.removeLast()

// GOOD — work with new strings
var s = "hello"
s = s + " world"        // assignment to a var works
let trimmed = String(s.dropLast())
```

**Strings are not `Collection`.** Skip adds many of `Collection`'s APIs to `String` (so `.count`, `.isEmpty`, `.first`, `.last`, `.contains(...)`, `.map`, `.filter` work), but you cannot **pass a `String` to code expecting `Collection<Character>`** — Kotlin doesn't let you bolt a protocol onto an existing type the way Swift does.

```swift
// BAD — function expects a generic Collection
func process<C: Collection>(_ c: C) where C.Element == Character { ... }
process("hello")        // Doesn't compile on Kotlin

// GOOD — convert explicitly
process(Array("hello"))
```

**`String(describing:)` doesn't transpile.** Kotlin sees `String(describing = value)` which is not a valid constructor. Use string interpolation:

```swift
// BAD
let s = String(describing: someValue)

// GOOD
let s = "\(someValue)"
```

## Memory management — `weak` and `unowned` are ignored

Skip uses Kotlin's garbage collector. `weak` and `unowned` on properties and in closure capture lists are **silently ignored**.

```swift
class Parent {
    var child: Child?
}
class Child {
    weak var parent: Parent?     // 'weak' compiles, but Kotlin sees a strong reference
}
```

The GC breaks reference cycles on its own, so most apps run correctly. The cases that break:

- Code that uses `weak` to detect deallocation (`weakRef.object == nil` as a signal). The check will lie.
- Tight coupling between objects where the user's mental model assumes immediate teardown.

If your code relies on `weak` for anything other than cycle-breaking, rework it.

## `deinit` becomes `finalize`, may not run

Swift `deinit` transpiles to Kotlin's `finalize()` method, which is called **at an indeterminate time and may not be called at all** — same semantics as Java finalizers.

```swift
final class FileHandle {
    deinit {
        close(fd)                // may never run on Android
    }
}
```

Don't put critical cleanup (file close, network disconnect, lock release) in `deinit` if the app must run correctly on Android. Use explicit cleanup:

- `defer { cleanup() }` inside a function scope
- `try` / `do { } catch { } cleanup()` patterns
- A typed `close()` method the caller invokes explicitly
- For Codable types backing a file, write on `didSet` rather than on deinit

## Structs and value semantics — `MutableStruct` and `.sref()`

Kotlin classes are reference types by default; Swift structs are value types. Skip bridges this by wrapping mutable structs in a `MutableStruct` protocol and inserting **`.sref()`** ("struct reference") calls automatically. You'll see `.sref()` calls all over your transpiled output. That's normal.

The overhead exists. If you have a struct that's technically mutable (it has `var` properties) but you only set its properties once at construction time and never modify them after, you can opt out of the wrapping with the `@nocopy` directive:

```swift
// SKIP @nocopy
struct Config {
    var apiURL: URL
    var timeout: TimeInterval
}
```

`@nocopy` is a [SKIP comment directive](#skip-comment-directives) — see below. Apply it when:

- You're storing immutable-after-init structs in a hot loop and the `.sref()` overhead shows up in profiling.
- You're getting Kotlin compile errors about `MutableStruct` conformance on a struct that doesn't need it.

Don't apply it to structs whose `var` properties genuinely change after construction; you'll get reference-sharing bugs.

Two related rules from the main skill:

- Prefer `final class` over `struct` for complex model types with many methods. The `.sref()` insertion is usually fine for plain-data structs but can introduce subtle issues with structs that have lots of methods.
- Prefer named methods (`add()`, `multiply()`) over operator overloads. Operator transpilation works for arithmetic types but is fragile for custom types.

## Generics

Skip has hard limits on what generics can do once they cross into Kotlin.

**Type erasure.** Generic type parameters are erased at runtime. `is` and `as?` checks against generic types **ignore the generics**:

```swift
let xs: Any = [1, 2, 3]
if xs is [String] {       // True on Kotlin — generics erased
    let strings = xs as! [String]
    // strings.first crashes — they're actually Ints
}
```

Don't rely on generic identity at runtime. Carry the type explicitly if you need it.

**Static members of generic types.** Skip can support a static member of a generic type only if (a) the member doesn't use the type's generic parameters, or (b) the member can be converted into a generic function on its own.

```swift
// OK — doesn't use T
struct Container<T> {
    static var version: Int { 1 }
}

// NOT OK — uses T
struct Container<T> {
    static var defaultValue: T? { nil }      // can't transpile
}
```

**Inner types of generic outer types are not supported.**

```swift
struct Outer<T> {
    struct Inner { ... }    // not supported
}
```

**Reified generics via `@inline(__always)`.** When you need the generic type available at runtime — for `is`, `as?`, type-based dispatch — annotate the function with `@inline(__always)`. Skip transpiles it to a Kotlin `inline fun <reified T>`, which preserves the type:

```swift
@inline(__always)
func decode<T: Decodable>(_ data: Data) throws -> T {
    return try JSONDecoder().decode(T.self, from: data)
}
```

Use sparingly — every call site inlines the body, which bloats Kotlin output.

## Variadic initializers don't transpile

Swift variadic parameters in initialisers (`init(_ items: Item...)`) don't transpile correctly. Use array parameters:

```swift
// BAD
init(_ events: HapticEvent...) { self.events = events }

// GOOD
init(_ events: [HapticEvent]) { self.events = events }
```

## Kotlin map iteration in `#if SKIP` blocks

When iterating over a Kotlin `Map` inside `#if SKIP`, use `entry.value` and `entry.key`:

```swift
#if SKIP
for entry in someMap {
    let key = entry.key
    let value = entry.value
}
#endif
```

The Swift `Dictionary` iteration form (`for (k, v) in dict`) does not work for Kotlin Maps.

## The Kotlish dialect

Inside `#if SKIP` blocks, Skip relaxes its Swift parsing so you can write things that look like Swift to Xcode and the Swift compiler but compile to Kotlin via the transpiler. This hybrid syntax is called **Kotlish**. It is what makes calls into Kotlin / Java APIs possible without writing a wrapper.

Key Kotlish quirks (vs. straight Swift):

- **Named parameters use `:` not `=`** inside Kotlish, even though Kotlin uses `=`: `someMap.put(key: "a", value: 1)`.
- **Trailing-closure lambdas use `in` not `->`**: `list.forEach { it in print(it) }`.
- **Wildcard imports use `.__` not `.*`** because `*` is not valid in a Swift import: `import android.content.__`.
- **Anonymous inner classes use `// SKIP INSERT`** because Swift has no equivalent syntax (see directives below).
- **Property delegates use `by`** which is invalid Swift; must go through `SKIP INSERT`.

Kotlish is the right tool when you need to call Kotlin/Java APIs from a Lite module and the call site is straightforward. When the call requires real Kotlin syntax that won't pass through Swift parsing — `:` class references, anonymous inner classes, property delegates — fall back to a `SKIP INSERT` block.

## SKIP comment directives

These special `// SKIP X` comments instruct the transpiler to do something it wouldn't do on its own. Most are Lite-only; a few work in both modes.

### `// SKIP INSERT: <kotlin>`

Inject literal Kotlin code at the point of the comment. The Swift compiler ignores it. Use for short single-line cases:

```swift
// SKIP INSERT: @OptIn(DelicateCoroutinesApi::class)
func backgroundWork() async { ... }
```

For multi-line insertions, prefer the `/* SKIP INSERT: ... */` block form:

```swift
/* SKIP INSERT:
private val greeting by remember { mutableStateOf("Hello") }
*/
```

Most common SKIP INSERT targets:

1. Class references with `:` or `::` syntax (illegal in Swift): `MyClass::class`
2. Anonymous inner classes: `timer.schedule(object : TimerTask() { ... })`
3. Kotlin property delegates with `by`: `val greeting by remember { mutableStateOf("Hello") }`

### `// SKIP REPLACE: <kotlin>`

Replace the next Swift statement with literal Kotlin. Used when a single Swift line needs to be replaced with a different Kotlin shape:

```swift
// SKIP REPLACE: mac.init(secretKeySpec)
mac.update(data: data)        // Swift call form; replaced wholesale
```

Prefer SKIP REPLACE over SKIP INSERT when the Swift form is plausible-looking but the transpiler can't handle it; the Swift compiler sees the original and the transpiled output gets the replacement.

### `// SKIP DECLARE: <kotlin declaration>`

Override the **declaration line** of a type or function while transpiling the body normally. Useful when the Kotlin signature needs to differ from what the Swift compiler would accept:

```swift
// SKIP DECLARE: open class JSONEncoder: TopLevelEncoder<Data>
public class JSONEncoder: TopLevelEncoder { ... }
```

The Swift compiler sees `public class JSONEncoder: TopLevelEncoder`; the transpiler emits the `open class JSONEncoder: TopLevelEncoder<Data>` from the directive.

### `// SKIP NOWARN`

Silence a transpiler warning or error on the next line. Use when you know the transpiler is wrong or when you've handled the edge case manually:

```swift
// SKIP NOWARN
let mirror = Mirror(reflecting: someValue)
```

### `// SKIP SYMBOLFILE`

Mark a Swift file as a **declaration-only** file: the transpiler emits no Kotlin output for it. The implementation must exist in a parallel `.kt` file you provide. Useful when you have a Kotlin implementation that doesn't have a clean Swift counterpart.

### `// SKIP EXTERN`

Mark a function as an external (JNI / native) reference. The transpiler emits the signature only; the Kotlin side links to a native implementation. Used in `SkipFFI`-style packages.

### `// SKIP @nocopy`

Apply to a `struct` to disable the `MutableStruct` wrapping (see [Structs](#structs-and-value-semantics--mutablestruct-and-sref)).

### `// SKIP @nodispatch`

Apply to an `async` function to prevent the transpiler from automatically dispatching it onto a Kotlin coroutine context. Useful when you've already wrapped the function in your own coroutine context.

### `// SKIP @bridge` / `// SKIP @bridgeMembers` / `// SKIP @nobridge`

Control which declarations get bridged across the JNI boundary in Fuse mode. Apply per-declaration (`@bridge`), per-type (`@bridgeMembers` bridges all public members), or exclude a single member (`@nobridge` with auto-bridging enabled). See [skip-fuse](../../skip-fuse/SKILL.md) for the Fuse mode workflow.

## `.kotlin()` conversion for Skip wrappers

Skip wraps a number of Foundation types (`Array`, `Calendar`, `Date`, `URL`) in Swift-shape wrappers that hold an underlying Kotlin/Java object. Call `.kotlin()` on a Skip wrapper to get the underlying Kotlin object:

```swift
#if SKIP
let url = URL(string: "https://example.com")!
let javaURI = url.kotlin()       // java.net.URI
#endif
```

`.kotlin()` takes an optional `nocopy: true` parameter when you want to share the underlying object without a defensive copy. Use this only when you control all the call sites and know the object won't be mutated through the Skip wrapper after the conversion.

## Conditional compilation — `#if SKIP` vs `#if os(Android)`

These two conditionals behave differently in Lite vs. Fuse. For Lite specifically:

- `#if SKIP` and `#if os(Android)` are **equivalent**. Both contents transpile to Kotlin.
- `#if !SKIP` is iOS-only — Skip excludes the contents from the Android build.

So in Lite, `#if os(Android)` blocks can call Kotlin and Java APIs directly. The behaviour diverges in Fuse — see [skip-fuse](../../skip-fuse/SKILL.md) for that mode's rules.
