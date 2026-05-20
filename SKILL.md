---
name: goal
description: |
  Socratic interview that converts a vague intention into a crisp, measurable
  definition of done — then emits a canonical Goal Statement the user (or compass)
  can act on.

  Run when:
  - User invokes /goal (standalone task refinement)
  - Compass init offers to define namespace intent
  - Compass DECIDE detects a vague session goal

  Usage:
    /goal                          — blank-slate interview, start from scratch
    /goal "build the auth service" — pre-load a rough intention, skip first question
    /goal --intent                 — intent mode: output suitable for compass intent.md
    /goal --session                — session mode: output is a list of session goals

compatibility: |
  No external tools required — conversational only.
  Output format is parseable by compass (Goal Statement block).
---

## Purpose

Extract the user's real goal through targeted questions. Stop as soon as you have
enough to write a crisp, unambiguous definition of done. Never ask more questions
than necessary.

**Rule:** prefer multiple-choice questions over open-ended ones wherever options
are predictable. Present at most 4 choices. Always include an "Other / none of
these" escape hatch.

---

## Phase 1 — Orientation

### Step 1a — Load context

Check `$ARGUMENTS`:
- If arguments present, treat them as the user's rough intention. Skip Step 1b.
- If `--intent` flag present: set mode = `intent` (output will populate intent.md)
- If `--session` flag present: set mode = `session` (output will be session goals)
- Default mode = `task` (standalone definition of done)

### Step 1b — Opening question (if no arguments)

Ask one open question:

```
What are you trying to do?
(One sentence is enough — I'll ask the follow-ups.)
```

Wait for free-text input. Capture as `raw_intention`.

---

## Phase 2 — Classify the work

From `raw_intention`, infer the work type. If confident (≥80%), set silently and skip
the question. If not confident, ask:

```
What kind of work is this?

A) New feature / capability
B) Bug fix / defect
C) Refactor / tech debt
D) Architecture decision / design
E) Research / investigation
F) Other
```

Capture as `work_type`. This shapes which follow-up questions fire.
Silently accumulate any rejected options as `rejected_work_type`.

---

## Phase 3 — Pin the deliverable

Ask one question shaped by `work_type`. Pick the most relevant variant:

**For feature / bug fix:**
```
What will exist or be true when this is done?

A) Working code — a specific function/service/endpoint behaves correctly
B) Tests pass — a suite that previously failed (or didn't exist) now passes
C) Deployed — the change is live in a named environment
D) Reviewed and merged — PR approved and on main
E) Other
```

**For refactor / tech debt:**
```
How will you know the refactor is complete?

A) The old pattern is gone — no remaining usages
B) All tests still pass — no behaviour change
C) Complexity metric improved — measurable reduction (e.g. class size, cyclomatic)
D) PR reviewed and merged
E) Other
```

**For architecture decision:**
```
What does done look like for a design decision?

A) ADR written and recorded
B) Design reviewed with stakeholders
C) Proof-of-concept built and assessed
D) Decision logged and communicated
E) Other
```

**For research / investigation:**
```
What will you have produced when the investigation is complete?

A) A written summary of findings
B) A recommended course of action
C) A reproducer / minimal example of the problem
D) A confidence level on a hypothesis
E) Other
```

Capture as `deliverable`.
Silently accumulate rejected options as `rejected_deliverables`.

---

## Phase 4 — Success criteria

Ask one question to surface the measurable bar:

```
What are the conditions that must ALL be true for this to count as done?
(List them — one per line. Aim for 2–4. I'll help sharpen them.)
```

Wait for free-text. Capture as `raw_criteria`.

If the user gives vague criteria (e.g. "it works", "looks good", "no bugs"), probe
exactly once per vague item:

```
"<vague criterion>" — what would prove that? 
(e.g. a test passes, a metric is within range, a person signs off)
```

Sharpen the response into a concrete, falsifiable statement before proceeding.

---

## Phase 5 — Quality bar and exclusions

Ask two short follow-ups. Combine into one message to keep it brisk:

```
Two quick ones:

1. What must NOT regress or break as a side-effect?
   (e.g. existing tests, performance baseline, a specific endpoint — or "nothing specific")

2. What's explicitly out of scope?
   (e.g. "not deploying yet", "no UI changes", "not touching auth" — or "nothing to exclude")
```

Capture as `quality_constraints` and `out_of_scope`. If the user says "nothing" / skips,
record as empty — do not invent constraints.

---

## Phase 6 — Synthesise

Construct the Goal Statement from the gathered inputs. Apply these rules:

- **Outcome line:** active present tense — "The auth service validates JWT tokens and
  returns 401 on expiry." Not "Build the auth service."
- **Criteria:** each must be independently falsifiable. If a criterion cannot be
  checked without judgment, sharpen it or flag it.
- **Quality bar:** only include items the user specified. Never pad with boilerplate.
- **Out of scope:** only include items the user specified.
- **Alternatives dismissed:** include when `rejected_work_type` or `rejected_deliverables`
  are non-empty. List each as a one-liner showing what was rejected and in which phase.
  Omit the block entirely if nothing was explicitly rejected during the interview.
- **TDD nudge:** if `work_type` is "New feature" or "Bug fix" and no success criterion
  mentions tests passing, add a single note line after the closing `---` of the Goal
  Statement, before the confirmation prompt.

Present for confirmation:

```
---
## Goal Statement

**Outcome:** <one-line statement of what will be true when done>

**Success criteria (all must be true):**
1. <specific, falsifiable criterion>
2. <specific, falsifiable criterion>
3. <specific, falsifiable criterion>

**Quality bar:**
- <must not break / regress>

**Out of scope:**
- <explicit exclusion>

**Alternatives dismissed:** *(omit this block if nothing was explicitly rejected)*
- Work type: rejected "<option text>"
- Deliverable: rejected "<option text>"
---

*(for New feature / Bug fix work where no success criterion references tests passing — omit otherwise)*
Note: no test criterion specified — consider whether a passing test suite
is an implicit condition of done for this work type.

Does this capture it? 
[Y — lock it in / E — edit a line / R — start over]
```

Wait for response:
- **Y / yes / enter** → proceed to Phase 7
- **E / edit** → ask which line to change; accept replacement text; re-present
- **R / restart** → go back to Phase 1b

---

## Phase 7 — Output and handoff

### Standard output (mode = task)

Print the confirmed Goal Statement block as the final output. No further action.
The user can copy it, hand it to an agent, or use it as a session anchor.

### Intent mode (mode = intent, `--intent` flag or called from compass init)

Reformat the Outcome line as a compass-compatible intent statement:

```
✓ Intent defined:
"<Outcome line>"

Success criteria stored alongside intent for reference.
```

Return the Outcome line as the `GOAL_OUTCOME` value for compass to write into
`intent.md`. The full Goal Statement block goes into a `## Intent criteria` section
appended to `intent.md`.

### Session mode (mode = session, `--session` flag or called from compass DECIDE)

Reformat the success criteria as a numbered goal list for compass session goals:

```
✓ Session goals derived from Goal Statement:
1. <criterion 1>
2. <criterion 2>
3. <criterion 3>
```

Return the criteria list as the `SESSION_GOALS` array for compass to pass to
`compass.py open`.

---

## Interview rules

1. **Brevity over completeness.** If the intention is clear, skip phases you don't need.
   A user who says "fix the NPE in OrderService.processRefund() — test must pass" needs
   only Phase 6 synthesis, not the full interview.

2. **One question at a time** in conversational mode. Never stack 3 questions in one
   message (exception: the Phase 5 double question, which is explicitly combined for
   pace).

3. **No invented criteria.** Never add quality constraints the user didn't mention.
   "Tests should pass" is not a default — only add it if the user said so or the work
   type makes it unavoidable (bug fix where a test reproduces the bug).

4. **Confidence scoring.** For non-trivial deliverables, append a one-liner at the end
   of the Goal Statement:
   ```
   Confidence this captures your intent: <N>/10
   ```
   If < 7: explain which aspect is uncertain and offer to refine it.

5. **Stop at "good enough."** A 4-criteria Goal Statement that is 90% right and
   agreed upon is better than a 7-criteria statement that took 15 exchanges to produce.
