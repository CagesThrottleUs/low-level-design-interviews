# Builder Pattern

**Category:** Creational
**LLD interview frequency:** Tier 2
**Problem:** 11 — Pizza Ordering System

---

## The Problem It Solves

When an object has many optional configuration parameters, constructors become unreadable.

```java
// Telescoping constructor — which boolean is which?
new Pizza(Size.LARGE, Crust.THIN, Sauce.TOMATO, Cheese.MOZZARELLA,
          true, false, true, false, false, true);

// Named parameters via setters — mutable, partially-constructed objects possible
Pizza p = new Pizza();
p.setSize(Size.LARGE);
p.addTopping(Topping.PEPPERONI);
// Forgot to call p.setCrust() — object is now incomplete
pizza = p; // broken object escapes
```

---

## Intent

Separate object construction from its representation. Provide a step-by-step fluent API. The fully-constructed object is immutable and always valid.

---

## Trigger Signals

Apply Builder when you see:
- A class with 4+ constructor parameters, especially optional ones
- A class with many boolean flags
- "The object should be immutable once created"
- Multiple preset configurations of the same object (Margherita, Pepperoni pizzas)

---

## Structure

```java
// Product — immutable; private constructor forces use of Builder
public final class Pizza {
    // All fields final — immutable by construction
    private final Size size;          // mandatory
    private final Crust crust;        // mandatory
    private final Sauce sauce;        // optional, has default
    private final Set<Topping> toppings; // optional, can be empty

    private Pizza(Builder b) { /* copy from builder */ }

    public static class Builder {
        // Mandatory fields in constructor — can't build without them
        private final Size size;
        private final Crust crust;
        // Optional with defaults
        private Sauce sauce = Sauce.TOMATO;
        private EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        public Builder(Size size, Crust crust) { this.size = size; this.crust = crust; }
        public Builder sauce(Sauce s)         { this.sauce = s;       return this; }
        public Builder addTopping(Topping t)  { toppings.add(t);      return this; }

        public Pizza build() {
            // validate here — last gate before object exits builder
            return new Pizza(this);
        }
    }
}

// Director — optional; encapsulates preset build sequences
public class PizzaDirector {
    public Pizza buildMargherita(Size size) {
        return new Pizza.Builder(size, Crust.THIN)
            .sauce(Sauce.TOMATO).build();
    }
}
```

---

## Key Properties of Good Builder Usage

1. **Mandatory fields in the Builder's constructor** — can't call `build()` without them
2. **`build()` is the validation gate** — no invalid object can escape
3. **Product is immutable** — all fields `final`, no setters
4. **Director encapsulates presets** — one place to update Margherita recipe
5. **Extensible via enum** — adding a new topping = new enum value, zero builder changes

---

## LLD Problems Using Builder

| Problem | Builder for |
| --- | --- |
| Pizza Ordering | Pizza construction with optional toppings |
| Order Management | Order with optional discount, notes, delivery address |
| HTTP Client | Request with optional headers, timeout, body |
| SQL Query Builder | Query with optional joins, conditions, ordering |

---

## vs. Telescoping Constructors

```java
// BEFORE — 10-param constructor
new Pizza(LARGE, THIN, TOMATO, MOZZARELLA, true, false, true, false, false, true);

// AFTER — readable, self-documenting
new Pizza.Builder(LARGE, THIN)
    .sauce(TOMATO).cheese(MOZZARELLA)
    .addTopping(PEPPERONI).addTopping(MUSHROOMS)
    .build();
```

---

## Underlying Principle

**SRP:** The Pizza class doesn't know how to build itself — that's the Builder's job.
**Immutability:** Thread-safe by default — all fields `final`, JMM guarantees visibility post-construction.

---

## Real-World Usage

- `StringBuilder` — fluent string construction
- `HttpRequest.newBuilder()` in Java 11+
- `AlertDialog.Builder` in Android
- `ProcessBuilder` in Java
- Lombok's `@Builder` generates this structure automatically

---

## When NOT to Use

- Object has 2 fields — use a constructor
- All fields are mandatory — use a constructor with named parameters
- You don't need immutability — setters are fine if mutability is intended
