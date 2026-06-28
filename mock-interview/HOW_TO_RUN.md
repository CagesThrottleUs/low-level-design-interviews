# How to Run a Mock Interview Session

Two modes: solo practice, or with an AI agent as interviewer.

---

## Mode 1: Solo (Self-Practice)

**Setup:**
1. Pick a problem from [problems/](../problems/)
2. Open only `PROBLEM.md` — do NOT look at `SOLUTION_GUIDE.md` yet
3. Set a timer: 45 minutes
4. Work on paper or in an IDE (no web browsing)

**During the session:**
1. Read the problem statement
2. Spend 5–8 min on requirements — write down actors, use cases, assumptions
3. Identify key entities and relationships
4. Design class structure (interfaces, abstract classes, concrete classes)
5. Sketch key algorithms or state machines
6. Talk through extensibility and concurrency

**After the session:**
1. Score yourself using `EVALUATION_RUBRIC.md`
2. Open `SOLUTION_GUIDE.md` — compare your design
3. Read `LESSONS_LEARNED.md` — check which mistakes you made
4. Log in `progress/TRACKER.md`
5. Identify your lowest-scoring dimension → run the drill in `GAP_REMEDIATION.md`

---

## Mode 2: With a Partner (Human Interviewer)

**Interviewer setup:**
1. Open `MOCK_INTERVIEW.md` for the chosen problem
2. Read it fully before the session starts
3. Do NOT share it with the candidate

**Session flow:**
- Follow the script phases: Opening → Requirements → Core Design → Extension Probe → Concurrency Probe → Closing
- Use hints in order (least specific first — don't skip to the answer)
- Score each dimension in real time using the checklist at the bottom of the script
- After session: debrief together using `LESSONS_LEARNED.md`

---

## Mode 3: AI Agent as Interviewer

See `AGENTS.md` in the repo root. AI agents can:
- Conduct a full mock interview session (interactive)
- Score your design against the evaluation rubric
- Explain the reference solution
- Identify gaps and recommend remediation

**How to start an AI mock interview:**

Open Claude Code (or another agent harness) and say:

```
Run a mock LRU Cache interview with me.
Use the MOCK_INTERVIEW.md script at problems/03-lru-cache/MOCK_INTERVIEW.md.
You are the interviewer. I am the candidate. Begin.
```

The agent will conduct the session interactively. After you finish your design, ask:

```
Score my design using EVALUATION_RUBRIC.md. 
Then show me the gaps from LESSONS_LEARNED.md that I hit.
```

---

## Session Timing Guide

| Phase | Recommended time | Red flag |
|-------|-----------------|----------|
| Requirements | 5–8 min | < 2 min = jumping to code |
| Entity identification | 5–7 min | > 12 min = over-scoping |
| Class design / patterns | 10–15 min | — |
| Core algorithm | 5–8 min | — |
| Extension + concurrency | 5–7 min | Never reached = ran out of time |
| Buffer | ~3 min | — |

**Total: 45 min.** Practice ending on time. Interviewers penalize running over.

---

## Signs You Are Ready to Interview

- Consistently scoring 15+ / 21 across different problem categories
- Requirements phase completes in 5–7 min with no missing actors
- Can articulate WHY you chose each pattern (not just which pattern)
- Extension questions absorb in < 2 min without redesign
- Concurrency issues raised proactively without prompting
