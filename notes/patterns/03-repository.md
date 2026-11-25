# Repository

## The idea

A repository hides **how** data is stored behind methods named for **what the business needs**.
The service depends on an interface. The `postgres` package implements it.

```go
// package freight: the consumer defines the contract
type ShipmentRepository interface {
    Get(ctx context.Context, id string) (Shipment, error)
    Create(ctx context.Context, s Shipment) error
    UpdateStatus(ctx context.Context, id string, from, to Status) error
    ListActiveByCustomer(ctx context.Context, customerID string, limit int) ([]Shipment, error)
}
```

Name methods after use cases (`ListActiveByCustomer`), not after a generic CRUD shape.

## Implementation

```go
package postgres

type ShipmentRepo struct{ db DBTX }

func NewShipmentRepo(db DBTX) *ShipmentRepo { return &ShipmentRepo{db: db} }

// DBTX is satisfied by both *sql.DB and *sql.Tx, so the same repo works inside a transaction
type DBTX interface {
    ExecContext(ctx context.Context, q string, args ...any) (sql.Result, error)
    QueryContext(ctx context.Context, q string, args ...any) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, q string, args ...any) *sql.Row
}

func (r *ShipmentRepo) Get(ctx context.Context, id string) (freight.Shipment, error) {
    var s freight.Shipment
    var notes sql.NullString                       // the nullable column stays in this layer
    err := r.db.QueryRowContext(ctx,
        `SELECT id, customer_id, status, price_cents, notes, created_at
           FROM shipments WHERE id = $1`, id,
    ).Scan(&s.ID, &s.CustomerID, &s.Status, &s.Price.Cents, &notes, &s.CreatedAt)

    if errors.Is(err, sql.ErrNoRows) {
        return freight.Shipment{}, freight.ErrNotFound   // TRANSLATE to a domain error
    }
    if err != nil {
        return freight.Shipment{}, fmt.Errorf("get shipment %s: %w", id, err)
    }
    s.Notes = notes.String
    return s, nil
}

func (r *ShipmentRepo) UpdateStatus(ctx context.Context, id string, from, to freight.Status) error {
    res, err := r.db.ExecContext(ctx,
        `UPDATE shipments SET status = $3 WHERE id = $1 AND status = $2`, id, from, to)
    if err != nil {
        return fmt.Errorf("update status: %w", err)
    }
    if n, _ := res.RowsAffected(); n == 0 {
        return freight.ErrConflict                     // wrong state, or the row is missing
    }
    return nil
}

func (r *ShipmentRepo) ListActiveByCustomer(ctx context.Context, cust string, limit int) ([]freight.Shipment, error) {
    rows, err := r.db.QueryContext(ctx, `SELECT ... WHERE customer_id = $1 AND status <> 'delivered' LIMIT $2`, cust, limit)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    var out []freight.Shipment
    for rows.Next() {
        var s freight.Shipment
        if err := rows.Scan(/* ... */); err != nil {
            return nil, err
        }
        out = append(out, s)
    }
    return out, rows.Err()                             // don't forget rows.Err()
}
```

## Rules of thumb

- **Translate errors at the boundary**: `sql.ErrNoRows` becomes `ErrNotFound`, and a unique violation (`23505`) becomes `ErrDuplicate` or `ErrConflict`. Nothing above the repo should import a driver package.
- **Keep SQL types inside**: `sql.NullString` and `pgtype.*` stay in the repo. The domain uses `string`, `*string`, or its own types.
- **Pass `ctx` through** to every `...Context` call.
- **Return domain types**, not row structs. With `sqlc`, map the generated structs to domain types in the repo.

## When it provides real value

- The service has real business logic you want to unit test with an in-memory fake.
- You want SQL in one place and not scattered across handlers.
- The storage is non-trivial or might change: caching, multiple stores, a read replica, an external API acting as the "store".

## When it doesn't

- A thin CRUD app where the "service" would just pass calls through. Calling `sqlc`-generated queries directly from the service, or even the handler, is fine.
- A generic `Repository[T]` with `FindAll`/`FindByID`/`Save`. It forces every entity into the same shape, hides query costs, and usually leaks anyway.
- Wrapping an ORM in a repository that mirrors the ORM's API. That's two abstractions doing the work of one.
- Abstracting only so you can "swap databases someday". You almost never will, and real transactional semantics differ between databases.

> Test the repository itself against a **real** database (see [testing](09-testing.md)). Faking the SQL layer proves nothing about the SQL.

## Repositories and transactions

The awkward part: an operation that touches two repos atomically. Common options:

1. Repos take a `DBTX`, and the service-side transaction helper builds tx-bound repos (see [transactions](06-transactions.md)).
2. A repo method performs the whole unit of work in one transaction, for example `CreateWithOutboxEvent`. It's simple, but some business logic drifts into the repo.

## Be ready to explain

- Why the interface lives in the service package and returns domain errors.
- A case where you'd skip the repository abstraction.
- How the same repo code runs inside and outside a transaction.
