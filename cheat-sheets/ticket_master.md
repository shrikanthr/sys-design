# Ticketmaster / High Contention Booking — System Design Cheat Sheet

## Core Problem

- Heavy write contention on scarce inventory. Not a fan-out problem — a serialization problem.
- Two distinct product modes with different read/write strategies: reserved seating vs. general admission.
- Thundering herd: 500K users hitting buy simultaneously for 50K seats.
- Distributed saga: booking spans payment processor, tickets table, bookings table — all need to stay consistent.

---

## Booking Flow — Saga Pattern

1. User selects seat → acquire TTL lock on seat (Redis or DB `valid_until`)
2. Initiate payment with payment processor (async)
3. Payment processor webhooks back with success/failure
4. On success: atomic commit — mark seat as booked + insert into bookings table + clear lock
5. On failure: release lock, notify user, trigger compensating transaction to void payment if partially captured

### TTL Lock — 5 minutes
- Long enough for user to complete payment flow.
- Short enough that abandoned holds don't tie up inventory.
- Too short → payment failures mid-flow. Too long → scalpers hold speculatively.

---

## Locking Mechanism — DB vs Redis

### DB-based TTL (recommended for simplicity)
- Add `valid_until` timestamp column to tickets table.
- Availability query: `WHERE status = 'available' AND (valid_until IS NULL OR valid_until < NOW())`
- Lock acquisition and booking commit are atomic — single transaction, no split-brain risk.
- Downside: `valid_until` filter on every read needs a good index, gets hammered at scale.

### Redis TTL lock
- Faster hot path. Key: `lock:{ticket_id}`, value: `user_session_id`, TTL: 5 min.
- Risk: Redis goes down between lock acquisition and webhook return → lock lost → potential double-book.
- DB is the final arbiter on commit regardless — Redis is just the fast hold layer.

**Either is defensible. DB-based is simpler and safer. Redis is faster. Know the trade-off.**

---

## Thundering Herd — Virtual Waiting Room

500K users hitting buy for 50K seats. Don't let them all hit the booking system.

### Without waiting room
- First 50K win, rest hammer refresh → self-inflicted DDoS, brutal UX, 10K+ reads/sec.

### With waiting room (correct answer)
- Demand exceeds inventory threshold → incoming users enter a queue, never touch booking system.
- Queue issues tokens in order. User sees "you are #12,847 in line."
- When a hold expires or booking fails, next token in queue gets admitted.
- Bounds concurrent active users on seat selection screen — maybe 5K instead of 500K.
- Eliminates thundering herd on both write path and read path.

---

## Read Path — Two Modes

### Reserved seating (exact seat selection)
- User picks seat 14C from a seat map. Needs accurate per-seat availability.
- Cache the seat map with short TTL (5–10 seconds). Stale display is acceptable.
- Conflict is caught at lock acquisition, not display layer. User sees "seat just taken, pick another."
- Don't skip cache entirely — 10s TTL dramatically reduces DB reads with minimal UX impact.

### General admission / section-based
- "200 seats left in floor section." Approximate is fine.
- Cache aggressively at section/zone level. 30-second stale is totally acceptable.
- Never cache at per-seat level for GA — precision doesn't matter, inventory moves fast.

**Cache availability at section/zone level, not seat level for high-volume events.**

---

## Partitioning Strategy

- Partition tickets by event, then by section/row within event.
- Improves write throughput — contention is localized to section, not entire event.
- Read trade-off: availability query fans out to all partitions. Mitigated by:
  - Section-level caching for GA events
  - Waiting room bounding concurrent readers for reserved seating

---

## Webhook Never Returns — Reconciliation Job

- Payment processor webhooks after 5 min → TTL lock already expired → seat may be re-sold.
- Background reconciliation job runs continuously:
  - Finds bookings in `PENDING` state older than TTL
  - Calls payment processor to check status
  - Completes booking if payment succeeded, voids if failed
  - Releases hold and notifies user on void
- This is the compensating transaction cleanup. Name it explicitly in interviews.

---

## Schema

```
tickets table
  ticket_id       PK
  event_id        FK, partition key
  section, row, seat
  status          enum(available, held, booked)
  valid_until     timestamp nullable
  held_by         user_session_id nullable

bookings table
  booking_id      PK
  ticket_id       FK
  user_id         FK
  payment_id
  status          enum(pending, confirmed, failed)
  created_at
```

---

## Key Data Stores

| Store | Purpose | Notes |
|---|---|---|
| Primary DB (Postgres) | Tickets, bookings, source of truth | Row-level locking on commit |
| Redis | TTL locks (optional), waiting room queue | Fast hold path, not durable |
| Cache (Redis) | Section-level availability, seat maps | Short TTL 5–30s depending on mode |
| Message queue | Payment webhook processing | Async, durable |

---

## Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Payment webhook never arrives | Seat held indefinitely | Reconciliation job checks PENDING bookings past TTL |
| Redis lock lost mid-payment | Seat could be re-sold | DB is final arbiter on commit — conflict caught at write |
| Double-book attempt | Two users commit same seat | DB atomic transaction + row version check |
| Thundering herd on sale start | Booking service overwhelmed | Virtual waiting room — queue users before they hit booking |
| Seat shown available, already taken | User selects stale seat | Caught at lock acquisition, prompt user to reselect |
| Reconciliation voids successful payment | User charged but no ticket | Check payment processor before voiding — never void on uncertainty |

---

## IC5 Talking Points

- Name the pattern: this is a **saga** with compensating transactions. Shows distributed systems vocabulary.
- DB-based TTL vs Redis lock trade-off — interviewers love this. Know both, defend your choice.
- Virtual waiting room is the mature answer. Most candidates say "rate limit it" — that's not enough.
- Two product modes (reserved vs GA) with different caching strategies shows product thinking.
- Reconciliation job for webhook failures — this is what separates someone who has built payments from someone who hasn't.
- Conflict is caught at lock acquisition, not the display layer. Display can be eventually consistent.
