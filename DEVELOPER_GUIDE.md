# Developer Guide — Understanding the Principles

This guide is for developers who want to use this repo to become better software designers — not just pass interviews. Interview prep and genuine understanding reinforce each other, but they need different lenses. This file provides the deeper lens.

---

## How to Use This Guide

Two modes:

**Interview Prep Mode** — use `PROBLEM.md` → `MOCK_INTERVIEW.md` → `SOLUTION_GUIDE.md` → `LESSONS_LEARNED.md`. Optimised for scoring 15+/21.

**Learning Mode** — use `DEVELOPER_GUIDE.md` → `patterns/*.md` → `SOLUTION_GUIDE.md`. Optimised for understanding *why the pattern exists* and *recognising it in real codebases*.

Run both. They compound.

---

## Part 1 — The Foundation: Why Object-Oriented Design Exists

OOP wasn't invented to organise code into classes. It was invented to manage **change**.

Software that never changes doesn't need OOP. But software always changes — requirements shift, teams grow, systems scale. The cost of change is the central problem OOP tries to solve.

Every design principle and every pattern in this repo is an answer to one question: **"How do I make this easier to change without breaking what already works?"**

Keep that question in your head while reading every solution guide.

---

## Part 2 — Reading Code as a Designer

Most developers read code to understand what it does. Designers read code to understand *who is responsible for what* and *how easy it is to change*.

When you read a class, ask three questions:

1. **What does this class know?** (its data — fields)
2. **What does this class decide?** (its behaviour — methods with logic)
3. **Who does this class talk to?** (its dependencies)

If the answer to question 3 is "many concrete classes", the design will be hard to change. If the answer is "a few interfaces", the design is extensible.

Patterns are named solutions to recurring patterns of answers to these three questions.

---

## Part 3 — The Five SOLID Principles, Actually Explained

These aren't rules to memorise. They're heuristics that tell you *when your design is about to cause pain*.

### Single Responsibility Principle

**Plain version:** A class should have one reason to change.

**Why it matters:** If a class has two responsibilities, it has two reasons to change. Changes for responsibility A can accidentally break responsibility B. The bug isn't in the changed code — it's in the coupling.

**How to detect violations:** Read the class aloud. If you use the word "and" to describe what it does, it might be violating SRP.

```java
// "UserService registers users AND sends welcome emails AND logs audit events"
// → Three reasons to change. Split into UserRegistrationService, EmailService, AuditLogger.
```

**The real goal:** Not fewer classes. Fewer *reasons for a class to change*. A 300-line class focused on one thing is better than five 10-line classes with tangled responsibilities.

---

### Open/Closed Principle

**Plain version:** Open for extension, closed for modification.

**Why it matters:** Every time you modify existing code, you risk breaking it. The OCP says: design your code so that *adding new behaviour* means *adding new code*, not *changing old code*.

**The mechanism:** Interfaces and polymorphism. Instead of asking "what type is this?" in a switch/if-else, you define an interface and let the types answer for themselves.

```java
// BEFORE (must modify to add a new payment type)
double processFee(VehicleType type) {
    if (type == CAR) return hours * 10;
    if (type == TRUCK) return hours * 20;
    // Adding MOTORCYCLE means editing this method
}

// AFTER (adding MOTORCYCLE = new class, zero edits to existing code)
interface Vehicle { double hourlyRate(); }
class Car     implements Vehicle { public double hourlyRate() { return 10; } }
class Truck   implements Vehicle { public double hourlyRate() { return 20; } }
class Motorcycle implements Vehicle { public double hourlyRate() { return 5; } }
```

**The OCP litmus test:** "To add feature X, how many existing files must I open and edit?" If the answer is more than 1 (the new file itself), your design doesn't satisfy OCP for that change.

---

### Liskov Substitution Principle

**Plain version:** If S is a subtype of T, objects of type T may be replaced with objects of type S without breaking the program.

**Why it matters:** Inheritance is a contract. When you extend a class, you're promising that your subclass behaves consistently with the base class. Violating LSP means polymorphism quietly breaks.

**The classic trap:**

```java
class Rectangle {
    void setWidth(int w)  { this.width = w; }
    void setHeight(int h) { this.height = h; }
}

class Square extends Rectangle {
    void setWidth(int w)  { this.width = w;  this.height = w; } // keeps it square
    void setHeight(int h) { this.height = h; this.width = h; }  // keeps it square
}

// This breaks:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(10);
assert r.getWidth() * r.getHeight() == 50; // FAILS — both are 10
```

`Square` *is* a square, but it's not a valid `Rectangle` in the programming sense. The hierarchy is wrong.

**The fix:** Don't model hierarchies by real-world "is-a" relationships. Model by *behavioural substitutability*. Use composition when the "is-a" relationship doesn't hold behaviourally.

---

### Interface Segregation Principle

**Plain version:** Clients should not be forced to depend on interfaces they don't use.

**The fat interface problem:**

```java
interface Animal {
    void eat();
    void fly();   // Penguins don't fly
    void swim();  // Eagles don't swim
    void run();
}

class Penguin implements Animal {
    public void fly() { throw new UnsupportedOperationException(); } // forced to implement
}
```

Penguins now have a `fly()` method that throws. This is worse than no method — it's a lying contract.

**The fix:** Segregate into `Flyable`, `Swimmable`, `Runnable` interfaces. Classes implement only what applies.

**How to detect violations:** Any `throw new UnsupportedOperationException()` in a method implementation is an ISP violation.

---

### Dependency Inversion Principle

**Plain version:** Depend on abstractions, not concretions. High-level modules should not depend on low-level modules — both should depend on abstractions.

**Why it matters:** This is what makes testing possible. If `OrderService` directly instantiates `MySQLOrderRepository`, you can't test `OrderService` without a database.

```java
// BEFORE — OrderService tightly coupled to MySQL
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository(); // hardwired
}

// AFTER — OrderService depends on interface; concrete impl injected
class OrderService {
    private final OrderRepository repo; // interface
    OrderService(OrderRepository repo) { this.repo = repo; }
}
// In tests: inject MockOrderRepository
// In production: inject MySQLOrderRepository
```

**DIP is the basis of dependency injection.** Spring, Guice, and every IoC container exist to enforce DIP at scale.

---

## Part 4 — Pattern Mental Models

The patterns in this repo exist because certain *shapes of problems* recur. Once you recognise the shape, the pattern is obvious.

### Shape 1: "I have multiple algorithms for the same operation"

**Examples:**
- Multiple pricing strategies (hourly, flat rate, surge)
- Multiple sorting orders (by price, by rating, by relevance)
- Multiple split types in Splitwise (equal, percentage, exact)
- Multiple rate-limiting algorithms (token bucket, sliding window)

**Pattern:** Strategy — extract the algorithm behind an interface. Pass the algorithm as a dependency.

**What makes it Strategy and not just polymorphism:** The algorithm is selected at runtime, often by configuration or user input. The context (the class using it) doesn't know which algorithm it has — just that it implements the interface.

---

### Shape 2: "My object behaves differently depending on its current mode"

**Examples:**
- Vending machine: Idle / HasMoney / ProductSelected / Dispensing
- ATM: Idle / CardInserted / PinVerified / Transaction
- Order: Created / Confirmed / Shipped / Delivered / Cancelled
- Connection: Disconnected / Connecting / Connected / Reconnecting

**Pattern:** State — model each mode as a class. The context delegates all behaviour to its current state object.

**Without State:** Nested if-else chains: `if (state == IDLE) { ... } else if (state == HAS_MONEY) { ... }`. Every new state means editing every existing condition. Every new operation means adding a new condition chain.

**The key insight:** State objects are responsible for their own valid operations. Invalid operations in a state throw or are ignored — not because of a central if-else, but because the state class simply doesn't implement that transition.

---

### Shape 3: "Multiple things need to react when something changes"

**Examples:**
- Order status change → email, SMS, analytics dashboard
- Stock price change → portfolio tracker, price alert, dashboard
- User join chat → all room members notified
- Balance change in Splitwise → notification to all group members

**Pattern:** Observer — the subject maintains a list of observers. On state change, it notifies all of them. Observers register/deregister without touching the subject.

**The anti-pattern it replaces:** Subject holds direct references to each concrete listener. Adding a new listener means modifying the subject's code.

**Where you've seen it without knowing:** Java `EventListener`, Android `OnClickListener`, JavaScript `addEventListener`, Spring `ApplicationEvent` — all Observer.

---

### Shape 4: "I have a tree structure and want to treat leaves and composites uniformly"

**Examples:**
- File system: Files and Directories both have `size()`, `ls()`, `rm()`
- Organization chart: Employees and Managers both have `salary()`, `report()`
- UI components: Buttons and Panels both have `render()`, `resize()`
- JSON/XML: Values and Objects both implement a `Node` interface

**Pattern:** Composite — both leaf and composite implement a common interface. Composite delegates to children; leaf implements directly. Client code never checks which it has.

**The anti-pattern it replaces:** `if (entry instanceof Directory) { recurse } else { process }` — every operation needs an instanceof check.

---

### Shape 5: "I want to add behaviour to an object without changing its class"

**Examples:**
- Adding buffering to a file reader without changing `FileReader`
- Adding rate limiting to a notification channel without changing `EmailChannel`
- Adding logging to any service without changing service code
- Adding compression to an output stream without changing the stream

**Pattern:** Decorator — wraps the object with another object that implements the same interface, adds behaviour, then delegates.

**Key difference from inheritance:** Decorator adds behaviour at runtime to a specific instance. Inheritance adds behaviour at compile time to all instances of a class.

**Where you've seen it:** `BufferedInputStream(new FileInputStream(path))` — that's a Decorator. `new JsonFormatter(new GzipFormatter(new FileAppender(...)))` — that's a chain of Decorators.

---

### Shape 6: "Creating this object requires many steps with many optional parts"

**Examples:**
- Building a pizza with size, crust, sauce, multiple toppings
- Building an HTTP request with headers, params, body, timeout
- Building a SQL query with joins, conditions, ordering, limits
- Building a complex test object with many optional fields

**Pattern:** Builder — step-by-step fluent API. Mandatory parts go in the constructor; optional parts go in chainable setters. `build()` validates and constructs the immutable final object.

**The problem it replaces:** Telescoping constructors:
```java
new Pizza(LARGE, THIN, TOMATO, MOZZARELLA, true, false, true, false, false, true);
// Which boolean is which? — Unreadable and error-prone
```

**The immutability bonus:** The product built by a Builder is typically immutable — all fields set in the constructor, no setters. This makes it thread-safe by default.

---

### Shape 7: "Multiple components need to coordinate but shouldn't know about each other"

**Examples:**
- Chat users sending messages through a chat room (not to each other directly)
- Air traffic control coordinating flights (pilots don't talk to each other)
- Form components notifying each other through a form controller
- Stock buyers and sellers routing through an exchange

**Pattern:** Mediator — components send messages to a central mediator. The mediator knows about everyone; components know only about the mediator.

**Without Mediator:** N components × N-1 connections = O(N²) references. Adding one component means updating N-1 others.

**With Mediator:** N components × 1 mediator reference = N connections. Adding one component means registering with the mediator.

**Mediator vs Observer:** Observer is one-to-many (one subject, many observers). Mediator is many-to-many (any component can communicate with any other, all through the mediator).

---

### Shape 8: "I want to control access to an object — add caching, lazy init, or auth"

**Examples:**
- Cache proxy: intercept calls, return cached result, skip the real call
- Virtual proxy: only create the expensive object on first use
- Protection proxy: check permissions before delegating
- Remote proxy: make a remote object look local

**Pattern:** Proxy — implements the same interface as the real object. Client can't tell if it's talking to the proxy or the real thing.

**Proxy vs Decorator:**
- **Proxy** controls *access* to an object. It often manages the real object's lifecycle.
- **Decorator** adds *behaviour* to an object. The object is passed in from outside.
- Both implement the same interface. The distinction is intent.

---

### Shape 9: "Multiple handlers can process a request; I want them in a chain"

**Examples:**
- Logging framework: DEBUG → INFO → WARN → ERROR handlers in a chain
- Middleware pipeline: auth → rate-limit → cache → handler
- Support escalation: L1 support → L2 support → engineering
- ATM cash dispensing: $100 → $50 → $20 → $10 denomination chain

**Pattern:** Chain of Responsibility — each handler decides: process this request (and optionally pass it on), or pass it to the next handler without processing.

**Two variants:**
1. **Stop at first handler** (strict CoR): each handler either fully handles or passes on. First match wins.
2. **All matching handlers fire** (fan-out): each handler checks if it should process, then always passes on. Used in logging frameworks.

---

### Shape 10: "I want to enforce a fixed sequence of steps, but allow variation in specific steps"

**Examples:**
- Data export: fetch → format → write (all formats use same sequence, formatting varies)
- Game play loop: init → play turns → declare winner (Chess and Checkers share the loop)
- Report generation: collect → aggregate → format → publish

**Pattern:** Template Method — define the skeleton in a `final` method in an abstract base class. Declare variable steps as `abstract`. Provide default no-op hooks for optional steps.

**Must use abstract class, not interface:** `final` methods cannot be declared on interfaces. The base class must be `abstract` to prevent direct instantiation and to enforce the skeleton.

**Template Method vs Strategy:**
- Template Method enforces step ORDER via inheritance
- Strategy swaps the whole algorithm via composition
- Template Method is compile-time; Strategy is runtime

---

### Shape 11: "I need to undo/redo operations"

**Examples:**
- Text editor undo stack
- Chess game move history
- Database transaction rollback
- Task scheduler (cancel a scheduled task)

**Pattern:** Command — encapsulate each operation as an object. The object has `execute()` and `undo()`. Store them in a stack for undo/redo.

**What makes it Command and not just a method call:** The operation is stored as an object with state (what it captured before executing). `undo()` uses that captured state to reverse.

---

## Part 5 — Recognising Patterns in Real Codebases

You don't encounter patterns labeled "this is a Strategy." You encounter code and need to recognise the shape.

### Reading JDK code

| You see | Pattern |
|---------|---------|
| `Comparator` | Strategy |
| `Runnable`, `Callable` | Command |
| `Iterator` | Iterator |
| `BufferedInputStream(FileInputStream)` | Decorator |
| `EventListener` | Observer |
| `HttpServlet.service() → doGet/doPost` | Template Method |
| CGLIB / JDK dynamic proxy in Spring | Proxy |
| `AbstractList` with abstract `get()/size()` | Template Method |

### Reading Spring code

| You see | Pattern |
|---------|---------|
| `@Transactional` | Proxy (AOP proxy wraps service) |
| `@Cacheable` | Proxy (caching proxy) |
| `ApplicationEvent` + `@EventListener` | Observer |
| `BeanFactoryPostProcessor` | Chain of Responsibility |
| `HandlerInterceptor` chain | Chain of Responsibility |
| `BeanDefinitionBuilder` | Builder |
| `@Conditional` | Strategy |

---

## Part 6 — When NOT to Use a Pattern

Patterns are tools. Using a tool where it doesn't fit is over-engineering.

| Don't use | When |
|-----------|------|
| Strategy | There's only one algorithm (YAGNI) |
| Observer | There's only one listener and it never changes |
| State | There are only 2 states (use a boolean) |
| Builder | Object has 2 fields (use a constructor) |
| Singleton | Just to avoid passing an argument — use DI instead |
| Factory | Object creation is a single `new` call with no complexity |
| Template Method | The steps are different, not just the implementation — use Strategy instead |

**The over-engineering smell:** Adding patterns because you *might* need flexibility later. Build for what you need now. Refactor to a pattern when the second variation appears.

---

## Part 7 — How to Use This Repository for Genuine Learning

### Option A: Pattern-first learning

1. Pick a pattern you want to understand deeply
2. Read `patterns/[PATTERN].md` — trigger signals, structure, JDK examples
3. Read `DEVELOPER_GUIDE.md` Part 4 for the mental model
4. Read the problem that uses it (e.g., Vending Machine for State)
5. Read `SOLUTION_GUIDE.md` — see the pattern applied to a concrete problem
6. Try implementing the problem from scratch
7. Read `LESSONS_LEARNED.md` — see what you missed

### Option B: Problem-first learning

1. Pick a problem you find interesting
2. Read `PROBLEM.md` — understand the requirements
3. Try designing it yourself (15–20 min)
4. Read `SOLUTION_GUIDE.md` — compare
5. For every design decision in the solution, ask: "Why this way and not another?"
6. Read `DEVELOPER_GUIDE.md` Part 4 for the pattern's mental model
7. Find another problem that uses the same pattern — see the same shape in a different context

### Option C: Principle-first learning (most rigorous)

1. Pick a SOLID principle from Part 3 of this file
2. Search for a violation of that principle in your own codebase at work
3. Find the problem in this repo that most directly shows how to fix that violation
4. Read `SOLUTION_GUIDE.md` — apply the fix mentally to your own codebase
5. Implement the fix at work — real practice beats simulated practice

### Option D: AI-guided deep dive

Say to the agent: `"Deep dive [pattern name]"` or `"Why does [pattern] exist?"`

The agent will:
1. Show the naive code before the pattern
2. Transform it step by step
3. Show which SOLID principle it enforces
4. Show real-world usage in JDK/Spring
5. Offer a fresh practice problem

---

## Part 8 — The Questions Better Developers Ask

When reviewing any piece of code, better developers ask these questions automatically. Practice asking them on every `SOLUTION_GUIDE.md` you read:

**On responsibilities:**
- "If requirement X changes, which classes need to change?"
- "Is that the right class to own that logic?"

**On extensibility:**
- "To add a new [type/strategy/channel], how many files do I touch?"
- "If the answer is more than 1, what needs to change about the design?"

**On dependencies:**
- "Does this class depend on a concrete class or an interface?"
- "Could I swap this implementation for a different one without changing the caller?"
- "Can I test this class without its real dependencies?"

**On concurrency:**
- "What mutable state does this class hold?"
- "Can two threads write to it simultaneously?"
- "What's the worst race condition that could happen?"

**On naming:**
- "Does the name of this class accurately describe its single responsibility?"
- "If I have to use the word 'and' or 'manager' or 'helper', is the responsibility clear?"

---

## Part 9 — The Learning Progression

Learning OOP design is not linear. It follows a spiral:

```
Awareness → Application → Internalization → Intuition
```

| Stage | What it looks like |
|-------|-------------------|
| **Awareness** | You recognise pattern names when someone tells you |
| **Application** | You can apply a pattern when told which one to use |
| **Internalization** | You identify which pattern applies without being told |
| **Intuition** | You see the shape of the problem before you see the solution |

Most interview prep stops at Awareness. The drills in this repo target Application. The problems in `SOLUTION_GUIDE.md` target Internalization. The questions in Part 8 of this guide target Intuition.

Target Intuition. It's the one that makes you a genuinely better developer.
