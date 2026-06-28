# How to Run a Session

Four modes. Pick based on your goal.

| Goal | Mode |
| --- | --- |
| Practice under time pressure | Solo (Mode 1) |
| Simulate a real interview with pressure and hints | Partner (Mode 2) |
| Full AI-driven interview with automatic scoring | AI Interview (Mode 3) |
| Understand WHY a pattern exists, become a better designer | Learning (Mode 4) |

---

## Mode 1: Solo Practice (45 min)

Best for: first attempt at a problem, building habits.

**Setup:**

1. Pick a problem from [problems/](../problems/)
2. Open only `PROBLEM.md` — do NOT open SOLUTION_GUIDE.md yet
3. Set a 45-minute timer
4. Open `PROPOSED_SOLUTION.md` in the same directory — you will fill this as you work

**During the session (follow this order):**

1. **Requirements (5–8 min):** Write actors, use cases, assumptions, out-of-scope into `PROPOSED_SOLUTION.md` Requirements section
2. **Entity design (5–7 min):** Identify classes, interfaces, enums — who owns what behavior
3. **Class design + patterns (10–15 min):** Sketch method signatures, relationships, apply patterns
4. **Core algorithm (5–8 min):** Key logic, state machines, data structures
5. **Extensibility + concurrency (5–7 min):** "What if we add X?" + "What race conditions exist?"

**After the session:**

1. Complete `PROPOSED_SOLUTION.md` with your actual design
2. Score yourself using `EVALUATION_RUBRIC.md` — fill the Post-Session Self-Score table
3. Open `SOLUTION_GUIDE.md` — compare with reference design section by section
4. Read `LESSONS_LEARNED.md` — check which anti-patterns you hit
5. Log in `progress/TRACKER.md`
6. Identify lowest-scoring dimension → run the drill in `GAP_REMEDIATION.md`
7. **Optional:** Read `DEVELOPER_GUIDE.md` Part 4 for the WHY behind the pattern used

---

## Mode 2: Partner Session (45 min)

Best for: interview simulation, getting real-time feedback.

**Interviewer setup:**

1. Open `MOCK_INTERVIEW.md` for the chosen problem — read it fully
2. Do NOT share it with the candidate
3. Prepare a timer

**Session flow:**

- Follow script phases: Opening → Requirements → Core Design → Extension Probe → Concurrency Probe → Closing
- Give hints in order (least specific first — never skip to the answer)
- Fill `PROPOSED_SOLUTION.md` as candidate speaks (or have candidate fill it)
- Score each dimension in real time using the checklist at the bottom of the script

**After the session:**

- Debrief together using `LESSONS_LEARNED.md`
- Both read `SOLUTION_GUIDE.md` together
- Log score in `progress/TRACKER.md`

---

## Mode 3: AI Interview (Full Session)

Best for: no partner available, want consistent scoring, want a persistent record.

**How to start:**

Open Claude Code and say:

```
Interview me on [problem name].
```

The agent will:

1. Read `PROBLEM.md` and `MOCK_INTERVIEW.md`
2. Announce the start and run the 45-minute session interactively
3. **Fill `PROPOSED_SOLUTION.md` in real time** as you describe your design
4. After session: score your design and write the evaluation into `PROPOSED_SOLUTION.md`

**After the interview, ask:**

```
Show me the reference solution and what I missed.
```

The agent reads both `PROPOSED_SOLUTION.md` (your design) and `SOLUTION_GUIDE.md` (reference) and gives you a direct comparison.

**Commands during/after a session:**

```
"Interview me on parking lot"
"Evaluate my attempt"
"Show me what I missed"
"Run the entity modeling drill with me"
"What should I practice next?"
```

---

## Mode 4: Learning / Deep Dive

Best for: understanding WHY patterns exist, reading DEVELOPER_GUIDE.md, becoming a better designer beyond interview prep.

**This mode is not time-pressured.** Take as long as you need.

### Option A: Pattern-first

1. Read `DEVELOPER_GUIDE.md` Part 4 for the pattern's mental model
2. Read `patterns/[PATTERN].md` for trigger signals, structure, JDK examples
3. Find a problem using that pattern — read `SOLUTION_GUIDE.md`
4. Implement the problem from scratch (no timer)
5. Read `LESSONS_LEARNED.md` to see what you missed

### Option B: Problem-first

1. Read `PROBLEM.md` for a problem you find interesting
2. Try designing it (15–20 min, no timer pressure)
3. Read `SOLUTION_GUIDE.md` — for each decision, ask: "Why this way and not another?"
4. Read `DEVELOPER_GUIDE.md` Part 4 for the deeper reasoning
5. Find another problem using the same pattern — see the shape in a different context

### Option C: AI-guided deep dive

```
"Deep dive Strategy pattern"
"Why does the Builder pattern exist?"
"Teach me the Observer pattern"
"What's the difference between Proxy and Decorator?"
"Where does JDK use Template Method?"
```

The agent reads `DEVELOPER_GUIDE.md` and the relevant `patterns/` file and teaches interactively — showing naive code before the pattern, the transformation step by step, and real-world usage.

### Option D: Principle-first (most rigorous)

1. Pick a SOLID principle from `DEVELOPER_GUIDE.md` Part 3
2. Find a violation of that principle in your own production code
3. Find the problem in this repo that most directly shows the fix
4. Read `SOLUTION_GUIDE.md` and apply the fix mentally to your own codebase
5. Implement it — real practice beats simulated practice

---

## Session Timing Guide

| Phase | Target | Red flag |
| --- | --- | --- |
| Requirements | 5–8 min | < 2 min = jumping to code |
| Entity identification | 5–7 min | > 12 min = over-scoping |
| Class design + patterns | 10–15 min | — |
| Core algorithm | 5–8 min | — |
| Extension + concurrency | 5–7 min | Never reached = ran out of time |
| Buffer | ~3 min | — |

**Total: 45 min.** Practice ending on time. Interviewers penalise running over.

---

## Signs You Are Ready to Interview

- Consistently scoring 15+/21 across different problem categories
- Requirements phase completes in 5–7 min with no missing actors
- Can articulate WHY you chose each pattern (not just which one)
- Extension questions absorbed in < 2 min without redesign
- Concurrency issues raised proactively without prompting

## Signs You Are a Better Designer (beyond interviews)

- You recognise pattern shapes in production code without being told the pattern name
- When adding a feature, you automatically ask "how many files will I touch?"
- You reach for an interface before a concrete class by instinct
- You identify shared mutable state before writing concurrent code
- You can explain WHY a design decision to a junior developer in one sentence
