# Concurrency

## The rule that prevents most bugs

**Never start a goroutine without knowing how it stops.** Before writing `go`, answer three questions:

1. Who waits for it?
2. How is it cancelled?
3. Where does its error go?

A bare `go f()` in a long-lived process answers none of them. That is a leak, and the leak is invisible until the process runs out of memory or a test suite starts flaking.

## Ownership patterns

### errgroup — several tasks, first error wins

```go
func (s *Server) Run(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)
    g.Go(func() error { return s.serveHTTP(ctx) })
    g.Go(func() error { return s.pollUpstream(ctx) })
    g.Go(func() error { return s.flushMetrics(ctx) })
    return g.Wait()
}
```

`errgroup.WithContext` cancels the derived context when any goroutine returns a non-nil error, so the others unwind. `g.Wait()` returns the first error. Use `g.SetLimit(n)` to bound concurrency.

### sync.WaitGroup — fan out, wait, no errors

Go 1.25+ has `WaitGroup.Go`, which removes the `Add`/`defer Done` pair:

```go
var wg sync.WaitGroup
for _, item := range items {
    wg.Go(func() {       // loop var is per-iteration since Go 1.22
        process(ctx, item)
    })
}
wg.Wait()
```

Pre-1.25:

```go
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func() {
        defer wg.Done()
        process(ctx, item)
    }()
}
wg.Wait()
```

`wg.Add` goes **before** the `go`, never inside the goroutine — inside, `Wait` can return before the counter is incremented.

### Long-running loop — select on ctx.Done

```go
func (w *Worker) Run(ctx context.Context) error {
    t := time.NewTicker(w.interval)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-t.C:
            if err := w.tick(ctx); err != nil {
                w.log.ErrorContext(ctx, "tick failed", "err", err)
            }
        }
    }
}
```

Note the tick error is logged, not returned — one failed tick should not kill the worker. That is a decision to make deliberately, not by accident.

### Bounded worker pool

```go
func processAll(ctx context.Context, items []Item, workers int) error {
    g, ctx := errgroup.WithContext(ctx)
    ch := make(chan Item)

    g.Go(func() error {
        defer close(ch)              // sender closes
        for _, it := range items {
            select {
            case ch <- it:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })

    for range workers {              // Go 1.22+ range-over-int
        g.Go(func() error {
            for it := range ch {
                if err := process(ctx, it); err != nil {
                    return fmt.Errorf("process %s: %w", it.ID, err)
                }
            }
            return nil
        })
    }
    return g.Wait()
}
```

Simpler alternative when the work is independent: `g.SetLimit(workers)` and one `g.Go` per item, no channel at all.

## Channels

| Rule | Why |
|---|---|
| The **sender** closes | Closing from the receiver races with other senders; a second close panics |
| **Unbuffered by default** | A buffer's capacity should have a meaning you can state |
| Never close a receive-only channel | It is a compile error, and the signature documents intent — use `<-chan T` in parameters |
| A `nil` channel blocks forever | Useful: set a case's channel to `nil` to disable it in a `select` |
| Don't use a channel as a mutex | `sync.Mutex` is clearer and faster for guarding a field |

Signal completion by **closing**, not by sending — closing broadcasts to every receiver:

```go
type Service struct{ done chan struct{} }

func (s *Service) Stop()            { close(s.done) }
func (s *Service) Done() <-chan struct{} { return s.done }
```

Guard against double close with `sync.Once`.

## Mutexes

```go
type Cache struct {
    mu    sync.Mutex           // unexported, above what it guards
    items map[string]Item      // guarded by mu
}

func (c *Cache) Get(k string) (Item, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    it, ok := c.items[k]
    return it, ok
}
```

- **Never embed `sync.Mutex` in an exported struct.** `type Cache struct { sync.Mutex }` puts `Lock`/`Unlock` in your public API and lets any caller deadlock you.
- **Declare the mutex immediately above the fields it guards**, with a comment when it is not obvious.
- **Any type containing a mutex needs pointer receivers.** Copying a locked mutex is a bug; `go vet` catches most instances.
- **`RWMutex` is not a free upgrade.** It is slower than `Mutex` under write contention and only pays off with many concurrent readers and rare writes. Measure.
- **Don't call unknown code while holding a lock** — a callback that re-enters your type deadlocks.

## Context and cancellation

```go
func (s *Server) handle(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()                     // always, even on the success path

    result, err := s.store.Query(ctx, r.URL.Query().Get("q"))
    ...
}
```

- `defer cancel()` always. Skipping it leaks the timer and the context until the parent is cancelled.
- Check `ctx.Err()` in loops that do many iterations of work.
- `ctx.Done()` in a `select` is how a blocking operation becomes cancellable.
- **The caller owns the deadline.** Don't shorten a passed-in context unless that is your documented job.

## Testing concurrency

**`-race` in CI, always.** It is the single highest-value concurrency tool Go has.

**`testing/synctest`** (GA in Go 1.25) gives a virtual clock inside a bubble: time jumps forward when every goroutine is blocked, so a test of a one-hour timeout runs instantly and deterministically.

```go
func TestRetryBackoff(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        c := NewClient(WithRetries(3))
        go c.Do(ctx, req)

        synctest.Wait()           // all goroutines blocked
        // assert on state after the first attempt
    })
}
```

Go 1.27 adds `synctest.Sleep`, combining `time.Sleep` and `synctest.Wait`.

**Never `time.Sleep` to synchronize a test.** It is slow when it works and flaky when it doesn't. Wait on a channel, or use synctest.

**Goroutine leak detection:** Go 1.26 added an experimental `goroutineleak` pprof profile (`GOEXPERIMENT=goroutineleakprofile`), generally available in 1.27 — it reports goroutines permanently blocked on unreachable channels, mutexes, or conditions. `go.uber.org/goleak` remains the common test-time option.

## Common mistakes

| Mistake | Fix |
|---|---|
| `wg.Add(1)` inside the goroutine | Move it before `go` |
| Sending on a closed channel | The sender closes, and only once |
| `defer` inside a loop | Extract the body into a function |
| Goroutine writing to a map | Mutex, or a channel that funnels writes to one owner |
| Ignoring `g.Wait()`'s error | Return it |
| `context.Background()` deep in a call stack | Thread the real `ctx` through |
| Unbuffered result channel with no reader | The sender blocks forever — buffer it or ensure a receiver |
| `time.After` in a hot `select` loop | Leaks a timer per iteration; use `time.NewTimer` and `Reset` |

## Loop variables (Go 1.22+)

Since **Go 1.22**, `for` loop variables are per-iteration. The old shadowing workaround is no longer needed:

```go
// Go 1.22+ — correct as written
for _, item := range items {
    go func() { process(item) }()
}

// Pre-1.22 required
for _, item := range items {
    item := item
    go func() { process(item) }()
}
```

Check `go.mod` before relying on this — the behavior is gated on the module's declared Go version, not the toolchain's.
