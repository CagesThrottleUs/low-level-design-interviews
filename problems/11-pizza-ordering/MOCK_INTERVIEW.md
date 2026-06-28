# Mock Interview Script — Pizza Ordering System

**For the interviewer.**

---

## Opening

> "Design a pizza ordering system. Customers can build a pizza by choosing a size, crust, sauce, cheese, and toppings. The system should also support preset pizzas like Margherita. Ask any clarifying questions before you start."

| If candidate asks | Answer |
|------------------|--------|
| What customisation options? | Size (S/M/L), crust (thin/thick/stuffed), sauce, cheese type, multiple toppings |
| Are preset pizzas fixed? | Yes — Margherita, Pepperoni, BBQ Chicken are presets; can still be customised |
| Should Pizza be mutable after creation? | No — once built, it's immutable |
| Pricing? | Each component has a fixed price; calculate total at build time |
| Multi-pizza orders? | Secondary — one pizza for MVP |
| Thread safety? | Pizza itself is already thread-safe if immutable; focus on that |

---

## Phase 2: Core Design — What to Watch For

**Key signal:** Does candidate identify the telescoping constructor problem?

Good candidates say: "With size, crust, sauce, cheese, and 6+ toppings as optional fields, a constructor becomes unreadable — `new Pizza(LARGE, THIN, TOMATO, true, false, true, false, true)`. Builder pattern solves this with a fluent API."

If candidate uses a constructor with many parameters:
> "How does the caller know which boolean means pepperoni vs mushrooms?"

If candidate puts mandatory fields as optional setters:
> "What prevents someone from calling `build()` without setting a size?"

**Director class signal:** Strong candidates introduce a `PizzaDirector` or similar that encapsulates preset recipes, so callers don't need to know the build sequence for a Margherita.

---

## Extension Probe (~35 min)

> "Now add new toppings easily — say, jalapeños and ham. How would your design absorb this?"

Good: Toppings as an enum or `Set<Topping>` in the builder — new topping = new enum value, zero builder changes.

Bad: Boolean field per topping (6 booleans now, grows forever).

> "Now the built pizza should be usable as a billing line item — it needs a price. Where does that go?"

Strong: `build()` computes total price from component prices and stores it in the immutable `Pizza`. Price is set once at build time.

---

## Concurrency Probe (~40 min)

> "Multiple orders come in simultaneously. What's thread-safe here?"

Expected: "Pizza itself is safe — all fields are final. The order queue needs a thread-safe collection (`ConcurrentLinkedQueue` or `LinkedBlockingQueue`). The pizza shop singleton needs double-checked locking with `volatile`."

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Mandatory vs optional fields identified, immutability discussed, preset support | |
| Entity Modeling | Pizza as immutable product, Builder as inner class, Director for presets | |
| Design Patterns | Builder pattern — mandatory fields in constructor, fluent setters, `build()` validates | |
| SOLID | SRP: Pizza doesn't know how to build itself; Builder does. OCP: new topping = enum value | |
| Extensibility | New topping absorbed without changing builder interface | |
| Concurrency | Pizza immutability = thread-safe; order queue + singleton addressed | |
| Communication | Explained WHY builder (telescoping constructor problem), WHY immutability | |
| **TOTAL** | | **/21** |
