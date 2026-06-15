# Concurrency Strategy

## Problem

The system must support 5 lakh concurrent users while guaranteeing zero double-bookings.

A double-booking occurs when two users successfully reserve the same seat.

Example:

User A books Seat A1
User B books Seat A1

Both bookings succeed.

This must never happen.

---

# Option A: PostgreSQL SELECT FOR UPDATE

## How It Works

When a user selects seats:

```sql
BEGIN;

SELECT *
FROM seats
WHERE id IN (...)
FOR UPDATE;

UPDATE seats
SET status = 'held'
WHERE id IN (...);

COMMIT;