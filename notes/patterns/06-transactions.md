# Transactions

## Basics with `database/sql`

```go
tx, err := db.BeginTx(ctx, nil)            // nil means the default isolation level
if err != nil {
    return err
}
defer tx.Rollback()                         // no-op after a successful Commit (returns ErrTxDone, ignored)

if _, err := tx.ExecContext(ctx, `UPDATE ...`); err != nil {
    return err                              // the deferred Rollback undoes everything
}
if _, err := tx.ExecContext(ctx, `INSERT ...`); err != nil {
    return err
}
return tx.Commit()
```

- **Every** statement in the unit of work must go through `tx`, not `db`. Using `db` by accident runs the statement outside the transaction, on another connection. This is a classic bug.
- If `ctx` is cancelled, `database/sql` rolls the transaction back.
- Keep transactions **short**. They hold locks and a pooled connection.

## A helper to keep it tidy

```go
func WithTx(ctx context.Context, db *sql.DB, fn func(tx *sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin: %w", err)
    }
    defer tx.Rollback()
    if err := fn(tx); err != nil {
        return err
    }
    return tx.Commit()
}
```

## Who owns the transaction? The service.

The **service** knows what must be atomic. Repos shouldn't each open their own transactions.

```go
// package freight
type Repos struct {
    Shipments ShipmentRepository
    Ledger    LedgerRepository
}

type TxRunner interface {
    // Runs fn in a transaction with repos bound to it. Commits if fn returns nil.
    InTx(ctx context.Context, fn func(ctx context.Context, r Repos) error) error
}

func (s *Service) BookShipment(ctx context.Context, id string) error {
    return s.tx.InTx(ctx, func(ctx context.Context, r Repos) error {
        sh, err := r.Shipments.GetForUpdate(ctx, id)
        if err != nil {
            return err
        }
        if sh.Status != StatusPending {
            return ErrInvalidTransition
        }
        if err := r.Shipments.UpdateStatus(ctx, id, StatusPending, StatusBooked); err != nil {
            return err
        }
        return r.Ledger.Debit(ctx, sh.CustomerID, sh.Price)
    })
}

// package postgres: repos take a DBTX (satisfied by both *sql.DB and *sql.Tx)
func (t *TxRunner) InTx(ctx context.Context, fn func(context.Context, freight.Repos) error) error {
    return WithTx(ctx, t.db, func(tx *sql.Tx) error {
        return fn(ctx, freight.Repos{
            Shipments: NewShipmentRepo(tx),
            Ledger:    NewLedgerRepo(tx),
        })
    })
}
```

In unit tests, a fake `TxRunner` just calls `fn` with in-memory repos.

## Concurrent updates: the lost update problem

Two requests both read `balance = 100`, both subtract 30, and both write `70`. One debit vanishes.

### 1. Atomic single statement (best when it fits)

```sql
UPDATE accounts SET balance = balance - $1
 WHERE id = $2 AND balance >= $1          -- the check and the write happen in one atomic step
```
If `RowsAffected() == 0`, return `ErrInsufficientFunds` (or not found).

### 2. Pessimistic locking: `SELECT ... FOR UPDATE`

```sql
SELECT status, price_cents FROM shipments WHERE id = $1 FOR UPDATE
```
This locks the row until commit or rollback, so other writers wait. Use it for read-decide-write logic too complex for a single statement.
To avoid deadlocks, lock multiple rows in a **consistent order** (for example, by ID).

### 3. Optimistic concurrency: a version column

```sql
UPDATE shipments SET status = $3, version = version + 1
 WHERE id = $1 AND version = $2
```
If 0 rows changed, someone else updated it first. Return `ErrConflict`, then either retry (re-read and re-apply) or send 409 to the client. This works well when conflicts are rare and across HTTP round trips (ETag / `If-Match`).

### 4. Conditional state transitions

```sql
UPDATE shipments SET status = 'cancelled' WHERE id = $1 AND status IN ('pending','booked')
```
The `WHERE` clause enforces the state machine atomically.

### 5. Constraints as concurrency control

`UNIQUE`, `CHECK (balance >= 0)`, and foreign keys are enforced by the database even under races. Catch the violation (Postgres `23505`, `23514`) and translate it to a domain error.

## Isolation levels (Postgres)

| Level | Notes |
|---|---|
| READ COMMITTED (default) | Each statement sees data committed before it started. Lost updates are possible with read-then-write in Go code. |
| REPEATABLE READ | One snapshot per transaction. Concurrent writes to the same row fail with a serialization error. |
| SERIALIZABLE | Behaves as if transactions ran one at a time. **You must retry** on SQLSTATE `40001`. |

```go
tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
```

## Retrying a transaction

```go
func isRetryable(err error) bool {
    var pgErr *pgconn.PgError              // github.com/jackc/pgx/v5/pgconn
    return errors.As(err, &pgErr) &&
        (pgErr.Code == "40001" || pgErr.Code == "40P01") // serialization failure, deadlock
}

for attempt := 0; ; attempt++ {
    err := s.tx.InTx(ctx, fn)
    if err == nil || !isRetryable(err) || attempt == 3 {
        return err
    }
    time.Sleep(time.Duration(attempt+1) * 50 * time.Millisecond)  // backoff; add jitter in real code
}
```

Retry the **whole** function, re-reading state. Never retry just the failed statement.

## Don't put external side effects inside a transaction

```go
InTx(ctx, func(...) error {
    repo.Create(...)
    payments.Charge(...)   // a slow call holding locks, and it CAN'T be rolled back
    email.Send(...)        // sent even if Commit fails afterwards
})
```

Instead, use the **transactional outbox**: insert an `outbox` row in the same transaction, then a separate worker publishes or sends it after commit. Combine that with [idempotency](08-idempotency.md), because the worker may send twice.

## Be ready to explain

- Why `defer tx.Rollback()` is safe after `Commit`.
- Four ways to prevent lost updates, and when you'd choose each.
- Why SERIALIZABLE requires retry logic.
- Why the outbox pattern exists.
