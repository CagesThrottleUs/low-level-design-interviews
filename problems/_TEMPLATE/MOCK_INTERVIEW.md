# Mock Interview Script — [Problem Name]

**For the interviewer.** Read this before the session. Do not share with candidate.

---

## Pre-Session Setup

- Set timer: 45 min
- Candidate has blank paper or empty IDE
- You have this script open (hidden from candidate)

---

## Opening (2 min)

Read aloud:

> "Today I'd like you to design [system]. Before you start designing, take a moment to ask any clarifying questions you have. I want to understand how you think through the problem."

Wait for candidate to ask questions. Use this table:

| If candidate asks | Your answer |
|------------------|-------------|
| [Expected question 1] | [Answer to give] |
| [Expected question 2] | [Answer to give] |
| [Expected question 3] | [Answer to give] |
| Anything not listed | "That's a reasonable assumption to make. Go with what you think is appropriate." |

**If candidate does NOT ask any questions and starts designing immediately:** Note this as a requirements gap (Dimension 1 penalty). Let them proceed — do not prompt. You can ask at the end: "Before we wrap up, what assumptions did you make?"

---

## Phase 1: Requirements Discussion (target: 5–8 min from start)

Good signal: candidate lists actors, use cases, explicitly states assumptions.
Bad signal: no questions, or questions are trivial implementation details (not design-relevant).

If candidate finishes requirements fast (< 3 min), ask:
> "Did you consider [missing actor or use case]?"

---

## Phase 2: Core Design (target: 20–25 min)

Let candidate drive. Do not interrupt unless they are stuck for 2+ minutes. Then use a hint.

**Hints by scenario (give in order — least specific first):**

If stuck on entity identification:
1. "What are the main things in this system? What nouns would you use to describe them?"
2. "What does a [key entity] need to be able to do?"
3. "Think about [specific entity] — what state does it hold? What can it do?"

If stuck on patterns:
1. "How would your design handle adding a new [type X] in the future?"
2. "Is there a way to avoid that switch statement?"
3. "Have you considered using a Strategy pattern here?"

If stuck on extensibility:
1. "What happens if we add a new [feature]? How many classes would you change?"
2. "Is there a way to add [feature] without touching existing code?"

---

## Phase 3: Extension Probe (target: 5–8 min from end)

At the ~35 min mark, say:

> "[Extension question — specific to this problem]"

Observe: does the design absorb it cleanly or require major rework?

Score this as Extensibility dimension.

---

## Phase 4: Concurrency Probe (if not raised by candidate)

If candidate never mentioned concurrency, at ~40 min ask:

> "What happens if two users try to [concurrent operation] at the same time?"

Score based on response (see Dimension 6 in EVALUATION_RUBRIC.md).

---

## Closing (last 2 min)

> "We're almost out of time. Is there anything you'd change if you had more time?"

Good answers: candidate identifies their own weaknesses — shows self-awareness.

---

## Post-Session Debrief

After timer ends, reveal scores per dimension. Walk through:
1. What was strongest
2. What was weakest
3. One specific thing to practice before next session

Open `LESSONS_LEARNED.md` together and identify which anti-patterns were hit.

---

## Scoring Checklist

| Dimension | Notes | Score (0–3) |
|-----------|-------|-------------|
| Requirements | | |
| Entity Modeling | | |
| Design Patterns | | |
| SOLID | | |
| Extensibility | | |
| Concurrency | | |
| Communication | | |
| **TOTAL** | | **/21** |
