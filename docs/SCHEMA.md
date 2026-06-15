# Database Schema Design

CREATE TABLE venues (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    capacity INT NOT NULL CHECK (capacity > 0)
);

CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    venue_id BIGINT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
    name VARCHAR(255) NOT NULL,
    start_time TIMESTAMP NOT NULL,
    total_seats INT NOT NULL CHECK (total_seats > 0),
    status VARCHAR(20) NOT NULL
        CHECK (status IN ('upcoming','on_sale','sold_out','cancelled'))
);

CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,

    event_id BIGINT NOT NULL
        REFERENCES events(id) ON DELETE CASCADE,

    section VARCHAR(50) NOT NULL,
    row_name VARCHAR(20) NOT NULL,
    seat_number VARCHAR(20) NOT NULL,

    price NUMERIC(10,2) NOT NULL
        CHECK (price > 0),

    category VARCHAR(20) NOT NULL
        CHECK (category IN ('VIP','Premium','General')),

    status VARCHAR(20) NOT NULL
        CHECK (status IN ('available','held','booked')),

    held_until TIMESTAMP NULL,

    held_by UUID NULL
        REFERENCES users(id) ON DELETE SET NULL,

    version INT NOT NULL DEFAULT 0
);

CREATE TABLE bookings (
    id UUID PRIMARY KEY,

    user_id UUID NOT NULL
        REFERENCES users(id) ON DELETE RESTRICT,

    event_id BIGINT NOT NULL
        REFERENCES events(id) ON DELETE RESTRICT,

    status VARCHAR(20) NOT NULL
        CHECK (status IN ('pending','confirmed','failed','refunded')),

    total_amount NUMERIC(10,2) NOT NULL
        CHECK (total_amount > 0),

    payment_reference VARCHAR(255),

    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE booking_seats (
    booking_id UUID NOT NULL
        REFERENCES bookings(id) ON DELETE CASCADE,

    seat_id BIGINT NOT NULL
        REFERENCES seats(id) ON DELETE RESTRICT,

    PRIMARY KEY (booking_id, seat_id)
);

CREATE INDEX idx_seats_event_status
ON seats(event_id, status);

CREATE INDEX idx_bookings_user
ON bookings(user_id, created_at DESC);

CREATE INDEX idx_bookings_unresolved
ON bookings(status)
WHERE status IN ('pending');

## Design Decisions

### Why UUID for booking.id instead of SERIAL?

UUIDs are globally unique and prevent predictable IDs. They are safer for distributed systems and public APIs.

### Why does seats have a version column?

The version column supports optimistic locking. Every update increments the version, helping detect conflicting updates.

### Why held_until instead of application-level holds?

If the application crashes, the database still knows when the hold expires. Expired seats can be released automatically.

### Why use a partial index on bookings.status?

Most queries focus on unresolved bookings. A partial index is smaller and faster than indexing all booking records.