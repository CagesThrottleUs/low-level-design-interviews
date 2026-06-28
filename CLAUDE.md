# CLAUDE.md — LLD Interview Practice Harness

Context for AI agents operating in this repository.

---

## Repository Purpose

Personal practice harness for low-level design (LLD) / object-oriented design (OOD) interviews. Two goals:
1. **Interview prep** — score 15+/21 on the evaluation rubric
2. **Developer growth** — understand the WHY behind patterns and principles

AI agents operate in one of two modes: **Interview Mode** (interviewer, evaluator, solution explainer, gap coach) or **Learning Mode** (deep dive teacher). See `AGENTS.md` for full workflow definitions.

---

## Directory Structure

```
README.md                    — overview, problem bank, pattern + concurrency coverage
EVALUATION_RUBRIC.md         — 7-dimension scoring system (authoritative)
GAP_REMEDIATION.md           — targeted drills per dimension gap
DEVELOPER_GUIDE.md           — deep learning layer: WHY patterns exist, SOLID explained,
                               mental models, JDK/Spring recognition, learning paths
CLAUDE.md                    — this file (AI agent context)
AGENTS.md                    — agent role definitions and 6 workflows

problems/
  _TEMPLATE/                 — blank template for new problems (5 files)
  NN-problem-name/
    PROBLEM.md               — problem statement (share with candidate)
    MOCK_INTERVIEW.md        — interviewer script (do NOT share with candidate)
    PROPOSED_SOLUTION.md     — candidate's design (AI writes during session; AI reads for scoring)
    SOLUTION_GUIDE.md        — reference solution (reveal after PROPOSED_SOLUTION.md is filled)
    LESSONS_LEARNED.md       — common mistakes + remediation targets

mock-interview/
  HOW_TO_RUN.md              — all session modes including learning mode
  SESSION_SCORECARD.md       — fill-in scorecard template

patterns/
  STRATEGY.md                — trigger signals, structure, LLD examples, JDK usage
  STATE.md
  OBSERVER.md
  FACTORY.md
  BUILDER.md
  COMPOSITE.md
  CHAIN_OF_RESPONSIBILITY.md
  MEDIATOR.md
  PROXY.md
  TEMPLATE_METHOD.md

progress/
  TRACKER.md                 — session log and dimension trend tracker
```

---

## The PROPOSED_SOLUTION.md Workflow

This file is the persistent record of what a candidate actually designed.

| Phase | Who acts | What happens |
| --- | --- | --- |
| During mock interview | AI Interviewer | Fills PROPOSED_SOLUTION.md incrementally per phase |
| Solo practice | Candidate | Fills PROPOSED_SOLUTION.md manually |
| Post-attempt | AI Evaluator | Reads PROPOSED_SOLUTION.md; scores against rubric; writes evaluation into the file |
| Solution reveal | AI Solution Explainer | Compares PROPOSED_SOLUTION.md with SOLUTION_GUIDE.md; explains diffs |
| Learning | AI Deep Dive | References PROPOSED_SOLUTION.md to show what the candidate did and why the pattern fixes it |

**Critical rule:** Never reveal SOLUTION_GUIDE.md until PROPOSED_SOLUTION.md has been filled.

---

## Evaluation System

7 dimensions, each scored 0–3. Source of truth: `EVALUATION_RUBRIC.md`.

| # | Dimension | What it measures |
| --- | --- | --- |
| 1 | Requirements | Did candidate ask the right questions and scope correctly? |
| 2 | Entity Modeling | Right classes with rich behavior (not anemic data bags)? |
| 3 | Design Patterns | Patterns applied correctly with justification? |
| 4 | SOLID | No god classes, OCP respected, DIP applied? |
| 5 | Extensibility | New feature absorbed without touching existing code? |
| 6 | Concurrency | Shared state identified, mechanism chosen? |
| 7 | Communication | Decisions narrated proactively with reasoning? |

Max: 21. Interview-ready threshold: 15+.

---

## Problem Bank (18 problems)

| # | Problem | Tier | Key Patterns |
| --- | --- | --- | --- |
| 01 | Parking Lot | Foundation | Strategy, Polymorphism |
| 02 | Vending Machine | Foundation | State |
| 03 | LRU Cache | Foundation | DS + Concurrency |
| 04 | Elevator System | Foundation | State, Strategy |
| 05 | Splitwise | Intermediate | Strategy, Observer |
| 06 | ATM System | Foundation-Intermediate | State, Strategy |
| 07 | Movie Ticket Booking | Intermediate | State, Strategy |
| 08 | Notification System | Intermediate | Strategy, Decorator |
| 09 | Rate Limiter | Intermediate | Strategy |
| 10 | Chess Game | Advanced | Command, Polymorphism |
| 11 | Pizza Ordering | Foundation | Builder |
| 12 | In-Memory File System | Intermediate | Composite |
| 13 | Logging Framework | Intermediate | Chain of Responsibility, Strategy |
| 14 | Chat Room | Intermediate | Mediator |
| 15 | Caching Proxy | Intermediate | Proxy |
| 16 | Data Exporter | Foundation | Template Method |
| 17 | Bounded Blocking Queue | Advanced | Producer-Consumer |
| 18 | Custom Thread Pool | Advanced | Thread Pool, State Machine |

---

## Agent Behavioral Rules

1. Fill `PROPOSED_SOLUTION.md` incrementally during mock interview — not after
2. Never reveal `SOLUTION_GUIDE.md` until `PROPOSED_SOLUTION.md` has content
3. `MOCK_INTERVIEW.md` is for the AI interviewer only — never share with candidate
4. Scoring always uses `EVALUATION_RUBRIC.md` — do not invent criteria
5. Write AI evaluation output into the **AI Evaluation Output** section of `PROPOSED_SOLUTION.md`
6. Remediation always links to `GAP_REMEDIATION.md` sections — not ad-hoc advice
7. Update `progress/TRACKER.md` after every evaluated session
8. After Solution Explainer, offer Deep Learning Mode — say "Type 'deep dive [pattern]' to understand why"

---

## Adding New Problems

1. Copy `problems/_TEMPLATE/` to `problems/NN-problem-name/`
2. Fill in: PROBLEM, MOCK_INTERVIEW, SOLUTION_GUIDE, LESSONS_LEARNED
3. Leave PROPOSED_SOLUTION.md blank (candidate fills during practice)
4. Add to README.md problem bank table, Pattern Coverage table, and Concurrency Coverage table
5. Commit: `docs(problems): add [name] — [key pattern] problem`

---

## Key Design Patterns — Quick Lookup

Full details in `patterns/`. Deep WHY in `DEVELOPER_GUIDE.md`.

| Pattern | Trigger signal | Core interface |
| --- | --- | --- |
| Strategy | "multiple algorithms for same operation" | `interface XStrategy { R execute(P); }` |
| Observer | "multiple listeners for same event" | `interface XObserver { void onEvent(D); }` |
| State | "behavior differs per current mode" | `interface XState { void op1(); void op2(); }` |
| Factory | "create objects without knowing concrete type" | `static X create(type, params)` |
| Builder | "complex object, many optional params" | `class XBuilder { XBuilder field(v); X build(); }` |
| Composite | "tree: treat leaf and composite uniformly" | `interface Component { int size(); }` |
| Chain of Responsibility | "multiple handlers, pass until one accepts" | `abstract void handle(R req); void setNext(H)` |
| Mediator | "N components communicate through one hub" | `interface Mediator { void notify(sender, event); }` |
| Proxy | "control access, same interface as real" | `class Proxy implements Service { delegate.method(); }` |
| Template Method | "fixed step order, variable step impl" | `abstract class X { final void run() { s1(); s2(); } }` |
| Command | "encapsulate operation for undo/redo" | `interface Command { void execute(); void undo(); }` |
| Decorator | "add behavior at runtime, same interface" | `class D implements I { I delegate; delegate.method(); }` |
