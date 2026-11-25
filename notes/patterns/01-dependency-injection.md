# Dependency injection

The examples across these pattern sheets use one domain: a `freight` service that creates and manages **shipments**.

## The idea

A type gets everything it depends on **passed in**, usually through its constructor. It never reaches out to globals, `init()`, or singletons.
In Go this is just function arguments. You don't need a framework.

```go
package freight

type Service struct {
    repo  ShipmentRepository   // interface: persistence
    rates RateProvider         // interface: external pricing API
    now   func() time.Time     // injected clock, so tests can control time
    log   *slog.Logger
}

func NewService(repo ShipmentRepository, rates RateProvider, log *slog.Logger) *Service {
    return &Service{repo: repo, rates: rates, now: time.Now, log: log}
}
```

Compare that with the hidden-dependency version:

```go
var db *sql.DB   // package-level global, set in init() or main

func CreateShipment(...) { db.ExecContext(...) }   // untestable without a real DB,
                                                    // and order-of-init bugs
```

## Wire everything in `main` (the "composition root")

All construction happens in one place. Everything below it just receives what it needs.

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    if err := run(ctx, os.Getenv); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func run(ctx context.Context, getenv func(string) string) error {
    log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    db, err := sql.Open("pgx", getenv("DATABASE_URL"))
    if err != nil {
        return fmt.Errorf("open db: %w", err)
    }
    defer db.Close()                 // main owns lifecycles

    repo  := postgres.NewShipmentRepo(db)
    rates := ratesapi.NewClient(&http.Client{Timeout: 5 * time.Second}, getenv("RATES_URL"))
    svc   := freight.NewService(repo, rates, log)
    h     := httpapi.NewHandler(svc, log)

    srv := &http.Server{Addr: ":8080", Handler: h, ReadHeaderTimeout: 5 * time.Second}
    // ... start srv, wait for ctx.Done(), call srv.Shutdown
    return nil
}
```

`run` returns an error instead of calling `log.Fatal`. That lets deferred cleanup run and makes `run` testable. Injecting `getenv` means tests don't need to touch the real environment.

## Interfaces: define them where they're consumed

```go
// package freight: only the methods the service actually calls
type ShipmentRepository interface {
    Get(ctx context.Context, id string) (Shipment, error)
    Create(ctx context.Context, s Shipment) error
}

type RateProvider interface {
    Quote(ctx context.Context, from, to string, kg float64) (Money, error)
}
```

- The `postgres` package returns a concrete `*ShipmentRepo` and doesn't import or mention `freight.ShipmentRepository`.
- Keep interfaces small, so a test fake is around 10 lines.
- Don't inject an interface when there's one implementation and no test need. A concrete `*slog.Logger` is fine.

## Many dependencies or optional config

```go
// A deps struct keeps the call sites readable when there are many dependencies
type Deps struct {
    Repo  ShipmentRepository
    Rates RateProvider
    Log   *slog.Logger
}
func NewService(d Deps) *Service { ... }

// Functional options for optional settings with sensible defaults
type Option func(*Service)
func WithClock(now func() time.Time) Option { return func(s *Service) { s.now = now } }

func NewService(repo ShipmentRepository, rates RateProvider, opts ...Option) *Service {
    s := &Service{repo: repo, rates: rates, now: time.Now, log: slog.Default()}
    for _, o := range opts { o(s) }
    return s
}
```

If a constructor needs more than about 6 dependencies, the type is probably doing too much.

## Things worth injecting that people forget

- **Time** (`func() time.Time`): expiry logic, timestamps in tests
- **ID generation** (`func() string`): deterministic IDs in tests
- **`*http.Client`**: timeouts, and test servers via `httptest`
- **Config values**: pass the values in, don't read env vars deep in the code
- **Randomness** (`*rand.Rand`)

## Anti-patterns

- **Service locator**: passing a `*Container` and calling `c.Get("db")`. The dependencies are hidden again.
- **Dependencies in `context.Context`**: context is for request-scoped values, not for your DB handle.
- **`init()` with side effects**: opening connections or reading config during package import.
- **DI frameworks by default**: `google/wire` (code generation) and `uber/fx` (reflection) exist, but hand wiring in `main` scales a long way and stays greppable.

## Be ready to explain

- Why constructor injection makes code testable, using a fake repository as the example.
- Where wiring lives and why `main` owns `Close()`.
- Why the interface is declared in `freight` and not in `postgres`.
