# Solution Guide — Pizza Ordering System

**Read after your attempt.**

---

## Entity Map

| Class | Type | Responsibility |
|-------|------|---------------|
| `Pizza` | Immutable product | All fields `private final`; private constructor; no setters |
| `Pizza.Builder` | Static inner builder | Mandatory fields in constructor; optional via fluent setters; `build()` validates |
| `PizzaDirector` | Director | Encapsulates preset recipes; calls builder methods in correct order |
| `Size` | Enum | SMALL(base price 5.00), MEDIUM(8.00), LARGE(12.00) |
| `Crust` | Enum | THIN(0), THICK(1.00), STUFFED(2.00) |
| `Sauce` | Enum | TOMATO(0.50), BBQ(0.75), PESTO(1.00), ALFREDO(1.00) |
| `Cheese` | Enum | MOZZARELLA(1.00), CHEDDAR(1.00), VEGAN(1.50) |
| `Topping` | Enum | PEPPERONI(1.50), MUSHROOMS(0.75), OLIVES(0.50), etc. |
| `Order` | Aggregate | List<Pizza>, customer info, status (own Builder too) |

---

## Why Builder

The telescoping constructor problem: a pizza has 2 mandatory fields and ~8 optional ones. Without Builder:

```java
// Unreadable — which boolean is which?
new Pizza(LARGE, THIN, TOMATO, MOZZARELLA, true, false, true, false, true);
```

With Builder — readable, self-documenting, no invalid partially-constructed object:
```java
Pizza pizza = new Pizza.Builder(Size.LARGE, Crust.THIN)
    .sauce(Sauce.TOMATO)
    .cheese(Cheese.MOZZARELLA)
    .addTopping(Topping.PEPPERONI)
    .addTopping(Topping.MUSHROOMS)
    .build();
```

---

## Core Implementation

```java
import java.util.*;

public final class Pizza {
    // Mandatory — final
    private final Size size;
    private final Crust crust;
    // Optional — final with defaults
    private final Sauce sauce;
    private final Cheese cheese;
    private final Set<Topping> toppings;
    private final double totalPrice;

    private Pizza(Builder b) {
        this.size       = b.size;
        this.crust      = b.crust;
        this.sauce      = b.sauce;
        this.cheese     = b.cheese;
        this.toppings   = Collections.unmodifiableSet(new EnumSet<Topping>(b.toppings));  
        this.totalPrice = b.calculatePrice();
    }

    // Only getters — no setters
    public Size getSize()          { return size; }
    public double getTotalPrice()  { return totalPrice; }
    public Set<Topping> getToppings() { return toppings; }

    @Override
    public String toString() {
        return String.format("Pizza[%s, %s, sauce=%s, cheese=%s, toppings=%s, $%.2f]",
            size, crust, sauce, cheese, toppings, totalPrice);
    }

    // ── Static inner Builder ──────────────────────────────────────────────────
    public static class Builder {
        // Mandatory
        private final Size size;
        private final Crust crust;
        // Optional with defaults
        private Sauce sauce      = Sauce.TOMATO;
        private Cheese cheese    = Cheese.MOZZARELLA;
        private EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        public Builder(Size size, Crust crust) {
            if (size == null || crust == null)
                throw new IllegalArgumentException("Size and crust are mandatory");
            this.size  = size;
            this.crust = crust;
        }

        public Builder sauce(Sauce s)         { this.sauce = s;          return this; }
        public Builder cheese(Cheese c)       { this.cheese = c;         return this; }
        public Builder addTopping(Topping t)  { this.toppings.add(t);    return this; }

        double calculatePrice() {
            double price = size.basePrice + crust.addedPrice
                         + (sauce  != null ? sauce.price  : 0)
                         + (cheese != null ? cheese.price : 0);
            for (Topping t : toppings) price += t.price;
            return price;
        }

        public Pizza build() {
            // Validation gate — only valid Pizza exits here
            if (toppings.size() > 10)
                throw new IllegalStateException("Max 10 toppings allowed");
            return new Pizza(this);
        }
    }
}

// Enums with embedded pricing
enum Size {
    SMALL(5.00), MEDIUM(8.00), LARGE(12.00);
    final double basePrice;
    Size(double p) { this.basePrice = p; }
}

enum Crust {
    THIN(0.00), THICK(1.00), STUFFED(2.00);
    final double addedPrice;
    Crust(double p) { this.addedPrice = p; }
}

enum Topping {
    PEPPERONI(1.50), MUSHROOMS(0.75), OLIVES(0.50),
    PEPPERS(0.50), ONIONS(0.50), JALAPENOS(0.75), HAM(1.25);
    final double price;
    Topping(double p) { this.price = p; }
}
```

---

## PizzaDirector — Preset Recipes

```java
public class PizzaDirector {
    // Director owns the sequence; caller owns the builder
    public Pizza buildMargherita(Size size) {
        return new Pizza.Builder(size, Crust.THIN)
            .sauce(Sauce.TOMATO)
            .cheese(Cheese.MOZZARELLA)
            .build();
    }

    public Pizza buildPepperoni(Size size) {
        return new Pizza.Builder(size, Crust.THICK)
            .sauce(Sauce.TOMATO)
            .cheese(Cheese.MOZZARELLA)
            .addTopping(Topping.PEPPERONI)
            .build();
    }

    public Pizza buildBBQChicken(Size size) {
        return new Pizza.Builder(size, Crust.THICK)
            .sauce(Sauce.BBQ)
            .cheese(Cheese.CHEDDAR)
            .build();
    }
}
```

---

## Order (also uses Builder)

```java
public final class Order {
    public enum Status { PLACED, PREPARING, READY, DELIVERED }

    private final String orderId;
    private final List<Pizza> pizzas;
    private final String customerName;
    private volatile Status status;

    private Order(Builder b) {
        this.orderId      = UUID.randomUUID().toString();
        this.pizzas       = List.copyOf(b.pizzas);
        this.customerName = b.customerName;
        this.status       = Status.PLACED;
    }

    public static class Builder {
        private String customerName;
        private final List<Pizza> pizzas = new ArrayList<>();

        public Builder customer(String name)  { this.customerName = name; return this; }
        public Builder addPizza(Pizza p)      { this.pizzas.add(p);       return this; }

        public Order build() {
            Objects.requireNonNull(customerName, "Customer name required");
            if (pizzas.isEmpty()) throw new IllegalStateException("At least one pizza required");
            return new Order(this);
        }
    }
}
```

---

## Concurrency

- `Pizza`: all fields `final` — JMM guarantees visibility post-construction. Inherently thread-safe.
- `Order.status`: `volatile` — main thread advances status; worker threads read it.
- Order queue in `PizzaShop`: `ConcurrentLinkedQueue` for non-blocking or `LinkedBlockingQueue` for worker-thread-based processing.
- `PizzaShop` singleton: double-checked locking with `volatile instance`.

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| New topping | New `Topping` enum value — zero builder changes |
| New crust type | New `Crust` enum value |
| Vegan validation | `build()` checks: if any meat topping + VEGAN cheese → throw or warn |
| Discount codes | `DiscountStrategy` applied to Order's total after build |
| Order status lifecycle | State pattern on `Order` |

---

## What Strong Candidates Do Differently

- Mandatory fields in Builder's constructor (not setters) — prevents invalid `build()` calls
- `EnumSet<Topping>` instead of 6 boolean fields — new topping = new enum value
- `build()` as the validation gate — no invalid Pizza can be constructed
- Both Pizza AND Order use Builder — applies the pattern consistently
- Director for presets — demonstrates understanding of the Director role

## What Average Candidates Miss

- Optional fields via constructor overloads (telescoping constructor anti-pattern)
- Mutable Pizza — setters after construction
- No Director — callers manually construct presets, duplicating build sequences
- Validation scattered (not centralized in `build()`)
- No enum pricing — prices hardcoded in if-else
