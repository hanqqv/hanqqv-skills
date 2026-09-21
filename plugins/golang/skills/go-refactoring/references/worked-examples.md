# Worked examples — refactoring Go

Each example gives the smell, the refactoring, and the mechanics step by step. Run the verification loop (`gofmt -l . && go build ./... && go vet ./... && go test -race ./...`) between every numbered step.

---

## 1. Characterization test first (no tests exist)

Do not refactor untested Go. Pin current behavior, including behavior that looks wrong.

```go
func TestCalculateDiscount_Characterization(t *testing.T) {
    tests := []struct {
        name string
        amt  int
        tier string
        want int
    }{
        {"gold", 100, "GOLD", 15},
        {"empty tier", 100, "", 0},      // probably a bug — pinned, not fixed
        {"negative amount", -5, "GOLD", -1}, // same
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := CalculateDiscount(tt.amt, tt.tier); got != tt.want {
                t.Errorf("CalculateDiscount(%d, %q) = %d, want %d", tt.amt, tt.tier, got, tt.want)
            }
        })
    }
}
```

Fixing a pinned bug is a separate commit, after the refactoring lands. Note the failure message names inputs and both values — Go has no assertion library and the message is all the reader gets.

---

## 2. God struct → Extract Struct

**Before**

```go
type Server struct {
    db          *sql.DB
    cache       *redis.Client
    mailer      *smtp.Client
    templates   *template.Template
    signingKey  []byte
    tokenTTL    time.Duration
    rateLimiter *rate.Limiter
    logger      *slog.Logger
}
```

Methods cluster: three touch only `signingKey`/`tokenTTL`, four touch only `mailer`/`templates`.

**Mechanics**

1. Create `type tokenIssuer struct { key []byte; ttl time.Duration }` with no users yet. Verify.
2. Copy one method's body onto `tokenIssuer`, leaving the `Server` method delegating to it. Verify.
3. Repeat per method, one at a time, verifying between each.
4. Replace the two fields on `Server` with `tokens tokenIssuer`. Verify.
5. Inline the delegating methods at call sites where callers can reach `s.tokens` directly; keep the delegation where it is part of `Server`'s public API. Verify.

**After**

```go
type Server struct {
    db        *sql.DB
    cache     *redis.Client
    notifier  notifier      // mailer + templates
    tokens    tokenIssuer   // signingKey + tokenTTL
    limiter   *rate.Limiter
    logger    *slog.Logger
}
```

**Do not** extract along "layers" (everything-service, everything-repository). Extract along the lines the methods already cluster on.

---

## 3. Type switch → interface (Replace Conditional with Polymorphism)

Only do this when the case set is **open**. A closed set of three cases is clearer as a switch.

**Before**

```go
func (n *Notification) Send(ctx context.Context) error {
    switch n.Channel {
    case "email":
        return sendEmail(ctx, n.To, n.Subject, n.Body)
    case "sms":
        return sendSMS(ctx, n.To, n.Body)
    case "push":
        return sendPush(ctx, n.DeviceToken, n.Title, n.Body)
    }
    return fmt.Errorf("unknown channel %q", n.Channel)
}
```

The same switch is repeated in `Validate()` and `RetryPolicy()` — that repetition is the actual signal.

**Mechanics**

1. Define the interface in the package that *uses* it:
   ```go
   type Sender interface {
       Send(ctx context.Context, n Notification) error
   }
   ```
   Verify — nothing uses it yet.
2. Write `emailSender` implementing `Send`, copying the branch body. Verify.
3. Add a registry: `map[string]Sender`. Route `Send` through it for the `"email"` case only, keeping the switch for the rest. Verify.
4. Repeat per case, one at a time. Verify between each.
5. When the switch is empty, delete it. Verify.
6. Now move `Validate()` and `RetryPolicy()` onto the interface, repeating steps 2–5 for each.

**After**

```go
type Sender interface {
    Send(ctx context.Context, n Notification) error
    Validate(n Notification) error
    RetryPolicy() retry.Policy
}
```

If after step 6 the interface has grown past ~4 methods, split it — a fat interface is its own smell.

---

## 4. Producer-side interface → Move Interface to Consumer

**Before** — `storage/store.go`:

```go
package storage

type Store interface {
    Get(ctx context.Context, id string) (*User, error)
    Put(ctx context.Context, u *User) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, f Filter) ([]*User, error)
    Count(ctx context.Context, f Filter) (int, error)
}

func NewPostgres(db *sql.DB) Store { return &postgres{db: db} }
```

Two problems: the constructor returns an interface, and the interface is defined next to its only implementation. The consumer `billing` calls `Get` and nothing else.

**Mechanics**

1. Change `NewPostgres` to return `*Postgres` (exported concrete type). Verify — callers assigning to `Store` still compile, since `*Postgres` satisfies it.
2. In `billing`, define exactly what it needs:
   ```go
   package billing

   type userGetter interface {
       Get(ctx context.Context, id string) (*storage.User, error)
   }
   ```
   Verify.
3. Change `billing`'s struct field and constructor parameter from `storage.Store` to `userGetter`. Verify.
4. Repeat per consumer package, each declaring its own narrow interface. Verify between each.
5. When no package references `storage.Store`, delete it. Verify.

**Why:** `billing` can now be tested with a three-line fake instead of a five-method stub, and `storage` can add methods without touching any consumer.

---

## 5. Orphan goroutine → Give the Goroutine an Owner

**Before**

```go
func (s *Server) Start() {
    go s.pollUpstream()
    go s.flushMetrics()
    go s.reapSessions()
}
```

Nothing can stop these, nothing knows if one died, and a test that calls `Start()` leaks three goroutines into every later test.

**Mechanics**

1. Give each function a `ctx` parameter and make its loop select on `ctx.Done()`:
   ```go
   func (s *Server) pollUpstream(ctx context.Context) error {
       t := time.NewTicker(s.interval)
       defer t.Stop()
       for {
           select {
           case <-ctx.Done():
               return ctx.Err()
           case <-t.C:
               if err := s.poll(ctx); err != nil {
                   s.logger.ErrorContext(ctx, "poll failed", "err", err)
               }
           }
       }
   }
   ```
   Verify after each function.
2. Replace the bare `go` calls with an `errgroup`:
   ```go
   func (s *Server) Run(ctx context.Context) error {
       g, ctx := errgroup.WithContext(ctx)
       g.Go(func() error { return s.pollUpstream(ctx) })
       g.Go(func() error { return s.flushMetrics(ctx) })
       g.Go(func() error { return s.reapSessions(ctx) })
       return g.Wait()
   }
   ```
   Verify with `-race`.
3. Update callers from `Start()` to `Run(ctx)` and handle the returned error.
4. Delete `Start()`.

Without `errgroup`, `sync.WaitGroup.Go` (Go 1.25+) removes the `Add`/`defer Done` boilerplate for the fire-and-wait case.

---

## 6. Primitive obsession → Replace Data Value with Object

**Before**

```go
func Transfer(fromAccount, toAccount string, amountCents int64, currency string) error
```

Nothing stops `Transfer(to, from, ...)`, and currency validation is duplicated at every call site.

**Mechanics**

1. Define the types with no users yet:
   ```go
   type AccountID string
   type Money struct {
       Cents    int64
       Currency Currency
   }
   ```
   Verify.
2. Move validation into a constructor: `func NewMoney(cents int64, c Currency) (Money, error)`. Verify.
3. Change the signature to `Transfer(from, to AccountID, amount Money) error`. Verify — the compiler now lists every caller.
4. Fix callers one at a time. Verify between each.
5. Delete the now-unreachable per-call-site validation. Verify.

Swapping `from` and `to` still compiles (both are `AccountID`), so keep the argument-order test. The win is the currency invariant, now unrepresentable-if-invalid.

---

## 7. Arrow code → Guard Clauses

**Before**

```go
func process(r *Request) error {
    if r != nil {
        if r.User != nil {
            if r.User.Active {
                if err := validate(r); err == nil {
                    return handle(r)
                } else {
                    return fmt.Errorf("validate: %v", err)
                }
            } else {
                return errors.New("inactive user")
            }
        } else {
            return errors.New("no user")
        }
    }
    return errors.New("nil request")
}
```

**Mechanics** — invert one condition per step, outermost first, verifying each time.

**After**

```go
func process(r *Request) error {
    if r == nil {
        return errors.New("nil request")
    }
    if r.User == nil {
        return errors.New("no user")
    }
    if !r.User.Active {
        return errors.New("inactive user")
    }
    if err := validate(r); err != nil {
        return fmt.Errorf("validate: %w", err)
    }
    return handle(r)
}
```

The happy path is now a straight line at the left margin. Note `%v` → `%w` — that is a *separate* concern; if callers might `errors.Is` on it, do it as its own commit.

---

## 8. `util` package → Collapse the Utility Package

**Before**

```
internal/util/
    strings.go   // TruncateName, SlugifyTitle
    time.go      // BusinessDaysBetween
    http.go      // WriteJSON, ReadJSON
```

**Mechanics** — one function per step, using `gopls` move/rename:

1. `TruncateName`, `SlugifyTitle` operate on `user.Name` → move to `internal/user`. Verify.
2. `BusinessDaysBetween` is used only by `billing` → move there, unexported. Verify.
3. `WriteJSON`/`ReadJSON` are genuinely cross-cutting HTTP concerns → move to `internal/httpx` (a real name for a real responsibility). Verify.
4. `internal/util` is now empty. Delete it. Verify.

A package named for what it *is not* can never have a coherent API. `httpx` is fine; `util` is not.
