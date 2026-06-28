# Solution Guide — Movie Ticket Booking

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `Movie` | Concrete | Title, genre, duration, rating |
| `Theatre` | Concrete | Name, city, list of screens |
| `Screen` | Concrete | Screen number; seat layout definition |
| `Show` | Concrete | Movie + Screen + DateTime; manages all seat states for that show |
| `Seat` | Concrete | Row, column, SeatType; holds current SeatStatus; park/unpark ops |
| `SeatStatus` | Enum | AVAILABLE, HELD, BOOKED |
| `Booking` | Concrete | Show, seats, customer, paymentStatus, holdExpiresAt |
| `PricingStrategy` | Interface | `calculatePrice(show, seat): long` |
| `StandardPricing` | Concrete | Base price per seat type |
| `WeekendSurgePricing` | Concrete | Decorator or strategy that adds weekend surcharge |
| `BookingService` | Concrete | Orchestrates: hold seats → process payment → confirm or release |
| `PaymentGateway` | Interface | `charge(amount, paymentMethod): PaymentResult` |

---

## Seat State Machine

```
[AVAILABLE] ──holdSeat()──> [HELD] (with expiry timestamp)
[HELD] ──confirmBooking()──> [BOOKED]
[HELD] ──(expiry reached)──> [AVAILABLE]
[HELD] ──releaseSeat()──> [AVAILABLE]
[BOOKED] ──cancel()──> [AVAILABLE]
```

---

## The Critical Design: Atomic Seat Hold

This is the entire point of the interview. Two users click the same seat simultaneously.

```java
class Seat {
    private SeatStatus status = SeatStatus.AVAILABLE;
    private Instant holdExpiresAt;
    private String heldByBookingId;

    // Synchronized on the seat object — not a global lock
    public synchronized boolean hold(String bookingId, Duration holdDuration) {
        if (status == SeatStatus.AVAILABLE ||
            (status == SeatStatus.HELD && Instant.now().isAfter(holdExpiresAt))) {
            // Second condition handles expired holds — treat as available
            this.status = SeatStatus.HELD;
            this.heldByBookingId = bookingId;
            this.holdExpiresAt = Instant.now().plus(holdDuration);
            return true;
        }
        return false; // seat is held by someone else or already booked
    }

    public synchronized boolean confirm(String bookingId) {
        if (status == SeatStatus.HELD && heldByBookingId.equals(bookingId)) {
            status = SeatStatus.BOOKED;
            return true;
        }
        return false;
    }

    public synchronized void release() {
        this.status = SeatStatus.AVAILABLE;
        this.heldByBookingId = null;
        this.holdExpiresAt = null;
    }
}
```

**Why per-seat lock (not per-show lock):** A show can have 200 seats. A global show lock means Seat A1 booking blocks Seat Z20 booking — terrible throughput for popular shows. Per-seat synchronization allows 200 concurrent bookings, one per seat.

---

## Booking Flow

```java
class BookingService {
    private final PaymentGateway paymentGateway;
    private final Duration HOLD_DURATION = Duration.ofMinutes(10);

    public Booking initiate(Show show, List<Seat> selectedSeats, Customer customer) {
        List<Seat> heldSeats = new ArrayList<>();
        String bookingId = UUID.randomUUID().toString();

        for (Seat seat : selectedSeats) {
            if (!seat.hold(bookingId, HOLD_DURATION)) {
                // At least one seat couldn't be held — release all already held
                heldSeats.forEach(Seat::release);
                throw new SeatsUnavailableException("One or more seats no longer available");
            }
            heldSeats.add(seat);
        }

        return new Booking(bookingId, show, heldSeats, customer,
                          BookingStatus.PAYMENT_PENDING,
                          Instant.now().plus(HOLD_DURATION));
    }

    public Booking confirmPayment(Booking booking, PaymentMethod paymentMethod) {
        long totalAmount = calculateTotal(booking);
        PaymentResult result = paymentGateway.charge(totalAmount, paymentMethod);

        if (result.isSuccess()) {
            booking.getSeats().forEach(s -> s.confirm(booking.getId()));
            booking.setStatus(BookingStatus.CONFIRMED);
            return booking;
        } else {
            booking.getSeats().forEach(Seat::release);
            booking.setStatus(BookingStatus.PAYMENT_FAILED);
            throw new PaymentFailedException(result.getReason());
        }
    }
}
```

---

## Expired Hold Handling

Two approaches:

**Approach 1 — Lazy (simpler for interview):**
Check `holdExpiresAt` inside `seat.hold()` — treat expired holds as available. No background thread needed.

**Approach 2 — Eager (production):**
`ScheduledExecutorService` scans for expired bookings every minute and calls `seat.release()`.

For interview: state both, implement Approach 1.

---

## Pricing Strategy

```java
interface PricingStrategy {
    long calculatePriceCents(Show show, Seat seat);
}

class StandardPricing implements PricingStrategy {
    public long calculatePriceCents(Show show, Seat seat) {
        return switch (seat.getType()) {
            case REGULAR -> 25000;   // $250.00
            case PREMIUM -> 40000;   // $400.00
            case RECLINER -> 60000;  // $600.00
        };
    }
}

class WeekendSurgePricing implements PricingStrategy {
    private final PricingStrategy base;
    public WeekendSurgePricing(PricingStrategy base) { this.base = base; }

    public long calculatePriceCents(Show show, Seat seat) {
        long basePrice = base.calculatePriceCents(show, seat);
        DayOfWeek day = show.getDateTime().getDayOfWeek();
        if (day == SATURDAY || day == SUNDAY) {
            return (long)(basePrice * 1.2); // 20% weekend surcharge
        }
        return basePrice;
    }
}
```

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| Cancellation with refund policy | `CancellationPolicy` Strategy; seat.release() on cancel |
| New seat type (VIP) | New enum value + pricing rule — only pricing class changes |
| Weekend surcharge | Decorator/Strategy wrapping standard pricing |
| Discount codes | `DiscountStrategy` applied before final charge |
| Waitlist | Observer: when seat released, notify waitlisted users |

---

## What Strong Candidates Do Differently

- Sync at seat level (not show level) for throughput
- Lazy expired-hold check inside `hold()` — elegant, no background thread
- Release ALL held seats if ANY seat in multi-seat booking fails
- Payment gateway as interface (not hardcoded payment provider)
- `Booking` has an `expiresAt` field — client can show countdown timer

## What Average Candidates Miss

- Mark seat as BOOKED on selection (skip HELD state entirely — wrong)
- Global show-level lock — correct but poor throughput
- Don't release seats if multi-seat booking partially fails
- No payment flow modeled — booking completes without payment
- Hold expiry never handled — seats can be stuck in HELD forever
