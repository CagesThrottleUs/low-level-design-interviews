# Parking Lot

**Difficulty:** Foundation
**Category:** Multi-Entity System
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Strategy (pricing), Factory (spot/vehicle creation), Observer (notifications)
**Asked at:** Amazon, Microsoft, Adobe, Flipkart, Uber

---

## Problem Statement

Design a parking lot management system. The parking lot has multiple floors, each with multiple spots of different types. Vehicles of different sizes can enter, park, and exit. When a vehicle exits, a fee is calculated based on time parked and vehicle type.

The system must handle real-time spot availability, issue parking tickets on entry, and process payment on exit.

---

## Actors

| Actor | Description |
|-------|-------------|
| Driver | Enters lot, parks vehicle, pays on exit |
| Attendant | Operates entry/exit gates (optional: can be automated) |
| Parking Lot System | Manages spots, tickets, and billing |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Vehicles can enter the parking lot and get a ticket
- FR2: System assigns the nearest available spot of appropriate type
- FR3: Vehicles can exit; system calculates fee based on duration + vehicle type
- FR4: System tracks real-time availability per floor per spot type
- FR5: System handles different vehicle types (motorcycle, car, truck) and spot types (compact, large, handicapped)

**Secondary (implement if time allows):**
- FR6: Support for hourly vs flat-rate pricing strategies
- FR7: Electric vehicle spots with charging capability
- FR8: Reserved parking for monthly subscribers
- FR9: Display panels showing available spots per floor

---

## Non-Functional Requirements

- **Concurrency:** Multiple vehicles enter/exit simultaneously
- **Persistence:** In-memory is fine for this interview
- **Scale:** Single-location lot, not distributed
- **Consistency:** Strong — two vehicles must not be assigned the same spot

---

## Constraints and Assumptions

- Each spot can hold exactly one vehicle at a time
- Motorcycles can park in any spot type; cars need compact or large; trucks need large only
- Pricing: hourly rate, per vehicle type (motorcycle cheaper than truck)
- No payment processing integration needed — just fee calculation
- Out of scope: reservations, real-time display panels, external payment gateways

---

## Good Clarifying Questions to Ask

Before designing, a strong candidate asks:

1. What vehicle types do we need to support? Can all vehicles park in all spots?
2. How many floors? How many spots per floor? (affects data structure choice)
3. How is pricing calculated — flat rate, hourly, by vehicle type?
4. Is the "nearest available spot" assignment important? Any priority rules?
5. Multiple entries/exits simultaneously — concurrency concern?
6. Do we need persistence or is in-memory sufficient?
7. What happens if the lot is full?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **ParkingLot** — singleton coordinator; holds floors, manages entry/exit
- **ParkingFloor** — holds spots; knows availability per type
- **ParkingSpot** — individual spot; knows type, occupied status, current vehicle
- **Vehicle** — abstract; has license plate and type; Car/Motorcycle/Truck extend it
- **ParkingTicket** — issued on entry; holds entry time, spot reference
- **PricingStrategy** — interface; HourlyPricing, FlatRatePricing implement it
- **Payment** — holds fee, payment method, status
- **EntryGate / ExitGate** — handle vehicle entry and exit flows

</details>
