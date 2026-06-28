# Movie Ticket Booking (BookMyShow)

**Difficulty:** Intermediate
**Category:** Concurrent Booking + State Management
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** State (seat/booking lifecycle), Strategy (seat selection, pricing)
**Asked at:** Amazon, Uber, Flipkart, BookMyShow, Paytm, Swiggy

---

## Problem Statement

Design an online movie ticket booking system. Users can browse movies, select shows, choose seats, and make a payment. The system must handle concurrent users trying to book the same seat — only one user should successfully book any given seat.

This is the most concurrency-sensitive LLD problem; the seat reservation race condition is the crux of the interview.

---

## Actors

| Actor | Description |
|-------|-------------|
| Customer | Browses movies, selects show + seats, pays, receives ticket |
| Theatre Admin | Adds movies, shows, seats (out of scope for MVP) |
| Payment System | External — processes payment (mocked in interview) |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Customer can browse movies showing in a city
- FR2: Customer can view available shows for a movie (date, time, theatre)
- FR3: Customer can view seat map for a show — which seats are available, booked, or temporarily held
- FR4: Customer can select seats and initiate booking
- FR5: Selected seats are temporarily locked for N minutes while customer pays
- FR6: On successful payment, seats are permanently booked; tickets issued
- FR7: If payment fails or lock expires, seats are released back to available

**Secondary (implement if time allows):**
- FR8: Multiple seat types — regular, premium, recliner — with different prices
- FR9: Cancellation with partial refund policy
- FR10: Discount codes / offers
- FR11: Seat map visual representation (row A–J, column 1–20)

---

## Non-Functional Requirements

- **Concurrency:** Two users can select the same seat simultaneously — exactly one succeeds
- **Consistency:** A seat must never be double-booked
- **Lock expiry:** Held seats auto-release after lock timeout (e.g., 10 minutes)
- **Scale:** Hundreds of concurrent users per popular show

---

## Constraints and Assumptions

- One screen per show; seats are fixed and identified by row+column
- Payment is external — assume a `PaymentGateway.charge()` that returns success/failure
- Lock duration: 10 minutes (configurable)
- In-memory is fine — no real DB needed for interview
- Out of scope: recommendation engine, waitlist, dynamic pricing surge

---

## Good Clarifying Questions to Ask

1. How many seats per show? How many concurrent users expected?
2. How long is the seat hold/lock when a user starts payment?
3. What happens if the payment gateway is slow or fails?
4. Can a user book multiple seats in one transaction?
5. Do we need cancellations?
6. Different seat types with different prices?
7. Is it one theatre or multiple theatres/cities?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **Movie** — title, genre, duration, rating
- **Theatre** — name, city, list of screens
- **Screen** — screen number; holds seat map for a show
- **Show** — Movie + Screen + DateTime; manages all seat bookings for that show
- **Seat** — row, column, SeatType; state machine (Available → Held → Booked → Available)
- **SeatStatus** — enum: AVAILABLE, HELD, BOOKED
- **Booking** — seats, show, customer, payment status, expiry time
- **Payment** — booking reference, amount, status
- **PricingStrategy** — interface; pricing varies by seat type and show time
- **BookingService** — orchestrates seat lock, payment, confirmation; handles lock expiry

</details>
