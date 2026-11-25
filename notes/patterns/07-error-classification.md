# Error classification

## Three kinds of error

| Kind | Examples | Whose fault | HTTP | Log | Retry? |
|---|---|---|---|---|---|
| **Validation** | missing field, bad format, weight < 0 | client | 400 / 422 | debug or info | no, fix the input |
| **Business / domain** | not found, already cancelled, over credit limit, forbidden | nobody: the rules said no | 404 / 409 / 403 / 422 | info | no (409 maybe after re-reading) |
| **Infrastructure** | DB down, timeout, upstream 5xx, bug | server | 500 / 502 / 503 / 504 | **error**, with details | often yes |

The client gets a **specific, safe** message for the first two kinds, and a **generic** message for infrastructure errors. Log internal details, but never send them to the client.

## Defining them in the domain package

```go
package freight

// Business errors: sentinels for conditions with no extra data
var (
    ErrNotFound          = errors.New("not found")
    ErrConflict          = errors.New("conflict")
    ErrForbidden         = errors.New("forbidden")
    ErrInvalidTransition = errors.New("invalid status transition")
    ErrOverweight        = errors.New("shipment exceeds maximum weight")
)

// Validation errors: a type, because they carry per-field detail
type ValidationError struct {
    Fields map[string]string   // field -> problem
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: %v", e.Fields)
}

func (in CreateShipmentInput) Validate() error {
    f := map[string]string{}
    if in.Origin == ""  { f["origin"] = "required" }
    if in.WeightKg <= 0 { f["weight_kg"] = "must be positive" }
    if len(f) > 0 {
        return &ValidationError{Fields: f}
    }
    return nil
}
```

**Infrastructure errors** don't need defining. They're whatever comes back from the driver or client, **wrapped** with context: `fmt.Errorf("save shipment: %w", err)`.

## Translating at the boundaries

```
postgres repo:   sql.ErrNoRows          -> freight.ErrNotFound
                 unique violation 23505 -> freight.ErrConflict
                 anything else          -> wrapped as-is (infrastructure)

rates client:    404 from upstream      -> freight.ErrRouteNotServed (business)
                 5xx / timeout          -> wrapped (infrastructure)
```

Anything **not** recognised as validation or business is treated as **infrastructure** by default. That's the safe fallback.

## Mapping to HTTP in one place

```go
func (h *Handler) writeError(w http.ResponseWriter, r *http.Request, err error) {
    var ve *freight.ValidationError
    var mbe *http.MaxBytesError
    var se *json.SyntaxError

    switch {
    case errors.As(err, &ve):
        writeJSON(w, 422, problem{Title: "validation failed", Fields: ve.Fields})
    case errors.As(err, &se), errors.As(err, &mbe):
        writeJSON(w, 400, problem{Title: "malformed request body"})
    case errors.Is(err, freight.ErrNotFound):
        writeJSON(w, 404, problem{Title: "not found"})
    case errors.Is(err, freight.ErrForbidden):
        writeJSON(w, 403, problem{Title: "forbidden"})
    case errors.Is(err, freight.ErrConflict), errors.Is(err, freight.ErrInvalidTransition):
        writeJSON(w, 409, problem{Title: err.Error()})
    case errors.Is(err, context.Canceled):
        // The client went away. Nobody is reading the response, so log at debug.
        h.log.DebugContext(r.Context(), "client canceled", "err", err)
    case errors.Is(err, context.DeadlineExceeded):
        h.log.ErrorContext(r.Context(), "timeout", "err", err)
        writeJSON(w, 504, problem{Title: "timeout"})
    default:
        // Infrastructure or unknown: log everything, reveal nothing
        h.log.ErrorContext(r.Context(), "internal error",
            "err", err, "request_id", RequestIDFrom(r.Context()))
        writeJSON(w, 500, problem{
            Title:     "internal server error",
            RequestID: RequestIDFrom(r.Context()),   // the user can quote it to support
        })
    }
}

// Loosely follows RFC 9457 "problem details"
type problem struct {
    Title     string            `json:"title"`
    Fields    map[string]string `json:"fields,omitempty"`
    RequestID string            `json:"request_id,omitempty"`
}
```

Be careful with `errors.Is(err, freight.ErrConflict)` followed by `err.Error()`. The wrapped message may include internal context such as `"save shipment: conflict"`. For client-facing text, either use a fixed string per case or make sure business errors are created at the top of the service without wrapping.

## Alternative: an error "kind" struct

This approach is common in larger codebases, inspired by upspin.

```go
type Kind int
const (
    KindInternal Kind = iota
    KindValidation
    KindNotFound
    KindConflict
)

type Error struct {
    Kind Kind
    Msg  string   // safe to show the client
    Err  error    // underlying cause, for logs only
}
func (e *Error) Error() string { return e.Msg }
func (e *Error) Unwrap() error { return e.Err }

// The handler then maps just the kind: errors.As(err, &e), then switch on e.Kind
```

It's more uniform, but it takes more ceremony to create each error. Sentinels plus a couple of types are enough for most services.

## Retryability (for callers and workers)

- **Retryable**: timeouts, connection resets, 503, 429 (honour `Retry-After`), serialization failures.
- **Not retryable**: validation, not found, forbidden, most 4xx.
- Retrying a non-idempotent operation is only safe with [idempotency](08-idempotency.md) protection.

## Pitfalls

- Returning `err.Error()` from infrastructure errors to clients. That leaks table names, hosts, and SQL.
- Logging the same error at every layer. Wrap it on the way up, and log once at the edge.
- Comparing with `==`, which breaks as soon as anything wraps the error. Use `errors.Is`.
- Treating a not-found from a **dependency** you expected to exist as a 404 to the client. That's often a 500, because your own data is inconsistent.

## Be ready to explain

- How the handler decides between 422, 409, and 500.
- Where `sql.ErrNoRows` gets translated, and why there.
- Why the default case must be "internal".
