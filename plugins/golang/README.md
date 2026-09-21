# golang

Two skills for working in Go: one for writing and reviewing it, one for changing it safely.

## Skills

### `go-best-practices`

Idiomatic Go per **Effective Go**, **Go Code Review Comments**, and the **Google Go Style Guide**. Loads when writing Go, reviewing Go, or answering "how should this be done in Go".

Covers naming, error handling, interfaces, context, concurrency, HTTP services, library API design, testing, and tooling — plus a version-awareness section so Claude checks `go.mod` before suggesting anything version-gated.

References, loaded on demand:

| File | Contents |
|---|---|
| `errors.md` | Opaque vs sentinel vs typed, `%w` as an API commitment, message style, panic/recover, boundary mapping |
| `concurrency.md` | Goroutine ownership, errgroup and WaitGroup patterns, channels, mutexes, cancellation, synctest |
| `http-services.md` | Server skeleton, timeouts, thin handlers, encode/decode helpers, middleware, graceful shutdown |
| `api-design.md` | Export minimum, accept interfaces/return structs, zero values, functional options, layout, compatibility |
| `testing.md` | Table-driven subtests, fakes over mocks, golden files, fuzzing, benchmarks, coverage |
| `version-features.md` | What landed in each release through Go 1.27, and the dated idioms each one retires |

### `go-refactoring`

The Fowler / Refactoring Guru method adapted to Go. Loads on "refactor this", "clean up this Go", "review for code smells".

Go has no classes, no inheritance, and no subclass polymorphism, so a large part of the classic catalog does not translate. This skill says which entries apply unchanged, which change shape, and which vanish — plus Go-only refactorings the catalog never had (Move Interface to Consumer, Return Concrete Type, Give the Goroutine an Owner, Collapse the Utility Package).

Go-specific smells it detects: producer-side interfaces, fat interfaces, `any` where a type is known, god structs, `util` packages, primitive obsession, swallowed and opaque errors, arrow code, orphan goroutines, contexts stored in structs, copied mutexes, `time.Sleep` in tests.

References:

| File | Contents |
|---|---|
| `catalog-mapping.md` | Every classic refactoring, marked applies / reshaped / no equivalent, with the Go form |
| `worked-examples.md` | Step-by-step mechanics for the eight refactorings that come up most |

## How they fit together

`go-best-practices` is the standard; `go-refactoring` is how you move existing code toward it without breaking anything. The refactoring skill's verification loop is:

```
gofmt -l . && go build ./... && go vet ./... && go test -race ./...
```

Run between every step. `-race` is not optional when touching concurrent code.

## Expect caution

Asked to refactor untested Go, Claude stops and offers a characterization test rather than editing blind. Asked to change an exported identifier in a library, it flags the break rather than doing it quietly. If that's stricter than you want, edit the **Core discipline** section of `skills/go-refactoring/SKILL.md`.

## Versions

Written against **Go 1.27** (August 2026), with features attributed to the release that introduced them. Both skills check the module's `go` directive before suggesting anything version-gated.
