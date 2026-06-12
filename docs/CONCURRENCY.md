# CONCURRENCY.md

# Concurrency Strategy – ShowTime Ticket Booking System

## Objective

The primary goal of the concurrency strategy is to ensure that **no seat is booked by more than one user**, even when approximately **500,000 users** attempt to purchase tickets simultaneously.

---

# Option A: PostgreSQL `SELECT ... FOR UPDATE`

## How It Works

When a user selects a seat, the application starts a database transaction and locks the corresponding seat row.

Example:

```sql
BEGIN;

SELECT *
FROM seats
WHERE id = 101
FOR UPDATE;

UPDATE seats
SET status = 'held',
    held_by = 'user-123'
WHERE id = 101;

COMMIT;
```

The `FOR UPDATE` clause places an exclusive lock on the selected row. Any other transaction attempting to lock the same seat must wait until the first transaction completes.

## How It Prevents Double Booking

* The first transaction acquires the row lock.
* Other transactions cannot modify the same seat until the lock is released.
* Only one transaction succeeds in changing the seat status from `available` to `held` or `booked`.

This guarantees transactional consistency and prevents duplicate seat assignments.

## Connection Pool Capacity

Assume:

* Maximum database connections: **500**
* 80% regular requests with an average execution time of **20 ms**
* 20% payment requests with an average execution time of **800 ms**

Formula:

```
Connections Held =
(RPS × 80% × 0.02)
+
(RPS × 20% × 0.80)
```

```
Connections Held =
0.016 × RPS + 0.16 × RPS
= 0.176 × RPS
```

Pool exhaustion occurs when:

```
0.176 × RPS = 500

RPS ≈ 2840
```

Therefore, the connection pool would become saturated at approximately **2,840 requests per second**.

## Deadlock Risk

Deadlocks may occur when users reserve multiple seats in different orders.

Example:

* User A locks Seat A then waits for Seat B.
* User B locks Seat B then waits for Seat A.

### Mitigation

Always acquire locks in a deterministic order (such as ascending `seat_id`) before updating records. This significantly reduces deadlock probability.

---

# Option B: Redis `SETNX` Distributed Lock

## How It Works

Before reserving a seat, the application creates a Redis lock.

Example key:

```
seat_lock:{event_id}:{seat_id}
```

Redis command:

```
SET seat_lock:501:101 user123 NX EX 30
```

* `NX` ensures the key is created only if it does not already exist.
* `EX 30` automatically expires the lock after 30 seconds.

If the command succeeds, the user obtains the lock. Otherwise, another user already holds it.

## Lock TTL

Recommended TTL: **30 seconds**

* Too short: the lock may expire before payment finishes.
* Too long: seats remain unavailable unnecessarily if a user abandons checkout.

Thirty seconds provides a reasonable balance.

## Redis Failure Scenario

If Redis crashes after issuing a lock but before the booking is finalized, lock information may be lost.

Therefore, Redis alone cannot guarantee consistency. The database must still verify seat availability before confirming the booking.

---

# Comparison

| Feature                     | PostgreSQL `SELECT FOR UPDATE` | Redis `SETNX`                          |
| --------------------------- | ------------------------------ | -------------------------------------- |
| Prevents double booking     | Yes                            | Yes (when combined with DB validation) |
| Transactional guarantee     | Strong                         | Requires additional safeguards         |
| Performance                 | Moderate                       | Very high                              |
| Infrastructure complexity   | Low                            | Medium                                 |
| Deadlock possibility        | Yes                            | No row-level deadlocks                 |
| Additional service required | No                             | Yes (Redis cluster)                    |
| Cost                        | Lower                          | Higher                                 |

---

# Chosen Strategy

## Decision

We choose **PostgreSQL `SELECT FOR UPDATE`** as the primary concurrency mechanism.

## Justification

* It provides strong ACID guarantees and reliable row-level locking.
* It completely prevents simultaneous updates to the same seat.
* It avoids introducing additional infrastructure complexity.
* It fits well within the **$2,000/month AWS budget** while maintaining correctness.

Although Redis locks can improve throughput, they add operational complexity and still require database validation to guarantee correctness.

## Limitations

`SELECT FOR UPDATE` may become a bottleneck under extremely high request rates because transactions occupy database connections until completion.

## When We Would Switch

If the platform scales significantly beyond the current workload (for example, tens of thousands of booking requests per second across many events), we would adopt a **hybrid strategy**:

* Use **Redis `SETNX`** for fast distributed locking.
* Use **PostgreSQL transactions** as the final source of truth before confirming bookings.

This approach combines high performance with strong consistency while continuing to prevent double bookings.
