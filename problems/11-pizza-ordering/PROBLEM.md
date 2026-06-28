# Pizza Ordering System

**Difficulty:** Foundation
**Category:** Builder Pattern
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Builder (pizza construction), Strategy (pricing/discount), Factory Method (pizza types)
**Asked at:** Amazon, Adobe, Swiggy, Domino's tech, Zomato

---

## Problem Statement

Design a pizza ordering system. Customers can build a pizza by choosing a base (size, crust), sauce, cheese, and toppings (multiple, optional). The system calculates the total price, generates an order summary, and supports different pizza presets (Margherita, Pepperoni, BBQ Chicken) that can be further customised.

The core challenge is: how do you construct a complex object (Pizza) with many optional components without a constructor that takes 15 parameters?

---

## Actors

| Actor | Description |
|-------|-------------|
| Customer | Selects pizza configuration; places order |
| Pizza System | Builds pizza object; calculates price; generates receipt |
| Chef / Kitchen | Receives order; prepares pizza (out of scope) |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Customer selects size (SMALL, MEDIUM, LARGE)
- FR2: Customer selects crust (THIN, THICK, STUFFED)
- FR3: Customer selects sauce (TOMATO, BBQ, PESTO, ALFREDO)
- FR4: Customer adds cheese (MOZZARELLA, CHEDDAR, VEGAN)
- FR5: Customer adds toppings (PEPPERONI, MUSHROOMS, OLIVES, PEPPERS, ONIONS) — multiple allowed
- FR6: System calculates total price based on all components
- FR7: System generates human-readable order summary
- FR8: Preset pizzas (Margherita, Pepperoni) can be ordered directly and customised

**Secondary (implement if time allows):**
- FR9: Order can be placed for multiple pizzas
- FR10: Discount codes (10% off on orders > $30)
- FR11: Vegetarian validation — flag if non-veg toppings added to a veg order

---

## Non-Functional Requirements

- **Immutability:** A built `Pizza` object should be immutable — once built, no changes
- **Readability:** Building a pizza should read like natural language: `new PizzaBuilder().size(LARGE).crust(THIN).addTopping(PEPPERONI).build()`
- **Extensibility:** New toppings, sizes, or crust types should not require changing the builder interface

---

## Constraints and Assumptions

- One pizza per order session for MVP (multi-pizza is secondary)
- Prices are fixed per component (no dynamic pricing for MVP)
- Pizza must have at least a size and a crust to be valid
- Out of scope: kitchen display system, delivery tracking, payment processing

---

## Good Clarifying Questions to Ask

1. What customisation options are needed — just toppings, or also crust, sauce, cheese?
2. Are preset pizzas just a convenience or do they enforce specific ingredient rules?
3. Can a customer add the same topping twice (double pepperoni)?
4. Should the built pizza object be mutable or immutable?
5. How is price calculated — per component, or fixed per pizza size?
6. Is there a maximum topping limit?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **Pizza** — immutable value object; size, crust, sauce, cheese, List<Topping>, totalPrice
- **PizzaBuilder** — fluent builder; accumulates choices; `build()` validates and constructs Pizza
- **PizzaDirector** — optional; encapsulates preset recipes (Margherita, Pepperoni) using PizzaBuilder
- **Size** — enum: SMALL, MEDIUM, LARGE (each has a base price)
- **Crust** — enum: THIN, THICK, STUFFED (each has a price delta)
- **Sauce** — enum: TOMATO, BBQ, PESTO, ALFREDO (each has a price)
- **Cheese** — enum: MOZZARELLA, CHEDDAR, VEGAN (each has a price)
- **Topping** — enum: PEPPERONI, MUSHROOMS, OLIVES, PEPPERS, ONIONS (each has a price)
- **Order** — holds pizza(s), customer info, applied discount, final price

</details>

---

## Why Builder Here?

**The telescoping constructor problem:**

```java
// BAD — which param is which?
new Pizza(Size.LARGE, Crust.THIN, Sauce.TOMATO, Cheese.MOZZARELLA,
          true, false, true, false, true, false); // 10 booleans for toppings

// GOOD — fluent, readable, optional steps
Pizza pizza = new PizzaBuilder()
    .size(Size.LARGE)
    .crust(Crust.THIN)
    .sauce(Sauce.TOMATO)
    .cheese(Cheese.MOZZARELLA)
    .addTopping(Topping.PEPPERONI)
    .addTopping(Topping.MUSHROOMS)
    .build();
```
