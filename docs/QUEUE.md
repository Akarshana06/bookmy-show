# QUEUE.md

# Asynchronous Order Processing Flow

## Overview

To handle heavy traffic (up to **500,000 concurrent users**) without overloading the booking service, payment processing is performed asynchronously using **Amazon SQS (Simple Queue Service)**.

The API creates a pending booking and immediately places a message in the queue. A separate payment worker processes the payment and updates the booking status.

---

# 1. Why Asynchronous Processing?

If the Booking API waits for the payment gateway to respond, database connections remain occupied for hundreds of milliseconds, reducing system capacity.

Assumptions:

* Database connection pool = **500**
* 80% normal requests = **20 ms**
* 20% payment requests = **800 ms**

```
Connections Held =
(RPS × 80% × 0.02)
+
(RPS × 20% × 0.80)

= 0.176 × RPS
```

At around **2,840 requests per second**, the connection pool would be exhausted.

Using an asynchronous queue allows the API to respond quickly while payment processing happens independently, improving scalability and user experience.

---

# 2. API Flow

```
User
   │
   ▼
Booking API
   │
   ▼
Create Pending Booking
   │
   ▼
Send Message to Amazon SQS
   │
   ▼
Payment Worker
   │
 ┌─┴─────────────┐
 │               │
 ▼               ▼
Payment OK   Payment Failed
 │               │
 ▼               ▼
Confirm Seat  Release Seat
```

---

# 3. SQS Message Format

Each queue message contains all information required for payment processing.

```json
{
  "bookingId": "3c52d7a8-1234-5678-9012",
  "userId": "8a11b2c3-4567-8901-2345",
  "eventId": 501,
  "seatIds": ["A1", "A2"],
  "totalAmount": 5000,
  "paymentToken": "tok_xyz123",
  "idempotencyKey": "booking-3c52d7a8"
}
```

### Field Descriptions

| Field            | Purpose                                               |
| ---------------- | ----------------------------------------------------- |
| `bookingId`      | Identifies the booking record to update.              |
| `userId`         | Identifies the customer making the purchase.          |
| `eventId`        | Specifies the event for which tickets are booked.     |
| `seatIds`        | Lists the seats included in the booking.              |
| `totalAmount`    | Total payment amount to be charged.                   |
| `paymentToken`   | Secure token used for payment authorization.          |
| `idempotencyKey` | Prevents duplicate charges if the message is retried. |

---

# 4. Payment Worker Logic

## Success Path

1. Read the message from Amazon SQS.
2. Verify that the booking is still in `pending` status.
3. Send the payment request to the payment gateway.
4. If payment succeeds:

   * Update booking status to `confirmed`.
   * Mark seats as `booked`.
   * Save the payment reference.
5. Acknowledge and remove the message from the queue.

---

## Failure Path

1. Read the message from Amazon SQS.
2. Payment fails or is declined.
3. Update booking status to `failed`.
4. Change reserved seats back to `available`.
5. Remove the message from the queue after processing.

---

# 5. Dead Letter Queue (DLQ)

If processing repeatedly fails, the message is automatically moved to a Dead Letter Queue for manual investigation.

This prevents problematic messages from blocking normal queue processing.

---

# 6. Edge Cases

## Case 1: Server crashes after publishing to SQS but before responding to the user

* The booking already exists with status `pending`.
* The payment worker continues processing the queued message.
* Even if the client retries, the `idempotencyKey` prevents duplicate payment processing.
* The user can later check the booking status.

---

## Case 2: Payment gateway times out

* The worker does not immediately mark the booking as failed.
* The message becomes visible again after the visibility timeout.
* The worker retries processing.
* After the configured retry limit is exceeded, the message is sent to the Dead Letter Queue.

---

# 7. SQS Configuration

## Visibility Timeout

**60 seconds**

Reason:

* Long enough for the payment worker to complete processing.
* Prevents multiple workers from processing the same message simultaneously.

## Maximum Receive Count

**5 retries**

Reason:

* Temporary failures often resolve after a few attempts.
* Messages that continue failing are automatically routed to the Dead Letter Queue for investigation.

---

# 8. Benefits of This Design

* Improves API responsiveness during traffic spikes.
* Prevents database connection exhaustion.
* Supports automatic retries for transient failures.
* Uses idempotency to avoid duplicate payments.
* Isolates failed messages through a Dead Letter Queue.
* Ensures seats are confirmed only after successful payment and released on failure.
