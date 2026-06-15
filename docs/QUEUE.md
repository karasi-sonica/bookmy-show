# Async Order Processing Flow

## Section 1: Why Async?

At peak traffic, 5 lakh users may attempt bookings simultaneously.

Payment gateways typically take between 1 and 5 seconds to respond.

If payment processing is synchronous, database connections remain occupied while waiting for payment responses.

Formula:

Connections Held =
(% Payment RPS × Payment Hold Time)
+
(% Non-Payment RPS × Query Time)

Assume:

- Payment requests = 20%
- Payment duration = 0.8 seconds
- Normal requests = 80%
- Query duration = 0.02 seconds

500 = RPS × [(0.2 × 0.8) + (0.8 × 0.02)]

500 = RPS × 0.176

RPS ≈ 2840

The database pool exhausts quickly under heavy load.

Using SQS allows the API to return quickly while payment processing happens asynchronously.

---

# Section 2: SQS Message Format

```json
{
  "bookingId": "uuid",
  "userId": "uuid",
  "eventId": 101,
  "seatIds": [1, 2, 3],
  "totalAmount": 5000,
  "paymentToken": "token123",
  "idempotencyKey": "uuid"
}
```

## Field Explanation

bookingId
- Unique booking identifier.

userId
- User who created the booking.

eventId
- Event being booked.

seatIds
- Seats included in the booking.

totalAmount
- Amount to charge.

paymentToken
- Token received from payment gateway.

idempotencyKey
- Prevents duplicate payment processing.

---

# Section 3: Worker Logic

## Success Path

1. Receive message from SQS.
2. Validate idempotency key.
3. Process payment.
4. Update booking status to CONFIRMED.
5. Update seat status to BOOKED.
6. Store payment reference.
7. Delete message from queue.

## Failure Path

1. Receive message.
2. Payment fails.
3. Update booking status to FAILED.
4. Release held seats.
5. Delete message from queue.

## DLQ Path

1. Message fails repeatedly.
2. Message reaches maximum receive count.
3. SQS routes message to Dead Letter Queue.
4. Engineering team investigates manually.

---

# Section 4: Edge Cases

## Server Crashes After SQS Publish But Before API Responds

What happens?

- Booking already exists.
- Message already exists in SQS.
- Worker still processes payment.

User Experience:

- User may see timeout/error.
- Refreshing booking history shows booking status.

No booking is lost.

---

## Payment Gateway Timeout

Gateway returns neither success nor failure.

Worker action:

1. Retry request.
2. Check payment status using idempotency key.
3. Avoid duplicate charges.
4. If retries exhausted, move to DLQ.

---

# Section 5: SQS Configuration

## Visibility Timeout

60 seconds

Reason:

- Worker normally completes in a few seconds.
- Allows retries if worker crashes.
- Prevents multiple workers processing the same message simultaneously.

## Maximum Receive Count

5

Reason:

- Temporary failures often recover after retry.
- Prevents infinite processing loops.
- Sends problematic messages to DLQ for investigation.

---

# Final Flow

1. User selects seats.
2. Seats marked HELD.
3. Booking created with PENDING status.
4. Message published to SQS.
5. API responds immediately.
6. Worker processes payment.
7. Success → CONFIRMED + BOOKED.
8. Failure → FAILED + Seats Released.