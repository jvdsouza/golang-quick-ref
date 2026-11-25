# Middleware

## Shape

Middleware wraps an `http.Handler` and returns a new one. It can run code before and after `next`, or skip `next` entirely.

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {   // wrap in reverse so mws[0] is outermost
        h = mws[i](h)
    }
    return h
}

mux := http.NewServeMux()
mux.HandleFunc("GET /shipments/{id}", h.getShipment)      // Go 1.22+ method and path patterns
mux.HandleFunc("POST /shipments", h.createShipment)       // read with r.PathValue("id")

handler := Chain(mux,
    RequestID,          // 1. outermost: every log line, even from a panic, gets an ID
    Logging(log),       // 2. logs every request, including recovered panics as 500s
    Recover(log),       // 3. turns panics into a 500 before Logging sees the result
    Authenticate(auth), // 4. innermost cross-cutting layer
)
```

**Order matters.** Execution flows outer → inner on the way in and inner → outer on the way out.
Apply auth per route when some routes are public: `mux.Handle("POST /shipments", Authenticate(auth)(http.HandlerFunc(h.create)))`.

## Context values: typed keys and accessors

```go
type ctxKey int
const (
    requestIDKey ctxKey = iota
    userKey
)

func RequestIDFrom(ctx context.Context) string {
    id, _ := ctx.Value(requestIDKey).(string)
    return id
}
```

An unexported key type means no other package can collide with or overwrite your keys.

## Request ID

```go
func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" || len(id) > 64 {          // don't trust arbitrary client input
            id = rand.Text()                   // Go 1.24+ crypto/rand; or a UUID library
        }
        w.Header().Set("X-Request-ID", id)
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Logging (capture the status code)

`http.ResponseWriter` doesn't expose the status code, so wrap it.

```go
type statusRecorder struct {
    http.ResponseWriter
    status int
    bytes  int
}

func (r *statusRecorder) WriteHeader(code int) {
    if r.status == 0 { r.status = code }
    r.ResponseWriter.WriteHeader(code)
}

func (r *statusRecorder) Write(b []byte) (int, error) {
    if r.status == 0 { r.status = http.StatusOK }
    n, err := r.ResponseWriter.Write(b)
    r.bytes += n
    return n, err
}

// Lets http.ResponseController reach Flush, Hijack, and so on (Go 1.20+)
func (r *statusRecorder) Unwrap() http.ResponseWriter { return r.ResponseWriter }

func Logging(log *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            rec := &statusRecorder{ResponseWriter: w}
            next.ServeHTTP(rec, r)
            log.InfoContext(r.Context(), "http request",
                "method", r.Method,
                "path", r.URL.Path,          // not RawQuery: it may contain secrets
                "status", rec.status,
                "bytes", rec.bytes,
                "duration", time.Since(start),
                "request_id", RequestIDFrom(r.Context()),
            )
        })
    }
}
```

## Recovery

```go
func Recover(log *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                rec := recover()
                if rec == nil {
                    return
                }
                if rec == http.ErrAbortHandler {   // a deliberate abort: let the server handle it
                    panic(rec)
                }
                log.ErrorContext(r.Context(), "panic",
                    "err", rec, "stack", string(debug.Stack()),
                    "request_id", RequestIDFrom(r.Context()))
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }()
            next.ServeHTTP(w, r)
        })
    }
}
```

`net/http` already recovers per connection, but it only logs and drops the connection. This middleware returns a proper 500, with a request ID in the logs.
It **doesn't** cover goroutines the handler starts. Those need their own `recover`.

## Authentication

```go
type TokenVerifier interface {
    Verify(ctx context.Context, token string) (User, error)
}

func Authenticate(v TokenVerifier) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token, ok := strings.CutPrefix(r.Header.Get("Authorization"), "Bearer ")
            if !ok || token == "" {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return                                   // don't call next
            }
            user, err := v.Verify(r.Context(), token)
            if err != nil {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }
            ctx := context.WithValue(r.Context(), userKey, user)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

- Authentication (who you are) is middleware. Authorization (can you do this to *that shipment*?) belongs in the service.
- Return 401 when the caller isn't authenticated, and 403 when they are authenticated but not allowed.

## Other common middleware

- **Timeouts**: `http.TimeoutHandler(h, 10*time.Second, "timeout")`, or set a context deadline per request.
- **Body limits**: `http.MaxBytesReader`.
- **CORS** and **rate limiting** (`golang.org/x/time/rate`).
- **Metrics and tracing**: OpenTelemetry's `otelhttp.NewHandler`.

## Be ready to explain

- Why the order is RequestID → Logging → Recover, and what breaks if Recover sits outside Logging.
- Why the status recorder exists, and why it has `Unwrap`.
- Why context keys use an unexported type.
