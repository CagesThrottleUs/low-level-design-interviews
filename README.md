# Low Level Design Interview Practice Harness

Structured practice system for LLD/OOD interviews — and for becoming a better object-oriented designer.

## What Is This

This repo serves two goals that reinforce each other:

**Goal 1 — Interview Prep:** Score 15+/21 on the evaluation rubric so you can pass LLD rounds at top companies.

**Goal 2 — Developer Growth:** Understand *why* each pattern exists, what problem it solves without it, and how to recognise pattern shapes in production codebases.

The harness gives you:

- **Problem bank** — 18 problems covering all commonly-tested GoF patterns + concurrency primitives
- **Mock interview scripts** — AI or human-playable interviewer scripts with hint progressions
- **PROPOSED_SOLUTION.md** — persistent record of your design per attempt; AI fills it during sessions
- **Evaluation rubric** — 7-dimension scoring aligned with what FAANG interviewers measure
- **Lessons learned** — per-problem common mistakes with code-level fixes
- **Gap remediation** — targeted drills per weak dimension
- **Developer Guide** — deep WHY for every pattern: naive before-code, SOLID mapping, JDK/Spring usage, mental models
- **Pattern files** — quick reference for 12 patterns with trigger signals and real-world examples
- **Progress tracker** — dimension trend log + interview-readiness gate

## Directory Layout

```
README.md                  — this file
EVALUATION_RUBRIC.md       — 7-dimension scoring (authoritative)
GAP_REMEDIATION.md         — targeted drills per weak dimension
DEVELOPER_GUIDE.md         — deep learning layer: SOLID, pattern WHY, JDK/Spring recognition

problems/
  _TEMPLATE/               — copy this to add new problems (5 files)
  NN-problem-name/
    PROBLEM.md             — problem statement (give to candidate)
    MOCK_INTERVIEW.md      — interviewer script (keep private)
    PROPOSED_SOLUTION.md   — candidate's design (AI fills during session; read for scoring)
    SOLUTION_GUIDE.md      — reference solution (reveal after attempt)
    LESSONS_LEARNED.md     — common mistakes + remediation targets

mock-interview/
  HOW_TO_RUN.md            — all 4 session modes including learning mode
  SESSION_SCORECARD.md     — fill-in scorecard template

patterns/
  STRATEGY.md  STATE.md  OBSERVER.md  FACTORY.md
  BUILDER.md  COMPOSITE.md  CHAIN_OF_RESPONSIBILITY.md
  MEDIATOR.md  PROXY.md  TEMPLATE_METHOD.md

progress/
  TRACKER.md               — session log and dimension trend tracker
```

## Problem Bank Overview

| # | Problem | Tier | Patterns | Key Challenge |
| --- | --- | --- | --- | --- |
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

Four modes — see `mock-interview/HOW_TO_RUN.md` for full detail.

### Mode 1: Solo Practice (45 min)

1. Open `PROBLEM.md` for a chosen problem — set timer
2. Fill `PROPOSED_SOLUTION.md` as you design
3. Score yourself with `EVALUATION_RUBRIC.md`
4. Compare against `SOLUTION_GUIDE.md`
5. Check `LESSONS_LEARNED.md` for anti-patterns you hit
6. Log in `progress/TRACKER.md`

### Mode 2: AI Interview (full session)

Say `"Interview me on parking lot"` to Claude Code. The agent:

1. Runs the 45-min session using `MOCK_INTERVIEW.md`
2. Fills `PROPOSED_SOLUTION.md` as you speak
3. Scores and writes evaluation into `PROPOSED_SOLUTION.md`
4. Compares your design against `SOLUTION_GUIDE.md`

### Mode 3: Learning / Deep Dive

Not time-pressured. For becoming a better designer.

- Read `DEVELOPER_GUIDE.md` for pattern mental models and SOLID principles
- Say `"Deep dive Strategy pattern"` to the AI agent for an interactive lesson
- Read `patterns/[PATTERN].md` for trigger signals, structure, JDK/Spring usage

### Gap Remediation

After 2–3 sessions, identify recurring weak dimensions. Go to `GAP_REMEDIATION.md`, find your gap, follow the targeted drill. Or say `"Run the entity modeling drill with me"` to the AI agent.

## Scoring

Each session scored on 7 dimensions, each 0–3:

| Dimension | 0 | 1 | 2 | 3 |
| --- | --- | --- | --- | --- |
| Requirements | Skipped | Partial | Structured | Complete + scoped |
| Entity Modeling | Wrong | Missing entities | Right entities, weak methods | Right entities, rich behavior |
| Design Patterns | None | Named only | Applied, no justification | Applied + justified |
| SOLID | Multiple violations | 1–2 violations | Minor violations | Clean |
| Extensibility | Requires rewrite | Partial | Mostly extensible | Absorbs new feature cleanly |
| Concurrency | Ignored | Noticed only | Identified + named solution | Full treatment |
| Communication | Silent | Reactive | Proactive | Narrates decisions in real time |

Max score: 21. Target: 15+ before interviewing.

## Practice Progression

### Interview Prep Track

| Week | Focus | Problems |
| --- | --- | --- |
| 1–2 | Entity modeling, SRP | 01 Parking Lot, 02 Vending Machine, 16 Data Exporter |
| 3–4 | Strategy + Observer | 08 Notification System, 05 Splitwise, 09 Rate Limiter |
| 5–6 | State machines | 06 ATM, 04 Elevator, 07 Movie Booking |
| 7–8 | Complex + concurrency | 03 LRU Cache, 17 Blocking Queue, 18 Thread Pool |
| 9+ | Full pattern coverage | 11 Pizza, 12 File System, 13 Logging, 14 Chat, 15 Proxy, 10 Chess |
| 10+ | Timed mocks | Any problem, 45-min sessions, AI interviewer |

### Developer Growth Track

| Stage | Focus | Resources |
| --- | --- | --- |
| Awareness | Understand each pattern's intent | `DEVELOPER_GUIDE.md` Part 4 + `patterns/*.md` |
| Application | Apply the right pattern when told | All `SOLUTION_GUIDE.md` files |
| Internalisation | Identify patterns without prompting | Timed solo sessions without hints |
| Intuition | See design shapes in production code | Deep dive mode + real codebase comparison |

## Getting the Most from This Repo

**If you only want to pass the interview:** Run Mode 1 or 2 on the Practice Progression above. Score your sessions. Fix your lowest dimension. Repeat until 15+/21 consistently.

**If you want to become a genuinely better designer:**

1. Start with `DEVELOPER_GUIDE.md` — read Part 3 (SOLID) and Part 4 (pattern mental models)
2. For each problem you practice, after reading `SOLUTION_GUIDE.md`, ask: "Why this way and not another?"
3. Use AI deep dive mode to get the pattern's WHY, not just HOW
4. After each pattern, find where it appears in JDK or a framework you use — the recognition is the skill
