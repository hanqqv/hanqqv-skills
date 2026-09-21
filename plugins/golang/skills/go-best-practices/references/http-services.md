# HTTP services

## Server skeleton

```go
func main() {
    if err := run(context.Background(), os.Args[1:], os.Stdout); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
}

func run(ctx context.Context, args []string, stdout io.Writer) error {
    ctx, cancel := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
    defer cancel()

    cfg, err := parseConfig(args)
    if err != nil {
        return fmt.Errorf("config: %w", err)
    }

    logger := slog.New(slog.NewJSONHandler(stdout, nil))

    db, err := sql.Open("pgx", cfg.DatabaseURL)
    if err != nil {
        return fmt.Errorf("open db: %w", err)
    }
    defer db.Close()

    srv := &http.Server{
        Addr:              cfg.Addr,
        Handler:           newRouter(logger, db),
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       30 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
    }

    g, ctx := errgroup.WithContext(ctx)
    g.Go(func() error {
        logger.Info("listening", "addr", srv.Addr)
        if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
            return err
        }
        return nil
    })
    g.Go(func() error {
        <-ctx.Done()
        shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), 20*time.Second)
        defer cancel()
        return srv.Shutdown(shutdownCtx)
    })
    return g.Wait()
}
```

Why `run(ctx, args, stdout)`: everything is injected, so a test can call `run` directly with a cancellable context and capture output. `main` does nothing but wire and exit. `context.WithoutCancel` (Go 1.21+) keeps the shutdown context alive after the parent is cancelled — otherwise `Shutdown` is cancelled the instant it starts.

## Timeouts are not optional

The zero-value `http.Server` and `http.Client` both wait forever. A server with no `ReadHeaderTimeout` is trivially DoS-able by opening connections and sending nothing.

```go
// Server — minimum viable
srv := &http.Server{
    ReadHeaderTimeout: 5 * time.Second,   // the critical one
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
}

// Client — never use http.DefaultClient
client := &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConnsPerHost: 100,
        IdleConnTimeout:     90 * time.Second,
    },
}
```

`http.DefaultClient` and `http.DefaultServeMux` are package-level mutable state that any imported package can modify. Construct your own, always.

## Handlers

**Thin.** Decode, call a domain function, encode:

```go
func handleCreateUser(svc *user.Service, logger *slog.Logger) http.HandlerFunc {
    type request struct {
        Email string `json:"email"`
        Name  string `json:"name"`
    }
    type response struct {
        ID string `json:"id"`
    }

    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()

        req, err := decode[request](r)
        if err != nil {
            writeError(w, r, http.StatusBadRequest, err)
            return
        }

        u, err := svc.Create(ctx, req.Email, req.Name)
        if err != nil {
            writeError(w, r, statusFor(err), err)
            return
        }

        encode(w, r, http.StatusCreated, response{ID: u.ID})
    }
}
```

Three things worth copying:

- **Returning `http.HandlerFunc` from a closure** injects dependencies without a handler struct and without globals.
- **Request/response types declared inside the handler** keeps them out of the package API when nothing else uses them.
- **`r.Context()` threaded through** means a client disconnect cancels the database query.

## Encode/decode helpers

```go
func decode[T any](r *http.Request) (T, error) {
    var v T
    dec := json.NewDecoder(http.MaxBytesReader(nil, r.Body, 1<<20))
    dec.DisallowUnknownFields()
    if err := dec.Decode(&v); err != nil {
        return v, fmt.Errorf("decode json: %w", err)
    }
    return v, nil
}

func encode[T any](w http.ResponseWriter, r *http.Request, status int, v T) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(v); err != nil {
        slog.ErrorContext(r.Context(), "encode response", "err", err)
    }
}
```

- **`http.MaxBytesReader`** before decoding. Without it, a client can stream gigabytes into your decoder.
- **`DisallowUnknownFields`** turns typos in client payloads into errors instead of silently-ignored fields. Skip it for public APIs where forward compatibility matters more.
- **Header before `WriteHeader` before body** — that order is mandatory; headers set afterwards are silently dropped.
- Go 1.27's `encoding/json/v2` is stricter by default (rejects duplicate keys and invalid UTF-8) and takes `Options` arguments. `encoding/json` is now backed by it while preserving old behavior.

## Errors to status codes, in one place

```go
func statusFor(err error) int {
    switch {
    case errors.Is(err, user.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, user.ErrEmailTaken):
        return http.StatusConflict
    case errors.Is(err, context.DeadlineExceeded):
        return http.StatusGatewayTimeout
    case errors.Is(err, context.Canceled):
        return 499 // client closed request
    }
    var ve *user.ValidationError
    if errors.As(err, &ve) {
        return http.StatusBadRequest
    }
    return http.StatusInternalServerError
}

func writeError(w http.ResponseWriter, r *http.Request, status int, err error) {
    if status >= 500 {
        slog.ErrorContext(r.Context(), "request failed", "err", err, "path", r.URL.Path)
        err = errors.New("internal error")   // never leak internals
    }
    encode(w, r, status, map[string]string{"error": err.Error()})
}
```

A wrapped internal error in a response body leaks file paths, SQL, and hostnames. Log the detail, return the generic message.

## Routing

`net/http`'s `ServeMux` gained method and wildcard patterns in **Go 1.22**, which covers most services without a dependency:

```go
func newRouter(logger *slog.Logger, db *sql.DB) http.Handler {
    mux := http.NewServeMux()
    svc := user.NewService(db)

    mux.Handle("POST /v1/users", handleCreateUser(svc, logger))
    mux.Handle("GET /v1/users/{id}", handleGetUser(svc))
    mux.Handle("GET /healthz", handleHealth())

    return withMiddleware(mux,
        recoverPanic(logger),
        requestID,
        logRequests(logger),
    )
}
```

Path values come from `r.PathValue("id")`. Reach for `chi` or similar only when you need route groups, per-route middleware trees, or sub-routers.

## Middleware

Middleware is `func(http.Handler) http.Handler`. Keep the signature; the ecosystem composes on it.

```go
func withMiddleware(h http.Handler, mw ...func(http.Handler) http.Handler) http.Handler {
    for i := len(mw) - 1; i >= 0; i-- {   // reverse: first listed runs first
        h = mw[i](h)
    }
    return h
}

func recoverPanic(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if rec := recover(); rec != nil {
                    logger.ErrorContext(r.Context(), "panic",
                        "panic", rec, "stack", string(debug.Stack()))
                    http.Error(w, "internal error", http.StatusInternalServerError)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}
```

Request-scoped values go in the context with an **unexported key type**:

```go
type ctxKey int
const requestIDKey ctxKey = 0

func requestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.NewString()   // stdlib uuid package as of Go 1.27
        }
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

An unexported key type makes collisions impossible — another package literally cannot construct your key.

## Testing handlers

`httptest.NewRecorder` for a single handler, `httptest.NewServer` for the full stack:

```go
func TestHandleGetUser(t *testing.T) {
    svc := user.NewService(newFakeStore(t))
    h := handleGetUser(svc)

    req := httptest.NewRequest(http.MethodGet, "/v1/users/abc", nil)
    req.SetPathValue("id", "abc")
    rec := httptest.NewRecorder()

    h.ServeHTTP(rec, req)

    if rec.Code != http.StatusOK {
        t.Fatalf("status = %d, want %d; body: %s", rec.Code, http.StatusOK, rec.Body)
    }
}
```

Go 1.27 adds `httptest.NewTestServer`, which runs on an in-memory network and works inside a `testing/synctest` bubble — that makes timeout and retry tests deterministic.

## Checklist

- [ ] `ReadHeaderTimeout` set on the server
- [ ] `Timeout` set on every client
- [ ] No `http.DefaultServeMux`, no `http.DefaultClient`
- [ ] `r.Context()` threaded into every downstream call
- [ ] Request bodies bounded with `MaxBytesReader`
- [ ] Panic recovery middleware at the top
- [ ] Graceful shutdown via `signal.NotifyContext` + `srv.Shutdown`
- [ ] Error→status mapping in exactly one function
- [ ] 5xx details logged, never returned in the body
- [ ] `/healthz` that does not depend on downstreams
