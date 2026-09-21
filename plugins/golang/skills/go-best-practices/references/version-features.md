# Go version features

**Always check `go.mod` before suggesting anything below.** A module declaring `go 1.21` does not get per-iteration loop variables even on a 1.27 toolchain — language changes are gated on the module's declared version.

Current release: **Go 1.27** (August 2026). Supported: 1.26 and 1.27.

---

## Quick "stop writing this" table

| Dated idiom | Replacement | Since |
|---|---|---|
| `x := x` before a goroutine in a loop | Nothing — loop vars are per-iteration | 1.22 |
| `interface{}` | `any` | 1.18 |
| Hand-written `Contains`, `IndexOf`, `Sort` | `slices` package | 1.21 |
| Hand-written `Keys`, `Values` on maps | `maps` package | 1.21 |
| `if a < b { m = a } else { m = b }` | `min` / `max` builtins | 1.21 |
| `for k := range m { delete(m, k) }` | `clear(m)` | 1.21 |
| `logrus`, `zap` by reflex | `log/slog` | 1.21 |
| `for i := 0; i < 10; i++` over a count | `for range 10` | 1.22 |
| Custom iterator interfaces | `iter.Seq`, range-over-func | 1.23 |
| `for i := 0; i < b.N; i++` | `b.Loop()` | 1.24 |
| `wg.Add(1)` + `defer wg.Done()` | `wg.Go(func(){...})` | 1.25 |
| `time.Sleep` in concurrency tests | `testing/synctest` | 1.25 (GA) |
| Manual `GOMAXPROCS` in containers | Automatic, cgroup-aware | 1.25 |
| `var e *MyErr; errors.As(err, &e)` | `errors.AsType[*MyErr](err)` | 1.26 |
| `s[:strings.LastIndex(s, sep)]` | `strings.CutLast` | 1.27 |
| `github.com/google/uuid` | `uuid` stdlib package | 1.27 |

---

## By release

### Go 1.18 — generics
Type parameters, `any`, `comparable`, workspaces (`go.work`), native fuzzing.

### Go 1.20
`errors.Join`, multiple `%w` verbs in `fmt.Errorf`, `context.WithCancelCause`.

### Go 1.21
- `log/slog` — structured logging in the stdlib.
- `slices`, `maps`, `cmp` packages.
- `min`, `max`, `clear` builtins.
- `context.WithoutCancel`, `context.AfterFunc`.
- `errors.ErrUnsupported`.

### Go 1.22
- **Per-iteration loop variables.** The single most consequential change for correctness; deletes the `x := x` idiom.
- `for range 10` — range over an integer.
- **`http.ServeMux` method and wildcard patterns**: `mux.Handle("GET /users/{id}", h)`, `r.PathValue("id")`. Removes the need for a router in most services.
- `math/rand/v2`.

### Go 1.23
- **Range-over-func iterators** and the `iter` package (`iter.Seq`, `iter.Seq2`).
- `slices.Collect`, `slices.Sorted`, `maps.All`, and friends returning iterators.
- `unique` package for value interning.

### Go 1.24
- `testing.B.Loop()` — accurate benchmarks without `b.N`.
- **Generic type aliases.**
- `os.Root` — directory-scoped filesystem access, resistant to path traversal.
- `go tool` directive in `go.mod` for tracking tool dependencies.
- `testing/synctest` as `GOEXPERIMENT=synctest`.
- `weak` package.

### Go 1.25
- **`testing/synctest` generally available** — `synctest.Test`, `synctest.Wait`, virtual clock.
- **`sync.WaitGroup.Go`** — combines `Add`, `go`, and `defer Done`.
- **Container-aware `GOMAXPROCS`** — respects cgroup CPU limits on Linux and updates as limits change. Disable with `GODEBUG=containermaxprocs=0`. `runtime.SetDefaultGOMAXPROCS` restores default behavior.
- `encoding/json/v2` as an experiment.

### Go 1.26
- **`new()` accepts expressions**: `Age: new(yearsSince(born))` — no temporary variable for optional pointer fields.
- **Self-referential generic constraints**: `type Adder[A Adder[A]] interface { Add(A) A }`.
- **`errors.AsType[T](err) (T, bool)`** — type-safe generic `errors.As`.
- `log/slog.NewMultiHandler` — fan out to several handlers.
- `bytes.Buffer.Peek`; `reflect` iterator methods (`Type.Fields()`, `Value.Methods()`, …).
- `testing.T.ArtifactDir()` for test output files.
- `testing.B.Loop()` no longer blocks inlining.
- **Green Tea GC** is the default — 10–40% less GC overhead.
- **`go fix` rewritten** to host "modernizers" that rewrite dated idioms automatically, plus `//go:fix inline` for automated API migration.
- `crypto/hpke`; post-quantum hybrid TLS key exchange on by default.
- Goroutine leak profile (experimental, `GOEXPERIMENT=goroutineleakprofile`).
- macOS 12 support ends after this release.

### Go 1.27
- **Generic methods** — methods may declare their own type parameters. Interfaces still cannot have generic methods, and a generic method cannot implement an interface method.
- Struct literals accept any valid field selector as a key, enabling nested initialization.
- **`encoding/json/v2` and `encoding/json/jsontext`** — stricter defaults (rejects duplicate keys and invalid UTF-8), variadic `Options`. `encoding/json` is now backed by v2 with compatibility preserved; opt out with `GOEXPERIMENT=nojsonv2`.
- **`uuid` package in the stdlib.**
- `strings.CutLast`, `bytes.CutLast`.
- `net/url.URL.Clone`, `Values.Clone`.
- **`goroutineleak` profile generally available** — finds goroutines blocked on unreachable primitives.
- `testing/synctest.Sleep` — `time.Sleep` plus `synctest.Wait`.
- `httptest.NewTestServer` — in-memory network, works inside a synctest bubble.
- HTTP/2 client priority signals (RFC 9218); HTTP/1 auto-drains unread response bodies for better connection reuse.
- `http.Server.MaxHeaderValueCount`.
- `go test` runs the `stdversion` vet check by default, flagging use of too-new stdlib symbols.
- `go mod tidy` normalizes `require` blocks into direct/indirect pairs.
- ~30% faster small allocations.
- macOS 13+ required.

---

## Version-gating advice

When the module's `go` directive is older than a feature you want to suggest:

1. Say which version introduced it and what bumping `go.mod` would cost.
2. Offer the pre-version idiom as the immediate answer.
3. Don't bump `go.mod` as part of an unrelated change — it affects every build and every consumer of a library.

The `stdversion` vet check (on by default in `go test` since 1.27) catches use of stdlib symbols newer than the declared version, so this class of mistake now fails loudly rather than at a user's build.
