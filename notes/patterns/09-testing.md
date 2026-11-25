# Testing

```bash
go test ./...                       # everything
go test -race -count=1 ./...        # race detector, no cached results (use this in CI)
go test -run 'TestCreate/overweight' ./internal/freight   # one subtest
go test -cover -coverprofile=c.out ./... && go tool cover -html=c.out
go test -short ./...                # skip slow tests that check testing.Short()
```

## Table-driven tests

```go
func TestCreateShipmentInput_Validate(t *testing.T) {
    tests := []struct {
        name    string
        in      CreateShipmentInput
        wantErr map[string]string   // nil means valid
    }{
        {"valid", CreateShipmentInput{Origin: "SYD", Dest: "MEL", WeightKg: 10}, nil},
        {"missing origin", CreateShipmentInput{Dest: "MEL", WeightKg: 10},
            map[string]string{"origin": "required"}},
        {"zero weight", CreateShipmentInput{Origin: "SYD", Dest: "MEL"},
            map[string]string{"weight_kg": "must be positive"}},
    }

    for _, tt := range tests {          // Go 1.22+: no `tt := tt` needed
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            err := tt.in.Validate()

            if tt.wantErr == nil {
                if err != nil {
                    t.Fatalf("unexpected error: %v", err)
                }
                return
            }
            var ve *ValidationError
            if !errors.As(err, &ve) {
                t.Fatalf("want *ValidationError, got %T: %v", err, err)
            }
            if diff := cmp.Diff(tt.wantErr, ve.Fields); diff != "" {   // github.com/google/go-cmp
                t.Errorf("fields mismatch (-want +got):\n%s", diff)
            }
        })
    }
}
```

- `t.Fatalf` stops the current (sub)test. `t.Errorf` records a failure and keeps going.
- Good failure messages say **got** and **want**.
- Useful helpers: `t.Helper()` (reports failures at the caller's line), `t.Cleanup(fn)`, `t.TempDir()`, `t.Setenv()`, and `t.Context()` (Go 1.24+).

## Test doubles

| Double | What it is | Use when |
|---|---|---|
| **Fake** | a working, simplified implementation (an in-memory repo) | the default for repos and stores |
| **Stub** | returns canned answers | forcing a specific path, such as an error |
| **Spy** | records calls for later assertions | checking *that* something was sent (email, event) |
| **Mock** | pre-programmed expectations, verified automatically (gomock, testify/mock) | sparingly: it couples tests to implementation details |

Hand-written doubles are cheap when interfaces are small:

```go
// Fake: an in-memory repository
type fakeShipments struct {
    mu   sync.Mutex
    byID map[string]freight.Shipment
}

func (f *fakeShipments) Get(_ context.Context, id string) (freight.Shipment, error) {
    f.mu.Lock(); defer f.mu.Unlock()
    s, ok := f.byID[id]
    if !ok {
        return freight.Shipment{}, freight.ErrNotFound
    }
    return s, nil
}

// Stub with function fields: configure the behaviour per test
type stubRates struct {
    QuoteFn func(ctx context.Context, from, to string, kg float64) (freight.Money, error)
}
func (s stubRates) Quote(ctx context.Context, from, to string, kg float64) (freight.Money, error) {
    return s.QuoteFn(ctx, from, to, kg)
}

// In a test
rates := stubRates{QuoteFn: func(context.Context, string, string, float64) (freight.Money, error) {
    return freight.Money{}, errors.New("upstream down")
}}
svc := freight.NewService(&fakeShipments{byID: map[string]freight.Shipment{}}, rates, slog.New(slog.DiscardHandler))
```

`slog.DiscardHandler` is Go 1.24+. On older versions, use `slog.New(slog.NewTextHandler(io.Discard, nil))`.

## Testing handlers with `httptest`

```go
func TestGetShipment(t *testing.T) {
    svc := &fakeService{shipment: freight.Shipment{ID: "s1", Status: freight.StatusBooked}}
    h := httpapi.NewHandler(svc, slog.New(slog.DiscardHandler))

    req := httptest.NewRequest(http.MethodGet, "/shipments/s1", nil)
    req.Header.Set("Authorization", "Bearer test-token")
    rec := httptest.NewRecorder()

    h.ServeHTTP(rec, req)                    // no network, no port

    if rec.Code != http.StatusOK {
        t.Fatalf("status = %d, want 200; body: %s", rec.Code, rec.Body)
    }
    var got struct{ ID, Status string }
    if err := json.NewDecoder(rec.Body).Decode(&got); err != nil {
        t.Fatal(err)
    }
    if got.ID != "s1" {
        t.Errorf("id = %q, want s1", got.ID)
    }
}
```

Add table rows for 404 (service returns `ErrNotFound`), 422 (validation), 500 (an arbitrary error, with the body checked for **no** leaked details), and 401 (no token).

### Testing an HTTP client against a fake server

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path != "/v1/quote" {
        t.Errorf("unexpected path %s", r.URL.Path)
    }
    w.WriteHeader(http.StatusServiceUnavailable)
}))
defer srv.Close()

client := ratesapi.NewClient(srv.Client(), srv.URL)
_, err := client.Quote(t.Context(), "SYD", "MEL", 10)
// assert the error is classified as retryable or infrastructure
```

## Integration tests (real database)

Unit tests with fakes can't prove your SQL works. Test repositories against **real Postgres**.

```go
//go:build integration          // run with: go test -tags=integration ./...

func TestShipmentRepo_UpdateStatus_Conflict(t *testing.T) {
    db := testDB(t)              // helper: connect, run migrations, register cleanup
    repo := postgres.NewShipmentRepo(db)
    ctx := t.Context()

    mustCreate(t, repo, freight.Shipment{ID: "s1", Status: freight.StatusBooked})

    err := repo.UpdateStatus(ctx, "s1", freight.StatusPending, freight.StatusBooked)
    if !errors.Is(err, freight.ErrConflict) {
        t.Fatalf("want ErrConflict, got %v", err)
    }
}
```

Ways to get a database:
- **testcontainers-go**: starts a Postgres container from the test itself. It's self-contained and works the same in CI.
- **docker compose** in CI, with a `TEST_DATABASE_URL` env var. If the variable is unset, call `t.Skip`.

Ways to isolate tests:
- A **transaction per test** that's rolled back in `t.Cleanup`. It's fast, but it can't test commit or concurrency behaviour.
- A **fresh schema or database per test package**, or truncate tables between tests.
- Unique IDs per test, so parallel tests don't collide.

Integration tests are also where you test **concurrency**: fire two goroutines at the same `UpdateStatus` or idempotency key, and assert that exactly one wins.

## Other tools

- **Fuzzing**: `func FuzzParse(f *testing.F)`, run with `go test -fuzz=FuzzParse`. Good for parsers and validators.
- **Benchmarks**: `func BenchmarkX(b *testing.B) { for b.Loop() { ... } }`. `b.Loop` is Go 1.24+.
- **`testing/synctest`** (Go 1.25+): tests code that uses timers and goroutines with a fake clock, deterministically.
- **Golden files**: compare output against `testdata/*.golden`, with an `-update` flag to regenerate them.
- The `testdata/` directory is ignored by the Go tool, so it's a safe home for fixtures.

## What to test where

| Layer | Test with |
|---|---|
| domain rules, validation | plain unit tests, table-driven |
| service | unit tests with fake repos and stub clients |
| handlers | `httptest.NewRecorder` with a fake service: status codes, JSON shape, error mapping |
| repositories | integration tests against real Postgres |
| outbound clients | `httptest.NewServer` |
| the whole thing | a few end-to-end tests: real server, real DB, HTTP calls |

## Be ready to explain

- Fake vs stub vs mock, and why fakes are usually preferred.
- How `httptest.NewRecorder` differs from `httptest.NewServer`.
- How you'd keep integration tests isolated and still fast.
- How you'd test that idempotency holds under concurrent duplicate requests.
