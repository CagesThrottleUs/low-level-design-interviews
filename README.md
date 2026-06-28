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
| 06 | Movie Ticket Booking | Intermediate | State, Strategy | Seat selection, concurrent booking |
| 07 | Notification System | Intermediate | Observer, Strategy | Multi-channel, retry logic |
| 08 | Rate Limiter | Intermediate | Strategy | Token bucket vs sliding window |
| 09 | Chess Game | Advanced | Command, State | Move validation, turn management |
| 10 | Ride Share (Uber) | Advanced | Observer, Strategy | Driver matching, ride lifecycle |

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
