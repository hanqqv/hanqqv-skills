---
name: go-best-practices
description: Write and review idiomatic Go following Effective Go, Go Code Review Comments, and the Google Go Style Guide — naming, error handling, interfaces, context, concurrency, HTTP services, library API design, and testing. Use when writing Go, reviewing Go, or answering how something should be done in Go.
---

# Go best practices

Baseline: **Effective Go**, **Go Code Review Comments**, and the **Google Go Style Guide**. Where those disagree with a third-party guide, follow these. Where they are silent, prefer the standard library's own example.

Apply these when writing new Go and when reviewing. For *changing* existing Go safely, see the `go-refactoring` skill.

## The non-negotiables

1. **`gofmt`.** Never discuss formatting; run the tool. `gofmt -l .` prints nothing.
2. **Handle every error.** `_ =` requires a comment saying why it is safe.
3. **Never start a goroutine without knowing how it stops.**
4. **`context.Context` is the first parameter, named `ctx`, never a struct field.**
5. **Run `go vet ./...` and `go test -race ./...`.** Both, in CI.
6. **Accept interfaces, return structs.**

## Naming

- **Packages:** short, lowercase, single word, no underscores, no plurals. The package name is part of every reference, so `user.New()` not `user.NewUser()`. Never `util`, `common`, `helpers`, `base`, `misc` — a package named for what it is not cannot have a coherent API.
- **No stuttering:** `http.Server`, not `http.HTTPServer`. `user.Service`, not `user.UserService`.
- **Length tracks scope.** `i` in a three-line loop is right; `i` as a struct field is not. Receivers are one or two letters (`s *Server`), consistently across all methods of a type.
- **Interfaces:** single-method interfaces take the method name plus `-er` (`Reader`, `Formatter`). Multi-method interfaces get a role name (`Sender`, not `SenderInterface` or `ISender`).
- **No getters prefix:** the accessor for `owner` is `Owner()`, not `GetOwner()`. Setters do take `Set`: `SetOwner()`.
- **Acronyms keep their case:** `userID`, `ServeHTTP`, `parseURL` — never `userId` or `ParseUrl`.
- **Errors:** sentinel values are `ErrNotFound`; types are `ParseError`.

## Errors

- **Wrap with `%w`** when the caller might inspect the cause; use `%v` when you are deliberately opaquing it. Wrapping is a promise about your API.
- **Error strings are lowercase, unpunctuated**, and read as a fragment: `"parse config: unexpected EOF"`. They get concatenated, so no capital letters or trailing periods.
- **Add context, don't repeat it.** `fmt.Errorf("read config %s: %w", path, err)` — never `"error: failed to: %w"`.
- **Inspect with `errors.Is` / `errors.As`**, never string matching. Go 1.26+ has `errors.AsType[T]` for a type-safe generic form.
- **Log *or* return, never both.** Double-reporting is how one failure becomes five log lines.
- **`panic` only for programmer error** that cannot be recovered from. Library code returns errors. A `recover()` at a goroutine boundary to keep a server up is legitimate; `recover()` as control flow is not.

Details, sentinel-vs-typed guidance, and `errors.Join`: `references/errors.md`.

## Interfaces and types

- **Define the interface where it is consumed**, not beside the implementation. The consumer knows which methods it needs.
- **Keep interfaces small.** One to three methods. "The bigger the interface, the weaker the abstraction."
- **Return concrete types** from constructors. Let callers choose their own abstraction.
- **Do not create an interface with one implementation** "for testability" until a second implementation or a real test double exists.
- **Make the zero value useful** where you can. `bytes.Buffer`, `sync.Mutex`, and `slog.Logger` all work unconstructed; that removes a whole class of nil checks.
- **Embed for behavior, field for data.** Embedding promotes the whole method set into your public API — that is an API decision, not a shortcut.
- **Pointer vs value receivers: pick one per type.** If any method needs a pointer, all do. Anything containing a `sync` type or a large array must use pointers.

## Context

- First parameter, named `ctx`, typed `context.Context`. Never stored in a struct, never optional.
- **Only request-scoped values** in `context.WithValue`, with an unexported key type. Never optional parameters, never dependencies.
- **The caller owns cancellation.** A function receiving a `ctx` respects it; it does not extend or replace its deadline unless that is its documented job.
- `context.TODO()` marks a gap to close, not a default. `context.Background()` belongs in `main`, tests, and top-level goroutine roots.

## Concurrency

- **Every goroutine needs an owner** who knows when it stops. In practice: `errgroup.Group`, or `sync.WaitGroup` plus context cancellation. `sync.WaitGroup.Go` (Go 1.25+) removes the `Add`/`defer Done` boilerplate.
- **Don't communicate by sharing memory; share memory by communicating** — but a `sync.Mutex` guarding a struct field is often the simpler, faster answer. Channels are for handing off ownership, not for protecting a counter.
- **The sender closes the channel**, never the receiver, never a second sender.
- **Unbuffered by default.** Reach for a buffer only when you can name the capacity's meaning.
- **`-race` in CI.** A race the detector finds in five seconds can take a quarter to reproduce in production.
- **Don't use `time.Sleep` to synchronize.** In tests, use `testing/synctest` (GA in Go 1.25); in production, use a channel or a condition.

Patterns, worker pools, cancellation, and leak detection: `references/concurrency.md`.

## HTTP services

- **Never use `http.DefaultServeMux` or `http.DefaultClient`** in library or server code — they are global mutable state any dependency can modify. Construct your own.
- **Always set timeouts** on both `http.Server` (`ReadHeaderTimeout` at minimum) and `http.Client`. The zero-value `http.Client` waits forever.
- **Handlers are thin.** Decode, call a domain function, encode. Business logic in a handler cannot be tested without a request.
- **Propagate `r.Context()`** into everything the handler calls, so client disconnects cancel the work.
- **Bound request bodies** with `http.MaxBytesReader` before decoding.
- **Return errors as data**, not as `http.Error` calls scattered through helpers — one place decides the status code.
- **Graceful shutdown**: `signal.NotifyContext` → `srv.Shutdown(ctx)`.

Routing, middleware shape, encoding helpers, and a full server skeleton: `references/http-services.md`.

## Library and package API design

- **Export the minimum.** Everything exported is a promise you maintain. Unexport first; export on demand.
- **Functional options** (`WithTimeout(d)`) for constructors that must stay backward compatible; a plain options struct for internal code.
- **No `pkg/` directory.** Use `internal/` for what must not be imported, and real package names for the rest.
- **Flat until it hurts.** Do not build a directory tree in anticipation of growth.
- **Dependencies are a liability.** The stdlib covers more than most people assume — `slices`, `maps`, `cmp`, `log/slog`, `net/http`, `testing`.
- **Document exported identifiers** with a comment starting with the identifier's name. `// Server handles ...`, not `// This struct is a server ...`.
- **Backward compatibility:** add, don't change. Deprecate with `// Deprecated: use X instead.` and keep the old path working.

Versioning, option patterns, and compatibility mechanics: `references/api-design.md`.

## Testing

- **Table-driven with subtests** is the default shape. `t.Run(tt.name, ...)` gives you per-case names and `-run` filtering.
- **No assertion library needed.** `if got != want { t.Errorf(...) }`. The message names the input, the got, and the want.
- **`t.Helper()`** in every helper, so failures report the caller's line.
- **`t.Cleanup()`** over `defer` for teardown that must survive subtests.
- **`t.Parallel()`** where tests are independent — and run `-race`, since parallel tests are how shared-state bugs surface.
- **Test the exported API** from `package foo_test` where practical. It keeps you honest about what the package actually offers.
- **`testing/synctest`** for anything time-dependent: virtual clock, no flakes, no sleeps.
- **`go test -race ./...` and `go vet ./...`** are part of "the tests pass".

Fakes vs mocks, golden files, fuzzing, benchmarks: `references/testing.md`.

## Tooling

| Tool | Role |
|---|---|
| `gofmt` / `gofumpt` | Formatting. Not a discussion. |
| `go vet` | Correctness checks the compiler skips. Runs by default under `go test`. |
| `staticcheck` | The high-signal linter. Worth adopting first. |
| `golangci-lint` | Aggregator, for repos that want a configured suite. |
| `go fix ./...` | Since Go 1.26, hosts "modernizers" that rewrite dated idioms. Run as its own commit. |
| `gopls rename` | Module-wide safe rename. |
| `go test -race -cover` | The real test command. |

## Go version awareness

Check `go.mod` before suggesting anything version-gated. Current release is **Go 1.27** (Aug 2026); 1.26 and 1.27 are the supported pair.

Things people still write that are now obsolete:

- **Loop variable capture** — since **1.22**, `for` variables are per-iteration. `x := x` shadowing is no longer needed.
- **`interface{}`** — use `any` (**1.18**).
- **Hand-written min/max/contains/sort helpers** — `min`/`max`/`clear` builtins (**1.21**), `slices`, `maps`, `cmp` packages.
- **`logrus`/`zap` by reflex** — `log/slog` is in the stdlib (**1.21**), with `NewMultiHandler` (**1.26**).
- **Custom iterator interfaces** — range-over-func and `iter.Seq` (**1.23**).
- **`for i := 0; i < b.N; i++`** — `b.Loop()` (**1.24**) is more accurate and, since **1.26**, no longer blocks inlining.
- **`time.Sleep` in concurrency tests** — `testing/synctest` (GA **1.25**, plus `synctest.Sleep` in **1.27**).
- **`wg.Add(1)` / `defer wg.Done()`** — `sync.WaitGroup.Go` (**1.25**).
- **Manually setting `GOMAXPROCS` in containers** — the runtime is cgroup-aware since **1.25**.
- **`errors.As` with a declared target variable** — `errors.AsType[T]` (**1.26**).
- **`strings.LastIndex` + slicing** — `strings.CutLast` / `bytes.CutLast` (**1.27**).

Full table with exact signatures: `references/version-features.md`.

## Review output format

```
## Findings
1. <Rule> — path/to/file.go:42
   Issue: <one line>
   Fix: <concrete change>

## Blocking
<correctness, races, unhandled errors, API breaks>

## Non-blocking
<naming, structure, idiom>
```

Separate blocking from non-blocking. A race and a naming nit are not the same finding, and a review that mixes them gets skimmed.

## Anti-patterns to refuse

- Adding an interface with one implementation for "flexibility".
- Suggesting a dependency where three stdlib lines would do.
- Getter/setter pairs on plain data.
- `pkg/`, `util/`, `models/`, `helpers/` directories.
- Struct tags or reflection where a plain function would work.
- Rewriting working code to use generics because generics exist.
- Blanket "add error handling" review comments that don't say what the caller should do with the error.
