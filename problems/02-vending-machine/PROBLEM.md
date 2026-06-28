# Vending Machine

**Difficulty:** Foundation
**Category:** State Machine
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** State (core FSM), Strategy (payment handling)
**Asked at:** Google, Amazon, Uber, Swiggy, Flipkart

---

## Problem Statement

Design a vending machine that dispenses products. The machine accepts coins/currency, allows product selection, dispenses the selected item if payment is sufficient, and returns change. The machine must handle multiple states: idle, coin inserted, product selected, and dispensing.

The machine should refuse invalid operations based on its current state (e.g., you can't select a product before inserting money, you can't insert money while dispensing).

---

## Actors

| Actor | Description |
|-------|-------------|
| Customer | Inserts money, selects product, collects item and change |
| Technician | Restocks products, collects cash (optional scope) |
| Vending Machine | Manages state, inventory, payment, dispensing |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Customer can insert coins/bills
- FR2: Customer can select a product by code
- FR3: Machine dispenses product if payment >= price and item in stock
- FR4: Machine returns change if payment > price
- FR5: Machine returns money if customer cancels
- FR6: Machine rejects operation if invalid for current state (e.g., select before paying)

**Secondary (implement if time allows):**
- FR7: Technician can restock items
- FR8: Technician can collect cash
- FR9: Machine displays available products and prices
- FR10: Support multiple payment types (coins, card)

---

## Non-Functional Requirements

- **Concurrency:** Single customer at a time per machine (no concurrent multi-user)
- **Persistence:** In-memory is fine
- **Scale:** Single machine; not distributed
- **State safety:** No invalid state transitions — machine must never dispense without payment

---

## Constraints and Assumptions

- Products identified by a slot/code (e.g., A1, B2)
- Payment is cumulative (insert multiple coins until sufficient)
- Machine holds limited inventory per slot
- Exact product codes are pre-loaded; no dynamic pricing changes at runtime
- Out of scope: network connectivity, remote monitoring, multi-machine fleet

---

## Good Clarifying Questions to Ask

1. What payment types? Coins only, or bills and cards too?
2. Does the machine handle partial payment (insert coin, insert more coins)?
3. What happens if item is out of stock after payment is inserted?
4. Can the customer cancel mid-flow and get money back?
5. Does the machine need to track which items are low stock?
6. Multiple customers simultaneously or one at a time?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **VendingMachine** — holds state, inventory, payment; delegates operations to current state
- **VendingMachineState** — interface; defines operations: insertCoin, selectProduct, dispense, cancel
- **IdleState** — accepts coin insertion; rejects all else
- **HasMoneyState** — accepts more coins or product selection; accepts cancel
- **ProductSelectedState** — validates payment sufficiency, transitions to dispensing
- **DispensingState** — dispenses product, returns change, transitions back to idle
- **Product** — name, price, quantity
- **Inventory** — map of code → Product; handles stock checks and decrements
- **Coin** — enum with denominations and values

</details>
