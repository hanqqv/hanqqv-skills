# Library and package API design

## Export the minimum

Everything exported is a promise. Start unexported; export when a caller outside the package actually needs it.

```go
package user

type Service struct {   // exported: callers construct it
    store  store         // unexported: an implementation detail
    hasher hasher
}

func NewService(db *sql.DB) *Service { ... }     // exported
func (s *Service) Create(...) (*User, error)     // exported
func (s *Service) validate(u *User) error        // unexported
```

The exported/unexported boundary **is** Go's encapsulation. There are no getters and setters to write, and adding `GetName()`/`SetName()` on a plain field is anti-idiomatic — use the field.

## Accept interfaces, return structs

```go
// Good
func NewStore(db *sql.DB) *PostgresStore

// Bad — the caller can no longer reach methods you add later,
// and is stuck with your idea of the right abstraction
func NewStore(db *sql.DB) Store
```

Define interfaces **where they are consumed**, narrowed to what that consumer needs:

```go
package billing

// Exactly the one method billing uses — a test fake is three lines.
type userGetter interface {
    Get(ctx context.Context, id string) (*user.User, error)
}

type Invoicer struct{ users userGetter }
```

Not exporting the interface is fine and usually right — the caller passes a concrete `*user.Service` and it just works.

## Zero values

A usable zero value removes a constructor and a whole class of nil checks. The stdlib does this everywhere: `bytes.Buffer`, `sync.Mutex`, `sync.WaitGroup`, `slog.Logger`.

```go
// Good — works unconstructed
type Counter struct {
    mu sync.Mutex
    n  map[string]int   // lazily initialized in Inc
}

func (c *Counter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.n == nil {
        c.n = make(map[string]int)
    }
    c.n[key]++
}
```

When the zero value genuinely cannot work (a required connection, a required key), a constructor returning an error is right — don't fake it with lazy panics.

## Functional options

For a constructor that must stay backward compatible across versions:

```go
type Server struct {
    addr    string
    timeout time.Duration
    logger  *slog.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithLogger(l *slog.Logger) Option {
    return func(s *Server) { s.logger = l }
}

func New(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        timeout: 30 * time.Second,        // defaults stated in one place
        logger:  slog.Default(),
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

Adding `WithTLS` later breaks nobody. Required arguments stay positional; only optional ones become options.

**Don't use options for internal code.** A plain struct is clearer and cheaper when you control every caller:

```go
type config struct {
    timeout time.Duration
    retries int
}
func newClient(cfg config) *client
```

Options that can fail return an error:

```go
type Option func(*Server) error
```

## Project layout

**Flat until it hurts.** A single-package program is a legitimate final state.

```
myservice/
    go.mod
    main.go
    server.go
    server_test.go
```

Grow by responsibility, not by anticipation:

```
myservice/
    go.mod
    cmd/
        myservice/main.go      // only when there are several binaries
    internal/
        user/                  // domain, named for the concept
        billing/
        httpx/                 // shared HTTP helpers — a real responsibility
    migrations/
```

- **`internal/`** is enforced by the toolchain: nothing outside the module can import it. Put everything there by default in an application.
- **No `pkg/`.** It carries no meaning and adds a directory level to every import path. Go's own repo does not use it.
- **No `models/`, `types/`, `util/`, `common/`, `helpers/`.** Packages are named for what they *do*, and types live with the behavior that operates on them.
- **`cmd/`** only with more than one binary.

## Documentation

Doc comments begin with the identifier's name and are complete sentences:

```go
// Service manages user accounts and their credentials.
// A zero Service is not usable; construct one with NewService.
type Service struct { ... }

// Create registers a new user with the given email and name.
// It returns ErrEmailTaken if the email is already registered.
func (s *Service) Create(ctx context.Context, email, name string) (*User, error)
```

Document on the **exported** identifier, and say what callers need: preconditions, what errors mean, whether it is safe for concurrent use, whether it retains a slice you pass in.

Package docs go in one file — `doc.go` when long:

```go
// Package user manages account lifecycle: registration, authentication,
// and profile updates.
//
// All Service methods are safe for concurrent use.
package user
```

Runnable `Example` functions in `_test.go` files appear in godoc and are compiled and run by `go test`, so they cannot rot.

## Backward compatibility

Within a major version: **add, don't change.**

| Change | Breaking? |
|---|---|
| Add a function or method | No |
| Add a field to a struct | No — unless callers use unkeyed literals |
| Add a method to an **interface** | **Yes** — every implementer breaks |
| Change a signature | Yes |
| Rename an exported identifier | Yes |
| Change an exported type's underlying type | Yes |
| Remove a `%w` wrap | **Yes** — callers' `errors.Is` stops matching |
| Tighten an input validation | Usually yes |

Deprecate rather than remove:

```go
// Deprecated: use NewServiceWithConfig instead. This will be removed in v2.
func NewService(db *sql.DB) *Service {
    return NewServiceWithConfig(db, DefaultConfig())
}
```

Tooling recognizes the `Deprecated:` prefix and flags callers.

For a module at `v2` or beyond, the major version goes in the import path: `example.com/mod/v2`. That is how Go allows a breaking change without breaking anyone who has not opted in.

**Defensive move:** add an unexported, zero-width field to structs callers construct, so unkeyed literals fail and adding a field stays non-breaking:

```go
type Options struct {
    Timeout time.Duration
    Retries int
    _       struct{}   // forces keyed literals
}
```

## Dependencies

Each one is code you did not write, running in your process, that can break your build. The stdlib covers more than most assume:

| Instead of | Use |
|---|---|
| `logrus`, `zap` | `log/slog` (1.21+) |
| `pkg/errors` | `fmt.Errorf` with `%w`, `errors.Is/As/Join` |
| `testify/assert` | `if got != want { t.Errorf(...) }` |
| `golang.org/x/exp/slices` | `slices` (1.21+) |
| `google/uuid` | `uuid` (1.27+) |
| `gorilla/mux` (simple routing) | `http.ServeMux` with method+wildcard patterns (1.22+) |
| hand-written iterators | `iter.Seq`, range-over-func (1.23+) |

`golang.org/x/...` is maintained by the Go team and is a reasonable default when the stdlib falls short. `golang.org/x/sync/errgroup` in particular is close to mandatory for concurrent code.

## Generics

Use type parameters when the **type itself** is the variation and the code is otherwise identical:

```go
// Good — identical logic, any comparable key
func Keys[K comparable, V any](m map[K]V) []K

// Bad — generics buying nothing over an interface
func Process[T Processor](items []T) error   // just take []Processor
```

Rules of thumb:

- If you write it once and never instantiate it with a second type, it does not need to be generic.
- If the body does a type switch on `T`, generics are the wrong tool — use an interface.
- Generic **methods** arrived in Go 1.27 (methods may declare their own type parameters), but interfaces still cannot have them, and a generic method cannot implement an interface method.
- Concrete beats generic for readability. Write it concrete twice; generalize on the third.
