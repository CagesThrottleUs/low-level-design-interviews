# Lessons Learned — Movie Ticket Booking

---

## Mistake 1: Marking Seat as BOOKED on Selection (No HELD State)

**What happens:** When user clicks a seat, `seat.setStatus(BOOKED)` immediately. No payment required. Payment is an afterthought.

**Why it's wrong:** Users who never complete payment permanently block seats. Real systems hold for N minutes and release on payment timeout.

**Fix:** Three-state seat lifecycle:
```
AVAILABLE → HELD (on selection, with expiry) → BOOKED (on payment success)
                ↓ (expiry or payment failure)
           AVAILABLE
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 2: Global Show-Level Lock

**What happens:** `synchronized(show)` or one lock for the entire show when booking any seat.

**Why it's wrong:** Booking seat A1 blocks booking seat Z20. 200 seats = 200 people can never book concurrently. For popular shows with 500 concurrent users, this is a bottleneck.

**Fix:** Per-seat lock (`synchronized` on individual Seat objects):
```java
// One seat locks independently — 200 concurrent bookings possible
public synchronized boolean hold(String bookingId, Duration duration) { ... }
```

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 3: Not Releasing Seats on Partial Failure

**What happens:** User selects 3 seats (A1, A2, A3). A1 and A2 hold successfully. A3 is taken. Candidate's code: throws exception. But A1 and A2 are now permanently HELD (no release).

**Why it's wrong:** A1 and A2 are stuck in HELD state until timeout. Frustrating UX. Can be a leak if timeout is long.

**Fix:** Release all seats held so far if any seat fails:
```java
List<Seat> heldSeats = new ArrayList<>();
for (Seat seat : selectedSeats) {
    if (!seat.hold(bookingId, HOLD_DURATION)) {
        heldSeats.forEach(Seat::release); // compensate
        throw new SeatsUnavailableException();
    }
    heldSeats.add(seat);
}
```

**Gap dimension:** Concurrency / Correctness (Dimension 6, 2)

---

## Mistake 4: Expired Hold Not Handled

**What happens:** Seat held for 10 minutes, user abandons session. Seat stays HELD forever (or until manual intervention).

**Fix — lazy approach (interview-appropriate):**
```java
public synchronized boolean hold(String bookingId, Duration duration) {
    // Check if existing hold is expired — treat as available
    if (status == SeatStatus.HELD && Instant.now().isAfter(holdExpiresAt)) {
        status = SeatStatus.AVAILABLE; // lazy reset
    }
    if (status == SeatStatus.AVAILABLE) {
        // proceed with hold
    }
}
```

Mention: production uses a scheduled task to sweep expired holds.

**Gap dimension:** Concurrency (Dimension 6)

---

## Mistake 5: No Payment Failure Handling

**What happens:** `paymentGateway.charge()` called, no check on result. Seats stay HELD regardless of payment outcome.

**Fix:**
```java
if (paymentResult.isSuccess()) {
    seats.forEach(s -> s.confirm(bookingId)); // HELD → BOOKED
} else {
    seats.forEach(Seat::release); // HELD → AVAILABLE
    throw new PaymentFailedException();
}
```

**Gap dimension:** Requirements / Correctness (Dimensions 1, 2)

---

## Problem-Specific Gotchas

- Theatre → Screen → Show hierarchy is different from Seat (Seat belongs to Screen, not Show directly — the Show *uses* the Screen's seat map)
- `Booking.expiresAt` enables frontend countdown timer — good detail to mention
- Multiple seats per booking: hold all or hold none (atomic group hold)
- `PaymentGateway` should be an interface — never instantiate Stripe directly in BookingService

---

## Self-Assessment Checklist

- [ ] Is seat state machine AVAILABLE → HELD → BOOKED (not just 2 states)?
- [ ] Is `hold()` synchronized per seat (not per show)?
- [ ] Are all held seats released if any seat in the group fails to hold?
- [ ] Are held seats released on payment failure?
- [ ] Is expired hold handled (lazy check inside `hold()`)?
- [ ] Is `PaymentGateway` an interface?
- [ ] Is Theatre → Screen → Show hierarchy correct?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Entity Modeling | No HELD state, wrong hierarchy | GAP_REMEDIATION.md#gap-2 — noun-verb extraction |
| Design Patterns | No pricing strategy | GAP_REMEDIATION.md#gap-3 |
| Concurrency | Global show lock, no expiry handling | GAP_REMEDIATION.md#gap-6 — shared state audit |
| Extensibility | No cancellation policy interface | GAP_REMEDIATION.md#gap-5 — extension challenge |
