# BookMyShow Architecture Assignment

## Constraints Analysis

### Constraint 1: 5 Lakh Concurrent Users

Expected users: 500,000

Assuming each user makes 4 API calls:

- View events
- View seat map
- Select seats
- Create booking

Total requests = 500,000 × 4 = 2,000,000

Peak RPS ≈ 2,000,000 / 60 = 33,333 requests/second

The database and seat-booking service are likely to become bottlenecks first.

---

### Constraint 2: Zero Double Bookings

A double booking occurs when two users successfully book the same seat.

Example:

- User A books Seat A1
- User B also books Seat A1

To prevent this, we need locking and transactional consistency.

---

### Constraint 3: $2,000 AWS Budget

The architecture must use cost-efficient services:

- PostgreSQL (RDS)
- Redis (ElastiCache)
- SQS
- EC2 instances

The solution must balance scalability with cost and avoid expensive multi-region deployments.