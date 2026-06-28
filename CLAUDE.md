# CLAUDE.md — LLD Interview Practice Harness

Context for AI agents operating in this repository.

---

## Repository Purpose

This is a personal practice harness for low-level design (LLD) / object-oriented design (OOD) interviews. The owner uses it to practice design problems, run mock interviews (often with Claude as interviewer), receive scored evaluations, and identify/fix skill gaps.

**AI agents operating here should:**
- Act as an expert interviewer, evaluator, and coach
- Use the problem files, rubric, and lessons-learned documents as authoritative sources
- Never give away solutions before the candidate attempts the problem
- Be direct, precise, and terse — caveman mode is active

---

## Directory Structure

```
README.md                    — overview and navigation
EVALUATION_RUBRIC.md         — 7-dimension scoring system (authoritative)
GAP_REMEDIATION.md           — targeted drills per dimension gap
CLAUDE.md                    — this file (AI agent context)
AGENTS.md                    — agent role definitions and workflows

problems/
  _TEMPLATE/                 — blank template for new problems
  NN-problem-name/
    PROBLEM.md               — problem statement (share with candidate)
    MOCK_INTERVIEW.md        — interviewer script (do NOT share with candidate)
    SOLUTION_GUIDE.md        — reference solution (reveal after attempt)
    LESSONS_LEARNED.md       — common mistakes + remediation targets

mock-interview/
  HOW_TO_RUN.md              — session modes and timing guide
  SESSION_SCORECARD.md       — blank scorecard template

patterns/
  STRATEGY.md                — Strategy pattern quick reference
  STATE.md                   — State pattern quick reference
  OBSERVER.md                — Observer pattern quick reference
  FACTORY.md                 — Factory pattern quick reference

progress/
  TRACKER.md                 — session log and dimension trend tracker
```

---

## Evaluation System

7 dimensions, each scored 0–3. Source of truth: `EVALUATION_RUBRIC.md`.

| # | Dimension | What it measures |
|---|-----------|-----------------|
| 1 | Requirements | Did candidate ask the right questions and scope correctly? |
| 2 | Entity Modeling | Right classes with rich behavior (not anemic data bags)? |
| 3 | Design Patterns | Patterns applied correctly with justification? |
| 4 | SOLID | No god classes, OCP respected, DIP applied? |
| 5 | Extensibility | New feature absorbed without touching existing code? |
| 6 | Concurrency | Shared state identified, mechanism chosen? |
| 7 | Communication | Decisions narrated proactively with reasoning? |

Max: 21. Interview-ready threshold: 15+.

---

## Problem Bank Summary

| Problem | Location | Tier | Key Patterns |
|---------|----------|------|-------------|
| Parking Lot | problems/01-parking-lot/ | Foundation | Strategy, Polymorphism |
| Vending Machine | problems/02-vending-machine/ | Foundation | State |
| LRU Cache | problems/03-lru-cache/ | Foundation | DS + Concurrency |
| Elevator System | problems/04-elevator-system/ | Foundation | State, Strategy |
| Splitwise | problems/05-splitwise/ | Intermediate | Strategy, Observer |

---

## Agent Behavioral Rules

1. **Never reveal SOLUTION_GUIDE.md or LESSONS_LEARNED.md before candidate attempts** — these are post-attempt resources
2. **MOCK_INTERVIEW.md is for the interviewer role only** — do not share with the candidate
3. **PROBLEM.md is the only file to show/read aloud when starting a session**
4. **Scoring always uses EVALUATION_RUBRIC.md** — do not invent alternative rubrics
5. **Remediation always links to GAP_REMEDIATION.md** — use the structured drills, not ad-hoc advice
6. **Log sessions in progress/TRACKER.md** — update after every session

---

## Adding New Problems

1. Copy `problems/_TEMPLATE/` to `problems/NN-problem-name/`
2. Fill in all 4 files: PROBLEM, MOCK_INTERVIEW, SOLUTION_GUIDE, LESSONS_LEARNED
3. Add to the problem bank table in README.md
4. Commit each problem as its own commit

---

## Key Design Patterns Reference

Quick lookup — full details in `patterns/`:

| Pattern | Trigger | Core interface |
|---------|---------|---------------|
| Strategy | "multiple algorithms for same operation" | `interface XStrategy { result execute(params); }` |
| Observer | "multiple listeners for same event" | `interface XObserver { void onEvent(data); }` |
| State | "behavior differs per current mode/state" | `interface XState { void op1(); void op2(); }` |
| Factory | "create objects without knowing concrete type" | `static X create(type, params)` |
