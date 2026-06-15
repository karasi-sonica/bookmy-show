# Cache Design

## Cache Strategy

We use the Cache-Aside Pattern.

Flow:

1. Read from Redis.
2. If cache hit → return data.
3. If cache miss → query PostgreSQL.
4. Store result in Redis with TTL.
5. Return response.

When data changes:

1. Update PostgreSQL (source of truth).
2. Delete related Redis keys.
3. Next read repopulates cache.

---

# Cache Entry 1: Event Details

## Key

event:{event_id}

Example:

event:101

## Data Cached

- Event name
- Venue
- Date
- Status

## TTL

5 minutes

## Why?

Event information changes rarely.

## Invalidation Trigger

- Event updated
- Event cancelled
- Event status changes

## Expiry Method

Event-driven invalidation + TTL

---

# Cache Entry 2: Seat Availability Count

## Key

availability:{event_id}:{category}

Examples:

availability:101:VIP

availability:101:Premium

availability:101:General

## Data Cached

Available seat count per category.

## TTL

30 seconds

## Why?

Seat availability changes frequently during ticket sales.

Short TTL reduces stale data.

## Invalidation Trigger

Any seat status change:

- available → held
- held → booked
- booked → refunded
- held → available

## Expiry Method

Event-driven invalidation

---

# Cache Entry 3: Seat Map Layout

## Key

seatmap:{event_id}

Example:

seatmap:101

## Data Cached

- Sections
- Rows
- Seat arrangement

## TTL

24 hours

## Why?

Seat layout rarely changes.

## Invalidation Trigger

Venue layout modification.

## Expiry Method

TTL + Manual Invalidation

---

# What We Will NOT Cache

## Individual Seat Status

Example:

seat:101:A1

Reason:

Seat status changes constantly.

Caching it may show a seat as available when it has already been booked.

This creates a risk of overselling and poor user experience.

PostgreSQL remains the source of truth for individual seat availability.

---

# Invalidation Strategy

We use Cache-Aside Invalidation.

Pseudo Flow:

When seat status changes:

1. Update PostgreSQL
2. Delete Redis key:

availability:{event_id}:{category}

3. Next request:
   - Cache miss
   - Read from PostgreSQL
   - Rebuild cache
   - Store in Redis

This ensures consistency while keeping Redis simple.