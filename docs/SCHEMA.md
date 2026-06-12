# SCHEMA.md

# PostgreSQL Database Schema – ShowTime Ticketing System

This document defines the database schema for the ShowTime ticket booking platform. The schema is designed to support high concurrency, prevent double bookings, and provide efficient query performance through constraints and indexing.

---

## 1. Venues Table

```sql
CREATE TABLE venues (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    capacity INTEGER NOT NULL CHECK (capacity > 0)
);

CREATE INDEX idx_venues_city
ON venues(city);
```

---

## 2. Events Table

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    venue_id BIGINT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
    name VARCHAR(255) NOT NULL,
    start_time TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL CHECK (
        status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled')
    ),
    total_seats INTEGER NOT NULL CHECK (total_seats > 0)
);

CREATE INDEX idx_events_status
ON events(status);

CREATE INDEX idx_events_start_time
ON events(start_time);
```

---

## 3. Users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email
ON users(email);
```

---

## 4. Seats Table

```sql
CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE CASCADE,

    section VARCHAR(50) NOT NULL,
    row_name VARCHAR(20) NOT NULL,
    seat_number VARCHAR(20) NOT NULL,

    price DECIMAL(10,2) NOT NULL CHECK (price > 0),

    category VARCHAR(20) NOT NULL CHECK (
        category IN ('VIP', 'Premium', 'General')
    ),

    status VARCHAR(20) NOT NULL CHECK (
        status IN ('available', 'held', 'booked')
    ),

    held_until TIMESTAMP NULL,
    held_by UUID NULL REFERENCES users(id) ON DELETE SET NULL,

    version INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_seats_event_status
ON seats(event_id, status);

CREATE INDEX idx_seats_category
ON seats(category);
```

---

## 5. Bookings Table

```sql
CREATE TABLE bookings (
    id UUID PRIMARY KEY,

    user_id UUID NOT NULL
        REFERENCES users(id)
        ON DELETE CASCADE,

    event_id BIGINT NOT NULL
        REFERENCES events(id)
        ON DELETE RESTRICT,

    status VARCHAR(20) NOT NULL CHECK (
        status IN (
            'pending',
            'confirmed',
            'failed',
            'refunded'
        )
    ),

    total_amount DECIMAL(10,2)
        NOT NULL CHECK (total_amount > 0),

    payment_reference VARCHAR(255),

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_bookings_user
ON bookings(user_id, created_at DESC);

CREATE INDEX idx_bookings_pending
ON bookings(status)
WHERE status IN ('pending', 'failed');
```

---

## 6. Booking_Seats Table

```sql
CREATE TABLE booking_seats (

    booking_id UUID NOT NULL
        REFERENCES bookings(id)
        ON DELETE CASCADE,

    seat_id BIGINT NOT NULL
        REFERENCES seats(id)
        ON DELETE RESTRICT,

    PRIMARY KEY (booking_id, seat_id),

    UNIQUE(seat_id)
);

CREATE INDEX idx_booking_seats_booking
ON booking_seats(booking_id);

CREATE INDEX idx_booking_seats_seat
ON booking_seats(seat_id);
```

---

# Design Decisions

## Why use UUID for `bookings.id` instead of SERIAL?

UUIDs are globally unique and difficult to guess, making them more secure for public-facing APIs. They also support distributed systems where multiple services may generate booking IDs independently without collisions.

---

## Why does the `seats` table have a `version` column?

The `version` column enables optimistic locking. Each update increments the version, allowing concurrent transactions to detect conflicts and avoid overwriting each other's changes.

---

## Why store `held_until` in the database?

Temporary seat holds should survive application restarts and be visible to all servers. Storing the expiration timestamp in PostgreSQL allows expired holds to be released reliably and prevents seats from remaining locked indefinitely.

---

## Why use a partial index on `bookings.status`?

Most queries for operational workflows focus on unresolved bookings (such as `pending` or `failed`). A partial index is smaller and faster than indexing every row, reducing storage overhead and improving lookup performance.
