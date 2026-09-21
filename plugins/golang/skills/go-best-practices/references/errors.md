# Errors

## The three kinds

**Opaque** — the caller only checks `err != nil`. The default, and the right choice most of the time.

```go
if err := s.save(ctx, u); err != nil {
    return fmt.Errorf("save user %s: %w", u.ID, err)
}
```

**Sentinel** — the caller checks identity. Use when there is a small, fixed set of conditions callers branch on.

```go
var ErrNotFound = errors.New("not found")

// caller
if errors.Is(err, ErrNotFound) {
    return http.StatusNotFound, nil
}
```

**Typed** — the caller needs data from the failure.

```go
type ValidationError struct {
    Field  string
    Reason string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("field %s: %s", e.Field, e.Reason)
}

// caller
var ve *ValidationError
if errors.As(err, &ve) {
    return badRequest(ve.Field, ve.Reason)
}
```

Go 1.26+ has a generic form that avoids the declared target:

```go
if ve, ok := errors.AsType[*ValidationError](err); ok {
    return badRequest(ve.Field, ve.Reason)
}
```

**Prefer opaque.** Every sentinel and every exported error type is API surface you must keep working. Add one when a caller demonstrably needs to branch, not in anticipation.

## Wrapping

`%w` makes the wrapped error reachable via `errors.Is`/`errors.As`. That is a **public commitment**: callers may depend on it, and removing it later is a breaking change.

```go
// Caller may inspect the cause — commit to it.
return fmt.Errorf("fetch user %s: %w", id, err)

// Deliberately opaque — the underlying driver error is an implementation detail.
return fmt.Errorf("fetch user %s: %v", id, err)
```

Multiple `%w` verbs in one `Errorf` are allowed (Go 1.20+), and `errors.Join` combines independent failures:

```go
var errs []error
for _, v := range validators {
    if err := v.Validate(cfg); err != nil {
        errs = append(errs, err)
    }
}
return errors.Join(errs...)  // nil if errs is empty
```

`errors.Is`/`errors.As` traverse joined errors, so a caller can still find a specific one.

## Message style

Error strings are **fragments that get concatenated**, so:

- lowercase (unless starting with a proper noun or acronym)
- no trailing punctuation
- no `"error"`, `"failed"`, `"unable to"` — the fact that it is an error is already known

```go
// Good — reads as a chain
errors.New("unexpected EOF")
fmt.Errorf("parse config %s: %w", path, err)
// → "load settings: parse config /etc/app.yaml: unexpected EOF"

// Bad
errors.New("Error: Failed to parse the configuration file.")
// → "load settings: Error: Failed to parse the configuration file.: unexpected EOF"
```

Add the context the *caller* lacks. The function that opened the file knows the path; the one that called it does not. Don't repeat what an inner layer already said.

## Handling rules

**Handle once.** Log or return, never both:

```go
// Bad — one failure, logged at every layer
if err != nil {
    log.Printf("save failed: %v", err)
    return err
}

// Good
if err != nil {
    return fmt.Errorf("save: %w", err)
}
```

The top of the stack — an HTTP handler, a `main`, a worker loop — logs once with full context.

**Ignoring requires a reason:**

```go
// Bad
_ = f.Close()

// Good
// Close on a read-only file can only fail on a double close, which we don't do.
_ = f.Close()

// Better, for writes — a failed Close means data was lost
defer func() {
    if cerr := f.Close(); cerr != nil && err == nil {
        err = fmt.Errorf("close %s: %w", path, cerr)
    }
}()
```

`defer resp.Body.Close()` is the conventional exception; nobody comments that one.

**Don't string-match:**

```go
// Bad — breaks when the message changes
if strings.Contains(err.Error(), "not found") { ... }

// Good
if errors.Is(err, ErrNotFound) { ... }
```

If a dependency gives you no sentinel and no type, wrap the string match in one function at the boundary and add a comment naming the version you tested.

## Panic and recover

`panic` is for **programmer error that cannot be recovered from**:

```go
// Legitimate — a compiled-in regex that doesn't compile is a bug, not a runtime condition
var pathRE = regexp.MustCompile(`^/v(\d+)/users/(\w+)$`)

// Not legitimate — a library panicking on caller input
func ParseDuration(s string) time.Duration {
    d, err := time.ParseDuration(s)
    if err != nil {
        panic(err) // return (time.Duration, error) instead
    }
    return d
}
```

A `recover()` at a goroutine or request boundary, to keep the process alive, is legitimate — and must log the stack:

```go
defer func() {
    if r := recover(); r != nil {
        logger.Error("panic in handler",
            "panic", r,
            "stack", string(debug.Stack()))
        http.Error(w, "internal error", http.StatusInternalServerError)
    }
}()
```

`recover()` as flow control — panicking to unwind a parser, recovering at the top — is a documented exception in the stdlib (`encoding/json` does it), but it must never cross a package boundary.

## Errors at API boundaries

Map domain errors to transport codes in **one place**:

```go
func statusFor(err error) int {
    switch {
    case errors.Is(err, ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, ErrConflict):
        return http.StatusConflict
    case errors.Is(err, context.DeadlineExceeded):
        return http.StatusGatewayTimeout
    }
    var ve *ValidationError
    if errors.As(err, &ve) {
        return http.StatusBadRequest
    }
    return http.StatusInternalServerError
}
```

Scattering `http.Error` calls through the call stack means no single place knows what the API returns. And never put a wrapped internal error in a response body — it leaks paths, queries, and hostnames.
