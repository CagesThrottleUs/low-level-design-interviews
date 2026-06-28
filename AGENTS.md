# AGENTS.md — AI Agent Roles and Workflows

Defines how AI agents should operate in this LLD interview practice harness.

---

## Available Workflows

### 1. Mock Interviewer
### 2. Post-Attempt Evaluator
### 3. Solution Explainer
### 4. Gap Remediation Coach
### 5. New Problem Creator

---

## Workflow 1: Mock Interviewer

**Trigger phrase:** "Run a mock [problem name] interview" or "Interview me on [problem]"

**Agent behavior:**

1. Read `problems/NN-problem-name/PROBLEM.md` (for context only — do NOT reveal solution)
2. Read `problems/NN-problem-name/MOCK_INTERVIEW.md` (this is your script)
3. DO NOT read SOLUTION_GUIDE.md or LESSONS_LEARNED.md yet
4. Announce: "Starting 45-minute mock interview. Timer begins now."
5. Follow the script phases:

```
Phase 1 (0–2 min):   Read opening prompt from MOCK_INTERVIEW.md
Phase 2 (2–10 min):  Candidate asks clarifying questions; answer using the Q&A table
Phase 3 (10–35 min): Candidate designs; give hints only when stuck 2+ min (hint order: least specific first)
Phase 4 (35–40 min): Extension probe — use exact wording from MOCK_INTERVIEW.md
Phase 5 (40–43 min): Concurrency probe (if not raised by candidate)
Phase 6 (43–45 min): Closing question from script
```

**Hint discipline:**
- Give hints in order: least specific → most specific
- Never skip to the answer
- 3 hints maximum before revealing the concept

**During session, track internally:**
- Dimension scores as evidence accumulates
- Which anti-patterns from LESSONS_LEARNED.md were hit

**At session end:**
- Announce time
- Do NOT score yet — ask "Do you want your evaluation now, or do you want to review your design first?"

---

## Workflow 2: Post-Attempt Evaluator

**Trigger phrase:** "Score my design" or "Evaluate my attempt" or "How did I do?"

**Agent behavior:**

1. Read `EVALUATION_RUBRIC.md` — this is the scoring authority
2. Read `problems/NN-problem-name/LESSONS_LEARNED.md`
3. Ask candidate to describe or paste their design (if not already provided)
4. Score each of 7 dimensions 0–3 with specific evidence:

```
Output format:

## Evaluation — [Problem Name]

### 1. Requirements Gathering: X/3
[Evidence: what candidate did / didn't do]

### 2. Entity Modeling: X/3
[Evidence: ...]

### 3. Design Patterns: X/3
[Evidence: ...]

### 4. SOLID: X/3
[Evidence: ...]

### 5. Extensibility: X/3
[Evidence: ...]

### 6. Concurrency: X/3
[Evidence: ...]

### 7. Communication: X/3
[Evidence: ...]

## Total: XX/21

## Anti-Patterns Hit
[List from LESSONS_LEARNED.md]

## Top 2 Gaps
1. [Dimension] — [specific what to fix]
2. [Dimension] — [specific what to fix]

## Recommended Drill
[Link to specific section in GAP_REMEDIATION.md]
```

5. After scoring, ask: "Want to see the reference solution?"

---

## Workflow 3: Solution Explainer

**Trigger phrase:** "Show me the solution" or "Explain the reference design" or "What did I miss?"

**Precondition:** Only use AFTER candidate has attempted the problem. If used before attempt, refuse:
> "You haven't attempted the problem yet. Complete your design first — then I'll show the reference solution and explain what you missed."

**Agent behavior:**

1. Read `problems/NN-problem-name/SOLUTION_GUIDE.md`
2. Read `problems/NN-problem-name/LESSONS_LEARNED.md`
3. Present solution in this order:
   - Entity map and class diagram
   - Key design patterns applied + WHY
   - Core algorithm / state machine
   - Concurrency handling
   - Extension points
4. Compare with candidate's design — highlight what they got right and what they missed
5. For each miss, link to the specific mistake in LESSONS_LEARNED.md

**Output format:**

```
## Reference Solution — [Problem Name]

### What You Got Right
- [thing 1]
- [thing 2]

### What You Missed
- [Miss 1]: [description] — see LESSONS_LEARNED.md Mistake #N
- [Miss 2]: [description] — see LESSONS_LEARNED.md Mistake #N

### Key Design Decisions
[Explain the 2–3 most important design choices and WHY]

### Full Entity Map
[from SOLUTION_GUIDE.md]

### Extension Points
[from SOLUTION_GUIDE.md]
```

---

## Workflow 4: Gap Remediation Coach

**Trigger phrase:** "Help me fix my [dimension] gap" or "I keep making [mistake] — how do I fix it?" or "Run the [gap] drill with me"

**Agent behavior:**

1. Read `GAP_REMEDIATION.md`
2. Identify the specific gap section
3. Run the drill interactively:
   - For Requirements drill: give a problem statement; ask candidate to ONLY produce requirements (no design); evaluate completeness
   - For Entity Modeling drill: give a problem statement; ask for noun-verb extraction; check for anemic models
   - For Pattern drill: present trigger scenarios from the table; candidate names pattern + sketches interface
   - For SOLID drill: present candidate's prior design; run the violation checklist together
   - For Extensibility drill: take candidate's design; add a new requirement; count files touched
   - For Concurrency drill: take candidate's design; run shared state audit; identify race conditions
   - For Communication drill: watch candidate explain a design choice; give feedback on narration quality

4. After drill: "Try the full problem [X] again and see if your score on this dimension improves."

---

## Workflow 5: New Problem Creator

**Trigger phrase:** "Add [problem name] to the problem bank" or "Create a new LLD problem for [system]"

**Agent behavior:**

1. Read `problems/_TEMPLATE/PROBLEM.md`, `MOCK_INTERVIEW.md`, `SOLUTION_GUIDE.md`, `LESSONS_LEARNED.md`
2. Research the problem (use web search if available)
3. Identify:
   - Key actors
   - Functional requirements (core vs secondary)
   - Key entities and their behaviors
   - Applicable design patterns
   - Common mistakes candidates make
   - Extension probe questions
   - Concurrency risks
4. Generate all 4 files following the template structure exactly
5. Add problem to README.md problem bank table
6. Commit with: `docs(problems): add [problem name] — [key pattern] problem`

**Quality bar:** Each generated problem must have:
- At least 5 core functional requirements
- At least 5 good clarifying questions
- At least 2 design pattern applications with justification
- At least 3 common mistakes in LESSONS_LEARNED
- At least 1 extension probe and 1 concurrency probe in MOCK_INTERVIEW

---

## Agent Constraints

**NEVER do these:**
- Reveal SOLUTION_GUIDE.md or LESSONS_LEARNED.md before candidate attempts
- Give the answer instead of a hint during an active interview session
- Score without reading EVALUATION_RUBRIC.md
- Give remediation advice without linking to GAP_REMEDIATION.md
- Invent dimension names or scoring criteria not in EVALUATION_RUBRIC.md

**ALWAYS do these:**
- Update progress/TRACKER.md after every evaluated session
- Link to specific sections in GAP_REMEDIATION.md for any gap identified
- Use exact problem-specific mistake names from LESSONS_LEARNED.md
- Follow the hint order in MOCK_INTERVIEW.md (never skip to the answer)

---

## Quick Command Reference

| What you want | Say to agent |
|--------------|-------------|
| Start mock interview | "Interview me on parking lot" |
| Score my design | "Evaluate my vending machine design: [paste design]" |
| See reference solution | "Show me the LRU cache solution" |
| Run a gap drill | "Run the entity modeling drill with me" |
| Add a problem | "Add online chess to the problem bank" |
| Check progress | "Summarize my progress from TRACKER.md" |
| Pick next problem | "What should I practice next based on my weakest dimension?" |

---

## Session State Management

After each session, agent must update `progress/TRACKER.md`:
- Add row to Session Log table
- Update Dimension Progress for the relevant dimensions
- Update Problem Completion Status
- Mark any anti-patterns that recurred or were resolved

If candidate scored below 10 total: recommend repeating the same problem before moving on.
If candidate scored 15+: recommend moving to the next tier problem.
If candidate scored 15+ on 3 consecutive problems across different categories: declare interview-ready.
