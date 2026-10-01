---
name: planning
description: Harness step 1. Plans a feature into features/<slug>/THESIS.md and features/<slug>/PLAN.md through a two-stage discussion (THESIS, then PLAN) and commits both to main once the user agrees. Also runs a replan (/planning <slug>) after /execute stopped on a problem the user confirmed. Only run when the user invokes it.
disable-model-invocation: true
argument-hint: <one sentence of intent> | <slug>
---

# /planning

Input: `$ARGUMENTS`

These instructions govern the whole session. If context gets compacted and anything here seems missing, re-read `${CLAUDE_SKILL_DIR}/SKILL.md` before continuing.

**Trunk:** `main`, unless CLAUDE.md names another branch. Wherever this file says "main", read it as the trunk.

---

## Pick the mode

Match the input to one of these three cases, checking them in order.

- **Replan:** the input is an existing feature's slug, and /execute stopped that feature on a problem the user confirmed. → Go to *Replan mode*.
  - How to tell: branch `<slug>` exists, and on that branch the last entry in `features/<slug>/EXECUTION.md` is a `## Problem:` entry marked `**Ruling:** Confirmed.`
- **Existing slug, no confirmed problem:** the input names an existing `features/<slug>/` folder, but the Replan check fails. → Tell the user what state the feature is in (planned, executing, or merged) and stop. Replans only follow a confirmed problem.
- **Anything else:** a new feature; the input is the intent. → Go to *Initial planning*.
  - You must be on main. If you aren't, say so and ask the user to switch. Don't switch a dirty tree yourself.

---

# Initial planning

## Before your first reply

Read these without announcing it:

1. `learnings/LEARNINGS.md`, if it exists.
2. Whatever is relevant to the intent: CLAUDE.md, the code, and any `features/*/` folder on the same topic.

Your first message engages directly with the topic and works toward filling THESIS. Don't list what you read.

---

## Stage 1: THESIS

Purpose: agree on what the feature is for and what could stand in its way, before discussing how to build it.

1. **Goal.** Ask questions until you can state the Goal back in your own words. The user confirms it.
2. **Risks.** Read the relevant code, then raise Risks from the code and from LEARNINGS.md. When a learning shapes a Risk or a Decision, cite its ID ("per L07").
3. **Decisions.** Propose one only when the code or the discussion makes it imperative. The user approves each one.
   - When the user states something firmly ("this can't change the public API"), offer it as a Decision. This conversation is gone after `/clear`; anything not written down is lost.
4. **Slug.** Propose one: kebab-case, derived from the Goal. The user can change it.
5. **Write** `features/<slug>/THESIS.md`. The user adjusts it.

**THESIS is done when:**
- every section is filled or deliberately left empty, and
- the user explicitly agrees the Goal says what they meant. Ask for this agreement. Don't start PLAN without it.

### THESIS.md format

This is the only definition.

```markdown
## Goal
What we're trying to achieve and why. The general direction of the work, without prescribing implementation.

## Risks
Known or reasonably foreseeable problems that might stand in the way.

## Decisions
High-level choices or principles already decided for this work that should be adhered to.

## Notes
{optional extra context that didn't fit prior categories, only if necessary}
```

### THESIS rules

- **The Decision test:** would breaking this be worth stopping work and reopening THESIS? Yes → it's a Decision. No → it belongs in PLAN, or nowhere.
  - Why the bar is high: during /execute, breaking a Decision stops all work. Departing from a PLAN step while still meeting every DoD is only logged as a Deviation, and work continues.
- Decisions are optional; most features have none. An empty section says `None.`
- A Risk that materializes during execution was foreseen, so it never counts as a THESIS problem.

---

## Stage 2: PLAN

Purpose: turn the agreed THESIS into ordered steps, each with observable proof that it's done.

Starts only after THESIS is agreed, in the same context window.

1. **Discuss the shape.** In chat, propose:
   - the Approach: one or two sentences on how the steps build on each other, and
   - the step outline: each step's header, plus one line on what it will prove.

   Also name any behavior the plan will accept or settle (edge cases, error handling, known limitations), so each one gets a DoD. Adjust with the user until they agree on the shape. Don't write the file before that.
2. **Draft** `features/<slug>/PLAN.md` from the agreed shape, complete with every DoD and Verify line.
3. **Mark** with `(you)` any Verify line that /execute can't check on its own (see the Verify rule below).
4. The user adjusts it if needed.

**PLAN is done when:**
- The Approach says how the steps build on each other, without repeating Risks or DoDs.
- Done when states the feature-level DoD.
- Every step has Areas and at least one DoD/Verify pair.
- Every step has an observable outcome of its own.
- Every Risk, and every behavior the plan accepts or settles, is guarded by at least one DoD.
- Every Verify line only the user can check is marked `(you)`.

**Size check:** more than 8 steps → suggest splitting the feature. It's a signal, not a rule.

### PLAN.md format

This is the only definition.

```markdown
## Approach
Record activity on each request first, then enforce expiry on top of it, then make the timeout configurable.

## Done when
- **DoD:** A session idle past the timeout gets a 401 on its next request; active sessions are unaffected.
- **Verify:** Run with SESSION_TIMEOUT=2s; authenticated request, wait 3s, request again → 401. Requests every 1s → 200 throughout.

## Steps
### Step 1: Record last-activity time on each authenticated request
- **Areas:** auth middleware, session store
- **DoD:** Each authenticated request advances the session's last-activity time.
- **Verify:** Two authenticated requests 1s apart; stored last-activity advances between them.
```

### PLAN rules

- **Approach** is the opener for the Steps: how they build on each other. It never names mechanisms or repeats what Risks and DoDs already say. Everything concrete lives in the Steps.
- **Done when** is the feature-level DoD: the end-to-end behavior that shows the Goal is met.
- **A DoD is observable behavior with specifics,** stated at the edge of the system: what a caller outside the code sees, such as requests, commands, public functions, outputs, and files produced. Internal helpers and data structures stay out, because DoDs can't change during execution and naming them would lock in the implementation. "Timeout works correctly" isn't a DoD.
- **Anything the plan accepts or settles is a DoD.** If it isn't a DoD, /execute won't test it and review won't check it.
- **A Verify line says how the DoD will be proven** with concrete inputs and expected results. /execute writes tests that implement it.
  - A Verify line only the user can check (it needs human judgment, or touches something off-limits) starts with `(you)`. These should be rare.
- **Every step has at least one DoD/Verify pair.** A step may have several; list them as repeated `- **DoD:**` / `- **Verify:**` lines.
  - A step with no observable outcome of its own (extracting a helper, adding a dependency) isn't a step. Merge it into the step it serves.
- **Areas** are modules or parts of the system, never file lists.
- No checkboxes, no status markers.
- `## Revision N` sections are only ever appended by a replan.

---

## Approval and handoff

- **User agrees to PLAN:**
  1. On main, stage exactly these two files: `git add features/<slug>/THESIS.md features/<slug>/PLAN.md`
  2. Commit: `plan(<slug>): thesis + plan`
  3. Tell the user: `/clear`, then `/execute <slug>`.
- **User aborts, at any point before that:** delete `features/<slug>/`. Nothing was staged, so nothing else needs undoing.
- Never stage or commit before the user agrees.

---

# Replan mode

Situation: /execute stopped on a problem the user confirmed. It has already stashed any unfinished work and tagged the failed attempt as `<slug>-attempt-N`.

Your job: fix THESIS and/or PLAN with the user, rewind the branch to the last step that's still valid, and hand back to /execute.

1. **Load context.** `git switch <slug>`. Read `learnings/LEARNINGS.md`; THESIS, PLAN and EXECUTION (including the problem entry) in `features/<slug>/`; and the relevant code. Open by engaging directly with the problem.
2. **Adjust THESIS and/or PLAN** with the user, using the same formats and rules as initial planning. If THESIS changes, get the user's agreement on it before touching PLAN.
3. **Compute the cut.** The cut is the first step that must be redone; every step before it is kept.
   - How to tell: the first step whose header, DoD or Verify changed, or the position where a step was inserted or removed.
   - A kept step whose EXECUTION entry assumed something that changed counts as changed. Flag it.
   - A step whose only change is wider Areas (covering work already done, header and DoD unchanged) doesn't move the cut.
   - A THESIS change that ripples through all of PLAN changes Step 1, so nothing is kept.
4. **Propose the cut:** "Keep Steps 1–k, redo from Step k+1." The user confirms it or moves it earlier.
5. **Save** the adjusted THESIS.md and PLAN.md outside the repo (e.g. in a `mktemp -d` folder). The reset in the next step would wipe them.
6. **Rewind the branch.**
   - Find the cut commit: step k's commit, message `<slug>: step k — …` (list with `git log main..<slug> --format='%h %s'`). If nothing is kept, use `git merge-base main <slug>`.
   - Show the user the exact command, then run `git reset --hard <cut>`.
7. **Restore** the adjusted files into `features/<slug>/`, and append a revision note to PLAN.md. N matches the attempt tag's number.

   ```markdown
   ## Revision N
   - **Problem:** Step 4's DoD can't be met: {what was found}
   - **Changed:** Step 4's DoD now {…}; new Step 5 {…}
   - **Cut:** Kept Steps 1–3, redoing from Step 4.
   - **Attempt:** <slug>-attempt-N
   ```

8. **Carry REVIEW.md forward** if `features/<slug>/REVIEW.md` exists at the attempt tag, so earlier review rounds stay on record: `git checkout <slug>-attempt-N -- features/<slug>/REVIEW.md`
9. **Commit** THESIS.md and PLAN.md (and REVIEW.md if restored): `<slug>: replan — revision N`. Replans commit to the branch, never to main.
10. **Hand off.** Tell the user: `/clear`, then `/execute <slug>`. /execute resumes from the first step that has no EXECUTION entry.