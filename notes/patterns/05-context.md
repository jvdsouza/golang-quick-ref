# Context propagation

`context.Context` carries three things down a call chain: **cancellation**, **deadlines**, and **request-scoped values**.

## Rules

```go
func (s *Service) Get(ctx context.Context, id string) (Shipment, error)   // first param, named ctx
```

- Pass it **explicitly** through every function that does I/O or might block.
- **Don't store it in a struct** (the exception is a struct that *is* one operation, like an `http.Request`).
- Never pass `nil`. Use `context.TODO()` if you're unsure, or `context.Background()` at the top level.
- Contexts are immutable. `With*` returns a **child**. Cancelling a parent cancels all its children, never the other way round.

## Where contexts come from

```go
r.Context()                 // HTTP server: cancelled when the client disconnects or ServeHTTP returns
context.Background()        // main, tests, top-level jobs
t.Context()                 // tests, Go 1.24+: cancelled when the test ends
signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)   // cancelled on Ctrl-C or SIGTERM
```

## Deadlines and cancellation

```go
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()            // ALWAYS: it releases the timer and resources (go vet checks for this)

rate, err := s.rates.Quote(ctx, from, to, kg)
```

- A child's deadline can only **shorten** the parent's. Setting `WithTimeout(ctx, 1*time.Hour)` on a context that has 3 seconds left still gives you 3 seconds.
- Budget per dependency: if the whole request gets 10s, maybe the rates API gets 2s.
- `ctx.Err()` returns `context.Canceled` (someone cancelled it) or `context.DeadlineExceeded` (time ran out).
- `context.WithCancelCause` / `WithTimeoutCause` and `context.Cause(ctx)` let you attach *why* it was cancelled.

## Passing it to everything that blocks

```go
db.QueryRowContext(ctx, q, args...)                     // database/sql
req, _ := http.NewRequestWithContext(ctx, "GET", u, nil) // outbound HTTP
conn, err := (&net.Dialer{}).DialContext(ctx, "tcp", addr)
```

Any call without `Context` in its name ignores your deadline, for example `db.Query` and `http.Get`.

## Respecting cancellation in your own loops

```go
for _, item := range batch {
    if err := ctx.Err(); err != nil {
        return err                     // stop early: the caller has gone
    }
    process(item)
}

select {
case res := <-results:
    return res, nil
case <-ctx.Done():
    return Result{}, ctx.Err()
}
```

## Fan-out with errgroup

```go
g, ctx := errgroup.WithContext(ctx)   // golang.org/x/sync/errgroup
var rate Money
var eta time.Duration

g.Go(func() error {
    var err error
    rate, err = s.rates.Quote(ctx, from, to, kg)
    return err
})
g.Go(func() error {
    var err error
    eta, err = s.eta.Estimate(ctx, from, to)
    return err
})
if err := g.Wait(); err != nil {      // first error cancels ctx for the others
    return err
}
```

## Work that outlives the request

`r.Context()` is cancelled the moment the handler returns, so background work started from a handler fails if it uses that context.

```go
// Go 1.21+: keeps the values (request ID, trace) but drops cancellation and the deadline
bg := context.WithoutCancel(r.Context())
go func() {
    ctx, cancel := context.WithTimeout(bg, 30*time.Second)   // give it its own bound
    defer cancel()
    s.audit.Record(ctx, event)
}()
```

For anything important, prefer a durable queue or an outbox table over a fire-and-forget goroutine. The goroutine is lost if the process restarts.

## Request-scoped values

**Good:** request ID, trace or span, authenticated principal, locale. That's data *about the request*.
**Bad:** DB handles, loggers you could inject, config, optional function parameters.

```go
type ctxKey struct{}               // unexported, so it can't collide with other packages

func WithUser(ctx context.Context, u User) context.Context {
    return context.WithValue(ctx, ctxKey{}, u)
}

func UserFrom(ctx context.Context) (User, bool) {
    u, ok := ctx.Value(ctxKey{}).(User)
    return u, ok
}
```

Logging tip: `log.InfoContext(ctx, ...)` with a custom `slog.Handler` can pull the request ID from `ctx` automatically.

## Graceful shutdown

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() { _ = srv.ListenAndServe() }()   // returns http.ErrServerClosed after Shutdown
<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx)   // stops accepting connections and waits for in-flight requests
```

## Be ready to explain

- What happens to a DB query when the client disconnects mid-request.
- Why a child timeout can't extend the parent's deadline.
- Why `r.Context()` is wrong for background work, and what `WithoutCancel` does.
- What belongs in context values and what doesn't.
