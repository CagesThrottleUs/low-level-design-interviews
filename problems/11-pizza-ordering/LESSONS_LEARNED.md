# Lessons Learned — Pizza Ordering System

---

## Mistake 1: Telescoping Constructor Instead of Builder

**What happens:** `new Pizza(LARGE, THIN, TOMATO, MOZZARELLA, true, false, true, false)` — positional boolean arguments.

**Why it's wrong:** Caller can't tell which `true` means pepperoni vs mushrooms. Swapping two booleans is a silent bug. Constructor grows with every new topping.

**Fix:** Builder with fluent API. New topping = new enum value + new setter. No existing calls break.

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 2: Mandatory Fields as Optional Setters

**What happens:**
```java
Pizza p = new Pizza.Builder()
    .sauce(Sauce.TOMATO)
    .build(); // size was never set!
```
`build()` creates a Pizza with `size = null`. NullPointerException later.

**Fix:** Mandatory fields go in the Builder's constructor:
```java
public Builder(Size size, Crust crust) {
    Objects.requireNonNull(size, "Size is mandatory");
    Objects.requireNonNull(crust, "Crust is mandatory");
    this.size = size; this.crust = crust;
}
// Can't even create a Builder without providing mandatory fields
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 3: Mutable Pizza After Construction

**What happens:** `Pizza` has setters — `setTopping(Topping t)`. Built pizza passed around, toppings added after the fact.

**Why it's wrong:** Breaks immutability contract. Thread-unsafe. Price calculated at build time is now stale. Defeats the Builder's purpose.

**Fix:** `private final` on all fields. No setters. `Collections.unmodifiableSet()` for toppings. Price computed in `build()`.

**Gap dimension:** Entity Modeling / Concurrency (Dimensions 2, 6)

---

## Mistake 4: No Director — Callers Manually Build Presets

**What happens:** Every caller who wants a Margherita writes the same 4 builder calls. Duplicate logic. If Margherita recipe changes, 10 call sites need updating.

**Fix:**
```java
class PizzaDirector {
    public Pizza buildMargherita(Size size) {
        return new Pizza.Builder(size, Crust.THIN)
            .sauce(Sauce.TOMATO).cheese(Cheese.MOZZARELLA).build();
    }
}
// ONE place to update when Margherita recipe changes
```

**Gap dimension:** Design Patterns (Dimension 3)

---

## Mistake 5: Boolean Fields for Toppings

**What happens:** `boolean hasPepperoni, hasMushrooms, hasOlives` — 3 fields now, 10 after requests come in.

**Fix:** `EnumSet<Topping>` in the Builder:
```java
private EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
public Builder addTopping(Topping t) { toppings.add(t); return this; }
// Adding jalapeños = new enum value, zero builder code changes
```

**Gap dimension:** SOLID / Extensibility (Dimensions 4, 5)

---

## Self-Assessment Checklist

- [ ] Are mandatory fields (size, crust) in the Builder's constructor?
- [ ] Are all Pizza fields `private final`?
- [ ] Does `build()` validate before constructing?
- [ ] Are toppings a `Set<Topping>` (not boolean fields)?
- [ ] Is there a Director for preset pizzas?
- [ ] Adding a new topping requires only a new enum value?
- [ ] Is Pizza price computed at `build()` time (not mutable after)?

---

## Remediation Targets

| Dimension | If scored 0–1 | Go to |
|-----------|--------------|-------|
| Design Patterns | No Builder, or broken Builder | GAP_REMEDIATION.md#gap-3 — Builder trigger: "many optional params" |
| Entity Modeling | Mutable Pizza, no Director | GAP_REMEDIATION.md#gap-2 |
| SOLID | Boolean fields per topping (OCP violation) | GAP_REMEDIATION.md#gap-4 |
| Extensibility | New topping requires builder change | GAP_REMEDIATION.md#gap-5 |
