# Testing

## Table-driven with subtests

The default shape for anything with more than one case:

```go
func TestParseDuration(t *testing.T) {
    tests := []struct {
        name    string
        in      string
        want    time.Duration
        wantErr bool
    }{
        {"seconds", "30s", 30 * time.Second, false},
        {"mixed", "1h30m", 90 * time.Minute, false},
        {"empty", "", 0, true},
        {"garbage", "abc", 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := ParseDuration(tt.in)
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseDuration(%q) error = %v, wantErr %v", tt.in, err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("ParseDuration(%q) = %v, want %v", tt.in, got, tt.want)
            }
        })
    }
}
```

- `t.Run(tt.name, ...)` gives per-case failure names and `-run 'TestParseDuration/empty'` filtering.
- `t.Fatalf` when continuing would panic or produce a second misleading failure; `t.Errorf` otherwise.
- Failure messages name the **input**, the **got**, and the **want**, in that order. That message is everything the reader gets.
- Since Go 1.22 the loop variable is per-iteration, so `tt := tt` is no longer needed.

## No assertion library

Go's testing philosophy is that a failure message should say what happened, not just that something did. `assert.Equal(t, a, b)` produces a worse message than four lines of Go.

```go
// Adequate
if got != want {
    t.Errorf("Sum(%v) = %d, want %d", in, got, want)
}

// For structs and slices
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("Process() mismatch (-want +got):\n%s", diff)
}
```

`github.com/google/go-cmp/cmp` is the one testing dependency worth taking — `cmp.Diff` on a large struct is dramatically more readable than `reflect.DeepEqual`'s boolean. Use `cmpopts.IgnoreFields`, `cmpopts.SortSlices` for the usual noise.

## Helpers

```go
func newTestServer(t *testing.T) *Server {
    t.Helper()                      // failures report the caller's line

    db := openTestDB(t)
    srv := New(db)

    t.Cleanup(func() {              // runs after subtests, unlike defer
        srv.Close()
    })
    return srv
}
```

- **`t.Helper()`** in every helper that calls `t.Error`/`t.Fatal`. Without it, every failure points at the helper.
- **`t.Cleanup()`** over `defer` — it runs in the right order relative to subtests and parallel tests.
- **Never call `t.Fatal` from a goroutine.** It calls `runtime.Goexit` on the wrong goroutine and the test hangs or misreports. Send the error to a channel and fail on the test goroutine.

## Fakes over mocks

Prefer a hand-written fake implementing the consumer's narrow interface:

```go
type fakeUserGetter struct {
    users map[string]*user.User
    err   error
}

func (f *fakeUserGetter) Get(_ context.Context, id string) (*user.User, error) {
    if f.err != nil {
        return nil, f.err
    }
    u, ok := f.users[id]
    if !ok {
        return nil, user.ErrNotFound
    }
    return u, nil
}
```

This is why narrow, consumer-side interfaces matter: a one-method interface takes four lines to fake. A generated mock with call-order expectations tests your wiring, not your behavior, and breaks on every refactor.

Use a real dependency when it is cheap — SQLite or a testcontainer for a repository test catches SQL errors a fake never will.

## External test packages

```go
package user_test    // not package user

import "example.com/internal/user"
```

Testing from outside forces you to use the package the way callers do, and stops tests from depending on internals. Keep a same-package `export_test.go` for the few internals that genuinely need direct testing:

```go
// export_test.go
package user

var ValidateEmail = validateEmail   // exported to the _test package only
```

## Time and concurrency

**`testing/synctest`** (GA in Go 1.25) runs a test inside a bubble with a virtual clock. Time advances instantly when every goroutine in the bubble is blocked:

```go
func TestCacheExpiry(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        c := NewCache(WithTTL(time.Hour))
        c.Set("k", "v")

        time.Sleep(59 * time.Minute)    // instant, virtual
        if _, ok := c.Get("k"); !ok {
            t.Error("entry expired early")
        }

        time.Sleep(2 * time.Minute)
        if _, ok := c.Get("k"); ok {
            t.Error("entry did not expire")
        }
    })
}
```

`synctest.Wait()` blocks until every goroutine in the bubble is blocked — the deterministic replacement for "sleep and hope". Go 1.27 adds `synctest.Sleep`, combining the two.

Before 1.25, inject a clock:

```go
type Clock interface{ Now() time.Time }
```

**Never `time.Sleep` to synchronize.** Slow when it works, flaky when it doesn't.

**`-race` always:**

```bash
go test -race ./...
```

## Golden files

For large outputs — rendered templates, formatted reports, serialized structures:

```go
var update = flag.Bool("update", false, "update golden files")

func TestRender(t *testing.T) {
    got := Render(input)
    golden := filepath.Join("testdata", t.Name()+".golden")

    if *update {
        if err := os.WriteFile(golden, got, 0o644); err != nil {
            t.Fatal(err)
        }
    }

    want, err := os.ReadFile(golden)
    if err != nil {
        t.Fatal(err)
    }
    if diff := cmp.Diff(string(want), string(got)); diff != "" {
        t.Errorf("Render() mismatch (-want +got):\n%s", diff)
    }
}
```

Regenerate with `go test -run TestRender -update`, then **read the diff** before committing — an unreviewed golden update silently blesses a bug.

`testdata/` is ignored by the go tool, so anything can live there.

## Fuzzing

Built in since Go 1.18. Worth it for any parser, decoder, or input validator:

```go
func FuzzParseConfig(f *testing.F) {
    f.Add([]byte("key = value"))       // seed corpus
    f.Add([]byte(""))

    f.Fuzz(func(t *testing.T, data []byte) {
        cfg, err := ParseConfig(data)
        if err != nil {
            return                      // rejecting bad input is correct
        }
        // Round-trip property: anything parsed must re-serialize and re-parse
        out, err := cfg.Marshal()
        if err != nil {
            t.Fatalf("Marshal after successful parse: %v", err)
        }
        if _, err := ParseConfig(out); err != nil {
            t.Fatalf("reparse own output: %v", err)
        }
    })
}
```

`go test -fuzz=FuzzParseConfig`. Failing inputs are written to `testdata/fuzz/` and become permanent regression cases.

## Benchmarks

```go
func BenchmarkRender(b *testing.B) {
    input := loadFixture(b)
    b.ReportAllocs()
    b.ResetTimer()

    for b.Loop() {          // Go 1.24+
        _ = Render(input)
    }
}
```

`b.Loop()` (Go 1.24) replaces `for i := 0; i < b.N; i++`. It keeps the value alive so the compiler cannot optimize the call away, and since Go 1.26 it no longer prevents inlining. Pre-1.24, assign to a package-level sink to defeat dead-code elimination.

Compare with `benchstat` across several `-count` runs; a single benchmark number is noise.

## Coverage

```bash
go test -race -coverprofile=cover.out ./...
go tool cover -html=cover.out
```

Use it to **find untested branches**, not as a target. A coverage percentage says nothing about whether the assertions are meaningful, and chasing 100% produces tests that execute code without checking it.

## Checklist

- [ ] Table-driven with `t.Run` subtests
- [ ] Failure messages name input, got, want
- [ ] `t.Helper()` in every helper
- [ ] `t.Cleanup()` for teardown
- [ ] `t.Parallel()` where independent
- [ ] `-race` in CI
- [ ] No `time.Sleep` for synchronization
- [ ] `package foo_test` where practical
- [ ] Fakes, not generated mocks
- [ ] Fuzz tests on parsers and decoders
