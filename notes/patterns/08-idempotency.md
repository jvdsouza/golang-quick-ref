# Idempotency

## The problem

A client calls `POST /shipments`. The server creates the shipment, but the response is lost (timeout, dropped connection, load balancer retry).
The client retries, and now there are **two shipments and two charges**.

Networks force at-least-once delivery. Idempotency makes "at least once" behave like "exactly once" **for business effects**.

## What's already idempotent

| Operation | Idempotent? |
|---|---|
| `GET`, `HEAD` | yes (no effects) |
| `PUT /shipments/42 {status: booked}` (set to a value) | yes |
| `DELETE /shipments/42` | yes (the second call returns 404 or 204, but has no further effect) |
| `POST /shipments` (create) | **no** |
| `POST /accounts/7/debit {amount: 30}` (relative change) | **no** |

Design toward "set state" over "apply delta" where you can.

## Technique 1: idempotency keys (for client-initiated creates)

The client generates a unique key (a UUID) **per logical operation** and sends it on every retry:

```
POST /shipments
Idempotency-Key: 9b2f6c1e-...
```

The server stores the key alongside the result, **in the same transaction as the business effect**:

```sql
CREATE TABLE idempotency_keys (
    customer_id  text        NOT NULL,
    key          text        NOT NULL,
    request_hash text        NOT NULL,     -- detects the same key reused with a different body
    shipment_id  text,                     -- the result (or store the response code and body)
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (customer_id, key)         -- scope keys per customer
);
```

```go
func (s *Service) CreateShipment(ctx context.Context, key string, in CreateShipmentInput) (Shipment, bool, error) {
    hash := hashInput(in)          // e.g. sha256 of canonical JSON
    var out Shipment
    replayed := false

    err := s.tx.InTx(ctx, func(ctx context.Context, r Repos) error {
        // INSERT ... ON CONFLICT (customer_id, key) DO NOTHING
        // If another tx holds the same key, this blocks until that tx commits or rolls back.
        claimed, err := r.Idem.Claim(ctx, in.CustomerID, key, hash)
        if err != nil {
            return err
        }

        if !claimed {
            rec, err := r.Idem.Get(ctx, in.CustomerID, key)
            if err != nil {
                return err
            }
            if rec.RequestHash != hash {
                return ErrIdempotencyKeyReused          // 422: same key, different request
            }
            out, err = r.Shipments.Get(ctx, rec.ShipmentID)
            replayed = true
            return err                                  // return the ORIGINAL result
        }

        out = newShipment(in, s.now())
        if err := r.Shipments.Create(ctx, out); err != nil {
            return err
        }
        return r.Idem.SetResult(ctx, in.CustomerID, key, out.ID)
    })
    return out, replayed, err
}
```

- Same key and same body: return the original result (the handler can respond `201` again, or `200` with `Idempotent-Replayed: true`).
- Same key and different body: `422`. The client has a bug.
- If the transaction rolls back, the key row rolls back too, so a retry starts fresh. That's correct.
- Expire keys after 24 to 72 hours with a cleanup job.
- Which operations require a key is an API design decision. Payment-like operations should.

### When the effect isn't in your database

Charging a card through a payment provider can't share your DB transaction. Instead:
1. Record the key with `status = 'started'` and commit.
2. Call the provider, **passing it an idempotency key derived from yours**. Stripe and most payment APIs support this.
3. Record the outcome. On retry, a `started` record means "resume or check with the provider", not "start again".

## Technique 2: natural keys and unique constraints

If the request contains something naturally unique, let the database enforce it:

```sql
ALTER TABLE shipments ADD CONSTRAINT uniq_customer_ref UNIQUE (customer_id, external_ref);

INSERT INTO shipments (...) VALUES (...)
ON CONFLICT (customer_id, external_ref) DO NOTHING
RETURNING id;               -- no row returned means it already exists, so SELECT it and return it
```

## Technique 3: conditional state transitions

```sql
UPDATE shipments SET status = 'booked', booked_at = now()
 WHERE id = $1 AND status = 'pending';
```

The first call books the shipment. A retry matches 0 rows. Then check the current status: if it's already `booked`, return success. Anything else is a real conflict.

## Technique 4: deduplicating message consumers

Queues (SQS, Kafka, NATS, an outbox worker) deliver **at least once**.

```go
func (c *Consumer) Handle(ctx context.Context, msg Message) error {
    return c.tx.InTx(ctx, func(ctx context.Context, r Repos) error {
        // INSERT INTO processed_messages (id) VALUES ($1) ON CONFLICT DO NOTHING
        fresh, err := r.Processed.Mark(ctx, msg.ID)
        if err != nil {
            return err
        }
        if !fresh {
            return nil                     // already handled: ack and move on
        }
        return applyEffect(ctx, r, msg)    // same tx as the dedupe mark
    })
}
```

## Client side: retry safely

- Only retry operations that are idempotent **or** carry an idempotency key.
- Use exponential backoff **with jitter**, and cap the number of attempts.
- Reuse the **same** key on every retry of the same logical operation, and generate a new key for a new operation.

## Pitfalls

- Checking "does it exist?" and then inserting, in two steps. Two concurrent retries both pass the check. Use a unique constraint or `ON CONFLICT`.
- Storing the idempotency key in a **separate** transaction from the effect. A crash between the two leaves them out of sync.
- Treating a timeout as a failure and creating a new key. Unknown outcome means retry with the **same** key.
- Side effects (email, webhooks) fired directly from the request path. Route them through an outbox, with consumers that deduplicate.

## Be ready to explain

- Why retries are unavoidable, and why "just don't retry" isn't an answer.
- The full idempotency-key flow, including concurrent duplicates and mismatched bodies.
- How you'd make a queue consumer safe under redelivery.
