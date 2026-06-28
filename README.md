# Low Level Design Interview Practice Harness

Structured practice system for LLD/OOD interviews. Covers problem bank, mock interview scripts, evaluation rubrics, and gap remediation.

## What Is This

LLD interviews test object-oriented design skills — entity modeling, design patterns, SOLID principles, and extensibility. This harness gives you:

- **Problem bank** — 30+ problems organized by category and difficulty
- **Mock interview scripts** — interviewer-side scripts to simulate real sessions
- **Evaluation rubrics** — 7-dimension scoring aligned with what companies actually measure
- **Lessons learned** — common mistakes per problem + how to fix them
- **Gap remediation** — diagnose weaknesses and target them systematically
- **Progress tracker** — log attempts, scores, and improvements over time

## Directory Layout

```
README.md                  — this file
EVALUATION_RUBRIC.md       — 7-dimension scoring system used across all problems
GAP_REMEDIATION.md         — weakness diagnosis + targeted fix guide

problems/
  _TEMPLATE/               — blank template for adding new problems
  01-parking-lot/          — Tier 1: Foundation
  02-vending-machine/      — Tier 1: Foundation
  03-lru-cache/            — Tier 1: Foundation
  04-elevator-system/      — Tier 1: Foundation
  05-splitwise/            — Tier 2: Intermediate

mock-interview/
  HOW_TO_RUN.md            — step-by-step guide for running a session
  SESSION_SCORECARD.md     — fill-in scorecard template per session

patterns/
  STRATEGY.md              — when to use, structure, LLD example
  OBSERVER.md
  STATE.md
  FACTORY.md

progress/
  TRACKER.md               — log of all practice sessions with scores
```

## Problem Bank Overview

| # | Problem | Tier | Patterns | Key Challenge |
|---|---------|------|----------|---------------|
| 01 | Parking Lot | Foundation | Strategy, Factory | Multi-entity, allocation strategy |
| 02 | Vending Machine | Foundation | State | FSM design, payment handling |
| 03 | LRU Cache | Foundation | — | DS + eviction + thread safety |
| 04 | Elevator System | Foundation | State, Strategy | Scheduling algorithm, FSM |
| 05 | Splitwise | Intermediate | Strategy, Observer | Expense splitting math, group management |
| 06 | ATM System | Foundation-Intermediate | State, Strategy | Atomic withdrawal, PIN lockout |
| 07 | Movie Ticket Booking | Intermediate | State, Strategy | Concurrent seat hold, payment flow |
| 08 | Notification System | Intermediate | Strategy, Decorator | Multi-channel, retry, user preferences |
| 09 | Rate Limiter | Intermediate | Strategy | Token bucket, lazy refill, CAS |
| 10 | Chess Game | Advanced | Command, Polymorphism | Move undo, check detection, piece hierarchy |
| 11 | Pizza Ordering | Foundation | **Builder** | Telescoping constructor, immutable product, Director |
| 12 | In-Memory File System | Intermediate | **Composite** | Uniform size() on leaf+composite, Map children, parent ref |
| 13 | Logging Framework | Intermediate | **Chain of Responsibility**, Strategy | Per-appender threshold, formatter independence, flush |
| 14 | Chat Room | Intermediate | **Mediator** | Zero user-to-user refs, CopyOnWriteArrayList, DIP |
| 15 | Caching Proxy | Intermediate | **Proxy** | TTL CacheEntry, thundering herd, virtual proxy volatile |
| 16 | Data Exporter | Foundation | **Template Method** | final skeleton, abstract class not interface, hooks |
| 17 | Bounded Blocking Queue | Advanced | Producer-Consumer | Two Conditions, while loop, signal direction, finally |
| 18 | Custom Thread Pool | Advanced | Thread Pool, State Machine | volatile state, poll(timeout), worker survives exceptions |

See [problems/](problems/) for full problem list.

## Pattern Coverage

All GoF patterns relevant to LLD interviews are covered:

| Category | Pattern | Problem(s) |
| --- | --- | --- |
| **Creational** | Builder | 11 Pizza Ordering |
| | Factory Method | 01 Parking Lot |
| | Singleton | embedded in multiple |
| **Structural** | Composite | 12 File System |
| | Decorator | 08 Notification System |
| | Proxy | 15 Caching Proxy |
| **Behavioral** | Chain of Responsibility | 13 Logging Framework |
| | Command | 10 Chess Game |
| | Mediator | 14 Chat Room |
| | Observer | 05 Splitwise, 08 Notification |
| | State | 02 Vending Machine, 04 Elevator, 06 ATM, 07 Movie Booking |
| | Strategy | 01, 05, 07, 08, 09, 13 |
| | Template Method | 16 Data Exporter |

## Concurrency Coverage

| Concept | Problem(s) |
| --- | --- |
| `synchronized` per-object | 01 Parking Lot, 07 Movie Booking |
| `ReentrantReadWriteLock` | 03 LRU Cache, 12 File System |
| `AtomicLong` / CAS | 09 Rate Limiter |
| `ConcurrentHashMap` | multiple |
| `ReentrantLock` + two `Condition` variables | **17 Blocking Queue** |
| `volatile` + double-checked locking | 15 Proxy, 18 Thread Pool |
| `CopyOnWriteArrayList` | 13 Logging, 14 Chat Room |
| `BlockingQueue` + worker lifecycle | **18 Thread Pool** |
| Per-resource locking (fine-grained) | 01 Parking Lot, 07 Movie Booking |
| Thundering herd (`FutureTask` + `putIfAbsent`) | 15 Caching Proxy |

See [problems/](problems/) for full problem list.

## How to Use

### Solo Practice (45-min session)

1. Pick a problem from the bank
2. Open `PROBLEM.md` — read requirements, set a timer for 45 min
3. Work through design on paper or in code
4. When done, score yourself against `EVALUATION_RUBRIC.md`
5. Review `SOLUTION_GUIDE.md` to see what you missed
6. Read `LESSONS_LEARNED.md` — identify which anti-patterns you hit
7. Update `progress/TRACKER.md` with your score and gaps

### Mock Interview (with a partner)

1. Partner reads `MOCK_INTERVIEW.md` for the chosen problem
2. Run the 45-min session — partner plays interviewer
3. Partner scores using the problem-specific evaluation in `EVALUATION.md`
4. Both review `LESSONS_LEARNED.md` together

### Gap Remediation

After 2–3 sessions, identify recurring weak dimensions. Go to `GAP_REMEDIATION.md`, find your gap, follow the targeted drill.

## Scoring

Each session scored on 7 dimensions, each 0–3:

| Dimension | 0 | 1 | 2 | 3 |
|-----------|---|---|---|---|
| Requirements | Skipped | Partial | Structured | Complete + scoped |
| Entity Modeling | Wrong | Missing entities | Right entities, weak methods | Right entities, rich behavior |
| Design Patterns | None | Named only | Applied, no justification | Applied + justified |
| SOLID | Multiple violations | 1–2 violations | Minor violations | Clean |
| Extensibility | Requires rewrite | Partial | Mostly extensible | Absorbs new feature cleanly |
| Concurrency | Ignored | Noticed only | Identified + named solution | Full treatment |
| Communication | Silent | Reactive | Proactive | Narrates decisions in real time |

Max score: 21. Target: 15+ before interviewing.

## Practice Progression

| Week | Focus | Problems |
|------|-------|---------|
| 1–2 | Entity modeling, SRP | Parking Lot, Tic-Tac-Toe, Library Management |
| 3–4 | Strategy + Observer | Notification System, Payment System, Stock Ticker |
| 5–6 | State machines | Vending Machine, ATM, Elevator, Order Lifecycle |
| 7–8 | Complex + concurrency | LRU Cache, Rate Limiter, Movie Booking, Splitwise |
| 9+ | Timed mocks | Random from bank, full 45-min sessions |
