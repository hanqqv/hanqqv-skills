---
name: go-refactoring
description: Refactor Go code using the Fowler / Refactoring Guru method adapted to Go — Go-specific smells, the catalog entries that do and do not translate to a language without classes or inheritance, and the go build/vet/test -race verification loop. Use when asked to refactor, clean up, restructure, or review Go code for smells.
---

# Refactoring Go

Refactoring is **restructuring without changing external behavior**. This skill applies that discipline to Go, where the classic object-oriented catalog only half applies: Go has no classes, no inheritance, and no subclass polymorphism, so roughly a third of the standard refactorings are dead entries and others take a different shape.

For general Go style and correctness rules, see the `go-best-practices` skill. This skill is about *changing* existing code safely.

## Core discipline

1. **Green before you start.** `go test ./...` must pass. No test on the code being changed? Write a characterization test first — Go's table-driven style makes this cheap. Never refactor untested Go blind; say so and offer the test.
2. **One named refactoring at a time.**
3. **Verify after every step** with the loop below. Green → continue. Red → revert that step, do not fix forward.
4. **Never mix refactoring with behavior change.** Separate commits.
5. **Let the tools do the mechanical work.** `gopls` rename is safe across a module; hand-editing identifiers is not.

## The verification loop

Run after every step. This is the Go equivalent of "the tests are green":

```
gofmt -l .                  # must print nothing
go build ./...
go vet ./...
go test -race ./...
```

Add `staticcheck ./...` or `golangci-lint run` when the repo already uses them — do not introduce a new linter mid-refactor, that is a behavior change to the build.

`-race` is not optional when touching anything concurrent. A refactoring that moves a variable across a goroutine boundary can introduce a race that non-race tests pass straight through.

## Catalog translation

Go lacks classes and inheritance, so the OO catalog does not map one-to-one. Full table in `references/catalog-mapping.md`. The short version:

| Classic refactoring | In Go |
|---|---|
| Extract Method | Extract Function / Extract Method — unchanged, the workhorse |
| Extract Class | Extract struct, or extract a new package |
| Replace Conditional with Polymorphism | Replace type switch with interface — only when the set of cases is open |
| Replace Type Code with Subclasses | Replace with an interface, or a named type plus methods |
| Extract Superclass | **No equivalent.** Extract an interface, or embed a shared struct |
| Pull Up / Push Down Method | **No equivalent.** Move the method, or hoist into an embedded type |
| Refused Bequest | **Does not exist** — no inheritance to refuse |
| Replace Inheritance with Delegation | Already the default; the smell is the reverse — embedding used where a field belongs |
| Introduce Parameter Object | Introduce an options struct, or functional options for a public API |
| Replace Temp with Query | Unchanged |
| Hide Delegate | Unchanged, but weigh it against Go's preference for flat, explicit calls |
| Introduce Null Object | Rare in Go — a usable zero value usually serves the same purpose |

## Go-specific smells

### Interface and type smells
| Smell | Signal | Refactoring |
|---|---|---|
| Producer-side interface | Package defines an interface then returns it, with one implementation | Move the interface to the consumer package; return the concrete type |
| Fat interface | Interface with 8+ methods, implementers stub most of them | Split into role interfaces; keep them 1–3 methods |
| `any` as a type | `map[string]any`, `func(any)` where the shape is known | Replace with a struct, or a type parameter |
| Speculative interface | Interface with exactly one implementation, no test double needed | Delete it; use the concrete type |
| Stuttering names | `http.HTTPServer`, `user.UserService` | Rename — the package qualifies it |

### Structure smells
| Smell | Signal | Refactoring |
|---|---|---|
| God struct | Service struct with 10+ dependency fields | Extract Struct along the lines the methods cluster on |
| `util` / `common` / `helpers` package | Package named for what it is not | Move each function to the package that owns its type; delete the package |
| Primitive obsession | `string` user IDs, `int` status codes | Define named types (`type UserID string`) and give them methods |
| Long parameter list | 4+ params, or a `bool` flag param | Options struct; split the function on the flag |
| Package-level mutable state | `var cache = map[...]` at package scope | Move into a struct the caller constructs and owns |
| Work in `init()` | `init()` reading config, opening connections | Move into an explicit constructor the caller calls |

### Flow and error smells
| Smell | Signal | Refactoring |
|---|---|---|
| Arrow code | Nesting 3+ deep on the happy path | Invert conditions, return early, keep the happy path at the left margin |
| Swallowed error | `_ = doThing()`, or `if err != nil { log.Println(err) }` and continue | Return it wrapped, or document why it is ignorable |
| Opaque error | `fmt.Errorf("failed: %v", err)` | `%w` and let callers use `errors.Is` / `errors.As` |
| Naked return in a long function | Named results returned bare, 20+ lines away from the signature | Extract Function, or return explicit values |
| `panic` as control flow | Library code panicking on bad input | Return an error; reserve panic for programmer error |

### Concurrency smells
| Smell | Signal | Refactoring |
|---|---|---|
| Orphan goroutine | `go f()` with no way to stop it or know it finished | Give it an owner: `errgroup`, or `sync.WaitGroup` + context cancellation |
| Context in a struct | `type S struct { ctx context.Context }` | Pass `ctx` as the first parameter of each method |
| Mutex copied | Value receiver on a type embedding `sync.Mutex` | Pointer receivers; `go vet` catches most of these |
| Exported mutex | `type S struct { sync.Mutex }` embedded and exported | Make it an unexported field: `mu sync.Mutex` |
| `time.Sleep` in tests | Test waits a fixed duration for a goroutine | `testing/synctest` (Go 1.25+), or a channel the test can wait on |

## Working procedure

1. **Read first.** Note the package's exported surface — that is the contract that must not change. Changing an exported identifier in a library is a breaking change, not a refactoring.
2. **Inventory smells** with `file:line` and the smell's name. `go vet ./...` and `staticcheck ./...` first — do not hand-report what a tool already finds.
3. **Rank by payoff:** code under active change first, then smells causing real bugs, then the rest.
4. **Propose the plan** before editing. Get agreement on anything touching an exported identifier, a shared package, or more than a few files.
5. **Execute one step at a time**, running the verification loop between each.
6. **Report** each step as `<Technique>: <what moved where>` plus the loop result.

## Mechanical helpers

- `gopls rename` — module-wide, safe. Prefer it over find-and-replace.
- `go fix ./...` — since Go 1.26 this hosts "modernizers" that rewrite dated idioms (e.g. pre-generics helpers, `interface{}` → `any`). Run it as its own commit so the diff is reviewable.
- `gofmt -r 'a[b:len(a)] -> a[b:]'` — pattern rewrites for mechanical shape changes.
- `go build ./... 2>&1 | head` after deleting an exported identifier tells you every caller.

## Anti-patterns to refuse

- Refactoring with no tests and no offer to add them.
- Introducing an interface with one implementation "for testability" when the concrete type is already testable.
- Splitting a package because it is long. Go packages are split by *responsibility*, not line count.
- Adding a `pkg/` or `util/` directory as part of "cleanup".
- Changing exported names in a published module without a deprecation path.
- Running the repo's linter with new rules enabled and calling the fallout a refactoring.

## Output format

When reviewing:

```
## Smells found
1. <Smell name> — path/to/file.go:42
   Why: <one line>
   Fix: <named refactoring>

## Proposed order
1. ... (highest payoff / lowest risk first)

## Risks
<exported surface touched, untested paths, concurrency>
```

## Reference

- `references/catalog-mapping.md` — every classic refactoring and what it becomes in Go, including the ones that vanish.
- `references/worked-examples.md` — step-by-step mechanics for the Go refactorings that come up most: extracting a struct, replacing a type switch with an interface, moving an interface to the consumer, giving a goroutine an owner, and writing a characterization test for untested Go.
