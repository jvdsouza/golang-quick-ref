# Service layer

## The idea

Business rules live in a layer that knows **nothing about HTTP or SQL**.
Handlers translate HTTP into plain function calls. Repositories translate those calls into SQL. The service decides **what is allowed to happen**.

```
HTTP / gRPC / CLI / queue consumer      (transport: decode, auth, map errors to status codes)
            │  plain Go types
            ▼
      freight.Service                   (business rules, orchestration, transaction boundaries)
            │  interfaces
            ▼
  postgres repo · rates API client      (infrastructure)
```

Dependencies point **inward**: `httpapi` imports `freight`, `postgres` imports `freight`, and `freight` imports neither.

## Package layout (one common shape)

```
cmd/api/main.go            wiring
internal/freight/          domain types, service, errors, interfaces it needs
internal/postgres/         repository implementations
internal/httpapi/          handlers, middleware, request/response DTOs
internal/ratesapi/         external API client
```

## Service: domain in, domain out

```go
package freight

type CreateShipmentInput struct {
    CustomerID string
    Origin     string
    Dest       string
    WeightKg   float64
}

func (s *Service) CreateShipment(ctx context.Context, in CreateShipmentInput) (Shipment, error) {
    if err := in.Validate(); err != nil {
        return Shipment{}, err                       // a *ValidationError
    }
    if in.WeightKg > MaxWeightKg {
        return Shipment{}, ErrOverweight             // a business rule
    }

    price, err := s.rates.Quote(ctx, in.Origin, in.Dest, in.WeightKg)
    if err != nil {
        return Shipment{}, fmt.Errorf("quote rate: %w", err)
    }

    sh := Shipment{
        ID:         newID(),
        CustomerID: in.CustomerID,
        Status:     StatusPending,
        Price:      price,
        CreatedAt:  s.now(),
    }
    if err := s.repo.Create(ctx, sh); err != nil {
        return Shipment{}, fmt.Errorf("save shipment: %w", err)
    }
    return sh, nil
}
```

Notice what's **absent**: `*http.Request`, `http.ResponseWriter`, status codes, `*sql.Tx`, `sql.NullString`, and JSON tags.

## Handler: thin translation only

```go
package httpapi

type createShipmentRequest struct {      // the DTO: the wire format, separate from the domain type
    Origin   string  `json:"origin"`
    Dest     string  `json:"destination"`
    WeightKg float64 `json:"weight_kg"`
}

func (h *Handler) createShipment(w http.ResponseWriter, r *http.Request) {
    var req createShipmentRequest
    if err := decodeJSON(w, r, &req); err != nil {
        h.writeError(w, r, err)
        return
    }
    sh, err := h.svc.CreateShipment(r.Context(), freight.CreateShipmentInput{
        CustomerID: auth.UserID(r.Context()),   // from middleware
        Origin:     req.Origin,
        Dest:       req.Dest,
        WeightKg:   req.WeightKg,
    })
    if err != nil {
        h.writeError(w, r, err)               // one place maps domain errors to status codes
        return
    }
    writeJSON(w, http.StatusCreated, toShipmentResponse(sh))
}
```

## Why bother

- **Testable**: test business rules with a fake repo. No HTTP server, no database.
- **Reusable**: the same `CreateShipment` can serve HTTP, a CLI, a queue consumer, or a cron job.
- **Swappable edges**: change the JSON shape or the DB schema without touching the rules.
- **One home for rules**: no logic duplicated across handlers or hidden in SQL triggers.

## Where things go

| Concern | Layer |
|---|---|
| JSON shape, field names, status codes | handler |
| "Is this well-formed?" (required, format, range) | input `Validate()` in the service package, called by the service |
| "Is this allowed?" (state transitions, limits, ownership) | service |
| Transaction boundaries | service (see [transactions](06-transactions.md)) |
| SQL, scanning, driver errors | repository |
| Authentication (who are you?) | middleware |
| Authorization (can you do *this*?) | service, because it's a business rule |

## Pitfalls

- **Pass-through services** that only call `repo.X`. That's fine for simple reads, but if every method is pass-through, question whether you need the layer.
- **Leaking transport into the service**: taking `*http.Request` or returning `int` status codes.
- **Leaking persistence up**: returning `sql.ErrNoRows` or `*pgconn.PgError` to handlers. Translate it in the repo.
- **Anemic domain**: put behaviour on domain types where it fits, for example `func (s Shipment) CanCancel() bool`.
- **God service**: split by business capability (`shipments`, `billing`, `customers`), not by technical layer.

## Be ready to explain

- The dependency direction, and why `freight` must not import `net/http`.
- The difference between a DTO and a domain type, and why to keep both.
- Where authorization belongs, and why that's not the middleware.
