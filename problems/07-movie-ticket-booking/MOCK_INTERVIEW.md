# Mock Interview Script — Movie Ticket Booking

**For the interviewer.**

---

## Opening

> "Design an online movie ticket booking system like BookMyShow. Users can browse movies, select a show, choose seats, and pay for tickets. Before designing, ask any clarifying questions."

| If candidate asks | Answer |
|------------------|--------|
| Multiple cities/theatres? | Yes — multiple cities, multiple theatres per city, multiple screens per theatre |
| Concurrent seat booking? | Yes — this is the core challenge. Multiple users can select same seat. |
| Seat hold duration? | 10 minutes to complete payment after seat selection |
| Multiple seats per booking? | Yes |
| Different seat types? | Regular, Premium, Recliner — with different prices (secondary) |
| Cancellation? | Secondary — focus on booking flow first |
| Payment? | Assume external PaymentGateway; mock its response |

---

## Phase 2: Core Design — What to Watch For

**The critical signal:** How does the candidate handle the seat booking race condition?

- **Level 0 (bad):** "I'll mark the seat as booked when the user selects it." (No payment step, no hold mechanism)
- **Level 1 (weak):** Mentions lock/hold but unclear implementation
- **Level 2 (good):** "Seat goes to HELD state when selected; payment must complete within 10 min; seat returns to AVAILABLE on failure or timeout"
- **Level 3 (strong):** "The transition from AVAILABLE → HELD must be atomic/synchronized per seat. Two threads cannot both successfully mark the same seat as HELD."

If candidate marks seat as booked immediately (skipping payment):
> "What happens between seat selection and when the user actually completes payment? Can another user book that seat during checkout?"

If no synchronization on seat state:
> "Two users click on seat A3 at the exact same time. Walk me through what happens in your design."

---

## Extension Probe (~35 min)

> "Add cancellation — user can cancel a booking within 2 hours for a full refund, after 2 hours for 50%."

Strong: `CancellationPolicy` Strategy interface; `FullRefundPolicy`, `PartialRefundPolicy`. `Booking` holds policy reference. Seats return to AVAILABLE on cancellation.

---

## Concurrency Probe (~40 min, if not raised)

> "1000 users try to book the last seat for a sold-out show simultaneously. What's the worst case in your design?"

Strong: 999 requests fail after attempting to transition seat from AVAILABLE→HELD. The synchronization ensures only one succeeds. Discusses optimistic locking vs synchronized block trade-off. Mentions that the seat hold expiry must also be handled (either scheduled task or lazy check on next access).

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Seat hold timeout, concurrent booking, multiple seat types, payment flow | |
| Entity Modeling | Seat with state machine (AVAILABLE/HELD/BOOKED), Booking with expiry, Show vs Screen vs Theatre hierarchy | |
| Design Patterns | State on Seat, Strategy on pricing/cancellation | |
| SOLID | BookingService as orchestrator; payment, seat, notification separated | |
| Extensibility | New seat type or pricing = new class | |
| Concurrency | Atomic AVAILABLE→HELD transition, lock expiry handling | |
| Communication | Explained the seat hold mechanism and WHY atomicity needed | |
| **TOTAL** | | **/21** |
