# The classic catalog in Go

Go has no classes, no inheritance, no subclass polymorphism, no constructors, no overloading, and no exceptions. That removes or reshapes a large part of the Fowler catalog. This table is the full mapping.

**Legend:** ✅ applies unchanged · 🔁 applies in a different shape · ❌ no Go equivalent

---

## Composing methods

| Refactoring | | Go |
|---|---|---|
| Extract Method | ✅ | Extract Function (package-level) or Extract Method (on a receiver). The single most-used refactoring in Go. |
| Inline Method | ✅ | Unchanged. Go 1.26+ can automate this via `//go:fix inline` directives. |
| Extract Variable | ✅ | Unchanged. |
| Inline Temp | ✅ | Unchanged. |
| Replace Temp with Query | ✅ | Unchanged, but weigh the cost — Go has no property syntax, so a query is a visible function call. |
| Split Temporary Variable | ✅ | Unchanged. The compiler's "declared and not used" helps. |
| Remove Assignments to Parameters | ✅ | Unchanged. Note Go params are already copies, so this is about clarity, not aliasing. |
| Replace Method with Method Object | 🔁 | Extract a struct holding the locals as fields, with a `run()` method. Common when untangling a 200-line handler. |
| Substitute Algorithm | ✅ | Unchanged. |

## Moving features between objects

| Refactoring | | Go |
|---|---|---|
| Move Method | ✅ | Move the method to the type whose data it uses, or to another package. |
| Move Field | ✅ | Unchanged. |
| Extract Class | 🔁 | **Extract Struct** when the cluster is data + behavior; **extract a package** when it is a whole responsibility. Extracting a package is the bigger win in Go and the more commonly needed one. |
| Inline Class | 🔁 | Inline Struct, or fold a thin package back into its only caller. Thin single-use packages are a real Go smell. |
| Hide Delegate | 🔁 | Applies, but Go culture prefers explicit chains over hidden ones. Use it to protect an invariant, not to shorten a line. |
| Remove Middle Man | ✅ | Unchanged. Go accumulates these when wrapper types are added "for testability". |
| Introduce Foreign Method | 🔁 | **Cannot add methods to another package's type.** Write a package-level function taking the type, or define a local named type wrapping it. |
| Introduce Local Extension | 🔁 | Define `type MyThing struct { otherpkg.Thing }` and add methods, or `type MyThing otherpkg.Thing` for a clean break. Embedding keeps the original's methods; conversion does not. |

## Organizing data

| Refactoring | | Go |
|---|---|---|
| Self Encapsulate Field | ❌ | Anti-idiomatic. Go has no property syntax and `GetX()`/`SetX()` pairs are discouraged. Use the field directly; add a method only when it enforces something. |
| Replace Data Value with Object | 🔁 | Define a named type: `type Email string` with a `Validate()` method, or a struct when it has parts. The single best fix for primitive obsession in Go. |
| Change Value to Reference | 🔁 | Switch to a pointer and an owning collection/registry. Consider whether a usable zero value removes the need. |
| Change Reference to Value | 🔁 | Make the type comparable and copyable. Watch for embedded `sync` types — those must not be copied. |
| Replace Array with Object | ✅ | Replace `[]string{name, email}` or `map[string]any` with a struct. |
| Duplicate Observed Data | ❌ | Rarely applies; Go has no UI-binding tradition. |
| Change Unidirectional to Bidirectional | ✅ | Unchanged, and still usually a mistake — prefer one direction plus a lookup. |
| Encapsulate Field | 🔁 | Move the field to lowercase (unexported) and add the method that maintains its invariant. The exported/unexported boundary is Go's encapsulation, not getters. |
| Encapsulate Collection | 🔁 | Return a copy or an iterator (`iter.Seq`, Go 1.23+) instead of the backing slice or map. Returning a map hands out write access. |
| Replace Type Code with Class | 🔁 | `type Status int` plus a `String()` method and a stringer-generated table. |
| Replace Type Code with Subclasses | 🔁 | **No subclasses.** Use an interface with one implementation per case, or keep the type code and switch on it in one place. |
| Replace Type Code with State/Strategy | 🔁 | An interface field on the struct, swapped at runtime. Idiomatic and common. |

## Simplifying conditional expressions

| Refactoring | | Go |
|---|---|---|
| Decompose Conditional | ✅ | Unchanged. |
| Consolidate Conditional Expression | ✅ | Unchanged. |
| Consolidate Duplicate Conditional Fragments | ✅ | Unchanged. |
| Remove Control Flag | ✅ | Use `break`, labeled `break`, or early `return`. |
| Replace Nested Conditional with Guard Clauses | ✅ | **The most valuable conditional refactoring in Go.** Idiomatic Go keeps the happy path at the leftmost indentation and returns early on every error. |
| Replace Conditional with Polymorphism | 🔁 | Replace a type switch with an interface — but only when the case set is *open* (callers may add cases). A closed set of 3 cases is clearer as a switch. Go does not reward polymorphism the way Java does. |
| Introduce Null Object | 🔁 | Usually unnecessary: design a usable zero value instead. Where you do need it, a no-op implementation of the interface (e.g. a discard logger) is the Go form. |
| Introduce Assertion | 🔁 | No `assert`. Return an error for caller mistakes; `panic` only for programmer error that cannot be recovered from. |

## Making method calls simpler

| Refactoring | | Go |
|---|---|---|
| Rename Method | ✅ | Use `gopls rename`. Renaming an *exported* identifier in a published module is a breaking change — deprecate, don't rename. |
| Add / Remove Parameter | ✅ | Same breaking-change caveat for exported functions. |
| Separate Query from Modifier | ✅ | Unchanged. |
| Parameterize Method | 🔁 | A parameter, or a type parameter (generics) where the variation is the type itself. |
| Replace Parameter with Explicit Methods | ✅ | Especially for `bool` flag parameters: `Sort()` / `SortDescending()` beats `Sort(desc bool)`. |
| Preserve Whole Object | ✅ | Unchanged. |
| Replace Parameter with Method Call | ✅ | Unchanged. |
| Introduce Parameter Object | 🔁 | An options struct for internal code; **functional options** (`WithTimeout(d)`) for a public API you must keep backward compatible. |
| Remove Setting Method | 🔁 | Make the field unexported and set it in the constructor. |
| Hide Method | 🔁 | Lowercase it. That is Go's entire access-control mechanism. |
| Replace Constructor with Factory Method | 🔁 | Go has no constructors. The idiom is already a `NewX()` function; this refactoring is usually about *splitting* one `New` into several named ones (`NewFromFile`, `NewFromReader`). |
| Replace Error Code with Exception | ❌ | Backwards in Go. Errors *are* values; the refactoring goes the other way. |
| Replace Exception with Test | 🔁 | Replace `recover()`-based flow with an explicit error return and a precondition check. |

## Dealing with generalization

| Refactoring | | Go |
|---|---|---|
| Pull Up Field / Method / Constructor Body | ❌ | No superclass. The nearest move is hoisting shared state into a struct that the others *embed*, which is composition, not inheritance. |
| Push Down Method / Field | ❌ | Same. |
| Extract Subclass | ❌ | Define a separate type; share via embedding or a common interface. |
| Extract Superclass | ❌ | Extract an **interface** (behavior) or a shared **struct to embed** (state). These are different tools — do not conflate them. |
| Extract Interface | ✅ | Applies, with a Go twist: **define the interface in the consuming package**, not next to the implementation. |
| Collapse Hierarchy | 🔁 | Remove a pointless embedding layer, or delete a single-implementation interface. |
| Form Template Method | 🔁 | **No abstract methods.** Use a struct with function fields, or pass the varying step as a parameter. `sort.Slice` is this pattern in the stdlib. |
| Replace Inheritance with Delegation | 🔁 | Already the default. In Go the live smell is the mirror image: embedding used where a plain field would be clearer, leaking the embedded type's whole method set into your API. |
| Replace Delegation with Inheritance | ❌ | No inheritance. If you want the method set promoted, embed — but that is a deliberate API decision, not a cleanup. |

---

## Go-only refactorings with no classic counterpart

| Refactoring | When |
|---|---|
| **Move Interface to Consumer** | An interface sits beside its implementation. Move it to the package that calls it, narrowed to the methods that caller actually uses. |
| **Return Concrete Type** | A constructor returns an interface. Return the struct; let callers pick their own abstraction. ("Accept interfaces, return structs.") |
| **Narrow Interface** | A function takes `*os.File` but only reads. Take `io.Reader`. |
| **Introduce Usable Zero Value** | A type requires `NewX()` for no reason. Make the zero value work (`bytes.Buffer`, `sync.Mutex`) and the constructor optional. |
| **Give the Goroutine an Owner** | A bare `go f()` becomes an `errgroup.Group` or a `sync.WaitGroup` + context, so the caller knows when it ends and can cancel it. |
| **Thread Context Through** | Replace a stored `ctx` field or `context.TODO()` with `ctx` as the first parameter along the whole call path. |
| **Wrap the Error** | `%v` → `%w`, plus sentinel errors or typed errors so callers can `errors.Is` / `errors.As`. |
| **Collapse the Utility Package** | Every function in `util/` moves to the package owning its primary type; the package is deleted. |
| **Replace Reflection with Generics** | A `func(any)` doing type switches becomes `func[T any]` — when the variation really is parametric. |
| **Replace Manual Loop with Stdlib** | `slices`, `maps`, `cmp`, and `min`/`max`/`clear` replace hand-written helpers. `go fix` modernizers automate many of these.  |
