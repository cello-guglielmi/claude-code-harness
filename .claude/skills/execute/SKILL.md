---
name: execute
description: Harness step 2. Builds a planned feature (features/<slug>/PLAN.md) step by step on branch <slug>, proving each step against its DoD and logging it to features/<slug>/EXECUTION.md, then runs the review loop, summarizer and extractor. Resumable by running the same command again. Only run when the user invokes it.
disable-model-invocation: true
argument-hint: <slug>
---

# /execute

Slug: `$ARGUMENTS`. Feature folder: `features/$ARGUMENTS/`. Below, `<slug>` means `$ARGUMENTS`.

These instructions govern the whole session. If context gets compacted and anything here seems missing, re-read `${CLAUDE_SKILL_DIR}/SKILL.md` before continuing.

**The whole run, in order:** build every PLAN step (§2) → review rounds (§4), with fixes between them (§5) → wrap-up and handoff (§6). §1 decides where in that sequence to start.

---

## Standing rules

These hold for the whole run.

**Logging**
- **Nothing is silent.** Everything you do lands in `features/<slug>/EXECUTION.md`: what was built and every detail you filled in where PLAN is unspecified (Did), snags (Notes), and departures from a step (Deviation).
- **EXECUTION.md is append-only.** An entry is appended only as part of the commit it belongs to.

**Scope**
- **PLAN dictates** what each step does (its header), where it may touch (its Areas), and what counts as proof (its DoD and Verify lines). It never dictates how. DoDs never change during execution.
- **Off-limits, always:** production, real databases, payments, messages to real people, `git push`, deploys, merging, and anything CLAUDE.md adds.

**Environment**
- **Checks:** CLAUDE.md lists the check commands, including tests. If it names no test command, use the language's built-in runner (Python: `unittest`, tests in `tests/`, run with `python -m unittest`).
- **Trunk:** `main`, unless CLAUDE.md names another branch. Wherever this file says "main", read it as the trunk.

**Working with the user**
- **Prompting the user** means: say what you found and what you intend, then end your turn and wait. Record their words faithfully.
- Keep focus with your todo list; it isn't a harness file.
- Don't read `learnings/`.

**Commit messages**

§1 (resume) and /planning's replan both read these. Keep them exact.

| Commit | Message |
| --- | --- |
| Step | `<slug>: step N — <step header text>` |
| Problem entry | `<slug>: problem — <title>` |
| Review round | `<slug>: review round N — PASS` or `— CHANGES` |
| Fix entry | `<slug>: fix RN.M` |
| Escalation entry | `<slug>: escalation after round N` |
| Wrap-up | `<slug>: summary + learning candidates` |

The replan commit, `<slug>: replan — revision N`, is written by /planning, never by you.

---

## 1. Start and resume

Purpose: get onto the feature branch and work out where the run picks up. Running /execute again always lands here, so this is also how a paused run resumes.

1. **Find the plan.** Look for `features/<slug>/THESIS.md` and `features/<slug>/PLAN.md` on branch `<slug>` if it exists, else on main. Missing → say so and stop.
2. **Get onto the branch.** One of two cases:
   - **Fresh start** (branch `<slug>` doesn't exist):
     1. The tree must be clean. If it isn't, show `git status --short` and ask.
     2. `git switch -c <slug> main`.
     3. Begin Step 1 (§2).
   - **Resume** (branch `<slug>` exists):
     1. `git switch <slug>`.
     2. If there are uncommitted changes, discard them with `git stash push -u -m "<slug>: discarded on resume"` and tell the user they're in the stash.
     3. Remove any leftover review worktree (§4, step 2).
     4. Read THESIS, PLAN and EXECUTION in `features/<slug>/`.
     5. Route (step 3 below).
3. **Route** (resume only). Check these cases in order; the first match decides.
   - **Replan pending:** you earlier stopped on a problem the user confirmed, and /planning hasn't replanned yet.
     - How to tell: the last entry in `features/<slug>/EXECUTION.md` is a `## Problem:` entry marked `**Ruling:** Confirmed.`
     - Action: say so, give the user `/clear`, then `/planning <slug>`, and stop.
   - **Steps remaining:** not every PLAN step has been built.
     - How to tell: some PLAN step has no `## Step N:` entry in EXECUTION.md.
     - Action: start at the first such step (§2). A half-done step is redone from scratch.
   - **Wrapped up:** only the handoff is left.
     - How to tell: `git log main..<slug> --format=%s` shows the wrap-up commit (`<slug>: summary + learning candidates`).
     - Action: handoff only (§6, step 3).
   - **Review passed:**
     - How to tell: the latest review-round commit ends in `— PASS`.
     - Action: wrap-up (§6).
   - **Fixes pending:**
     - How to tell: the latest review-round commit ends in `— CHANGES`, and some of that round's findings in `features/<slug>/REVIEW.md` have no matching `## Fix RN.M` entry in EXECUTION.md.
     - Action: fixes (§5) for those findings.
   - **Ready for review:** anything else (all steps built, no round yet or every finding of the latest round has a fix entry).
     - Action: cap check (§5), then a new review round (§4).

---

## 2. Building a step

Purpose: build each PLAN step in order, prove it with tests that implement its Verify lines, and commit it together with its log entry.

For each step, in order:

1. 1. **Announce gap-fills.** Before building, list the behavior choices this step needs that PLAN doesn't settle: anything a user of the software would notice. The test: would it be a DoD if PLAN had thought of it? Say what you intend for each, then wait for the user. 
   - If one turns up mid-step: stop and announce it the same way.
   - Internal choices (structure, naming, helpers) aren't announced; they go in Did as usual.
2. **Do the step,** within its Areas. If it needs an area outside its Areas, or something other than what its header says → Deviation (§3) **before** doing it.
3. **Write tests** implementing each of the step's Verify lines as written. Tests take their expected values from the Verify line. On the final step, also write the test for Done when's Verify line. `(you)` lines get no test.
4. **Run** the existing checks plus every DoD test written so far, for all steps up to this one.
5. **Failing →** fix within the step, up to 3 attempts. Record what it took in Notes.
6. **Still failing →** triage. State which of these is broken:
   - what the step says, or where it may touch → Deviation (§3)
   - the DoD itself: it can't be met as written, by any approach → PLAN problem (§3)
7. **Log.** Append the step entry to EXECUTION.md, written from what actually happened.
8. **Commit.** `git add -A`, then commit code, tests and entry together: `<slug>: step N — <step header text>`.

You choose how to test, never what counts as proof: the proof is the Verify line. Tests are ordinary project code and persist.

### Step entry format

This is the only definition.

```markdown
## Step 3: Make the timeout value configurable
- **Did:** Timeout read from the SESSION_TIMEOUT env var, defaulting to 30 minutes; chose an env var over a config file so it can differ per deployment. Registered SESSION_TIMEOUT in the config loader.
- **Tests:** Wrote tests/test_session_timeout.py for Step 3's Verify line. Ran existing checks and DoD tests for Steps 1–3: all pass.
- **Notes:** First attempt read SESSION_TIMEOUT directly in the session store; the loader drops unregistered env vars at startup, so it was always unset.
- **Deviation:**
  - **Step:** Areas: session store.
  - **Instead:** Also touched the config loader, to register SESSION_TIMEOUT.
  - **Why:** Env vars only reach the app through the loader; the session store can't read one on its own.
  - **Your input:** Agreed; register only this variable, no loader refactor.
```

- The header copies the step header from PLAN exactly.
- Every field is always present. Empty ones say `None.`
- **Did:** what was built, including every detail filled in where PLAN is unspecified.
- **Tests:** tests written for the Verify lines, what ran, the result.
- **Notes:** snags and surprises, including DoD tests that took several attempts.
- **Deviation:** `None.` or one sub-block per Deviation (§3).

---

## 3. Deviations and problems

Purpose: when a step can't go exactly as written, classify what's wrong. Each kind is handled differently.

**The four cases:**
- **Not a Deviation:** how you reach a step's DoD. That's your call; record it in Did.
- **Deviation:** departing from what a step states (touching an area outside its Areas, or doing something other than its header says) while every DoD still holds. → Ask the user, then continue.
- **PLAN problem:** a DoD can't be met as written, by any approach. → Stop.
- **THESIS problem:** something works against the Goal or breaks a Decision. → Stop.

**Edge rules:**
- When unsure whether something falls outside the header or Areas, prompt the user.
- Work no DoD needs is not a Deviation. Don't do it.
- A change that would need a DoD change is never a Deviation; it's a PLAN problem.
- A Risk from THESIS that materializes is never a THESIS problem.

### Handling a Deviation

1. Stop. Tell the user what you found and what you intend to do instead. Wait.
2. The user's input shapes the change; it isn't a yes/no. If their input is that the DoD itself can't be met, it's a PLAN problem instead.
3. Proceed as agreed, and log a Deviation sub-block in the entry of the step (or fix) during which it came up:

```markdown
- **Deviation:**
  - **Step:** What the step states that couldn't hold: its header or its areas.
  - **Instead:** What was done.
  - **Why:** What made the step impossible as written.
  - **Your input:** What the user said, or `Agreed as proposed.`
```

### Handling a problem (PLAN or THESIS)

1. Stop. Tell the user what you found. Wait for them to confirm it, or disagree with a reason.
2. Append the problem entry to EXECUTION.md and commit **only** that file (`git add features/<slug>/EXECUTION.md`): `<slug>: problem — <title>`. In-progress step work stays uncommitted.
3. Then, by the user's ruling:
   - **Disagreed:** continue where you stopped.
   - **Confirmed:**
     1. Set N = the number of existing `<slug>-attempt-*` tags + 1.
     2. Stash in-progress work: `git stash push -u -m "<slug>-attempt-N wip"`.
     3. Tag the branch head `<slug>-attempt-N`.
     4. Give the user `/clear`, then `/planning <slug>`, and stop.

### Problem entry format

This is the only definition.

```markdown
## Problem: API clients get a new mid-session 401
- **Kind:** THESIS
- **Conflicts with:** Decision: "Don't change the public API."
- **Found:** A server-side timeout makes API clients receive a new 401 mid-session, which they'd have to handle.
- **Why the approach can't fix it:** Any idle expiry becomes visible to API clients; only exempting API sessions avoids it, which defeats the Goal.
- **Ruling:** Confirmed.
```

- **Kind** is `THESIS` or `PLAN`. For a PLAN problem, Conflicts with names the step's DoD.
- **Ruling** is `Confirmed.` or `Disagreed:` followed by the user's reason.

---

## 4. Review round

Purpose: have an independent reviewer check all the work so far, from a frozen snapshot of the branch.

Runs once every step has an entry, and again after each completed set of fixes.

1. Make sure `.worktrees/` is listed in `.git/info/exclude` (append it if not).
2. Remove any leftover worktree: `git worktree remove --force .worktrees/review-<slug> 2>/dev/null; git worktree prune`.
3. Create the snapshot:
   ```bash
   EXECUTION_HEAD=$(git rev-parse <slug>)
   git worktree add --detach .worktrees/review-<slug> "$EXECUTION_HEAD"
   ```
4. Spawn the `reviewer` subagent with the prompt `slug: <slug>` and **nothing else**. Adding anything would pass on your own reasoning and break the reviewer's independence.
5. When it returns, remove the worktree (step 2's command).
6. Confirm `features/<slug>/REVIEW.md` gained a new `## Round` (count `^## Round ` lines). If not, tell the user and stop.
7. Commit only `features/<slug>/REVIEW.md`: `<slug>: review round N — <verdict>`. Commit before acting on findings.
8. By verdict:
   - **PASS** → wrap-up (§6).
   - **CHANGES** → fixes (§5).

---

## 5. Fixes, cap and escalation

### Fixes

Purpose: resolve every finding from the latest CHANGES round.

For each finding in the latest round, in order, match one case:

- **Fixable with DoDs unchanged:** fix it, append a fix entry, and commit code + entry: `<slug>: fix RN.M`.
  - Undo any unexplained or out-of-scope change. If a DoD genuinely needed it, raise it as a Deviation (§3) instead; the fix entry logs it with the user's input.
  - A hollow test gets strengthened until it catches the break the finding names.
- **You believe the finding doesn't hold:** tell the user why and wait. They rule: fix or dismiss. Never dismiss on your own.
- **Fixing it needs a DoD or THESIS change:** it's a problem (§3).

Once every finding has a fix entry, run the existing checks and all DoD tests, then do the cap check and start a new round (§4).

### Fix entry format

This is the only definition.

```markdown
## Fix R1.2: Hollow DoD test
- **Reference:** Step 3: Make the timeout value configurable
- **Status:** Completed
- **Did:** Test now unsets SESSION_TIMEOUT and asserts expiry at the 30-minute default.
- **Tests:** Ran existing checks and all DoD tests: all pass.
- **Notes:** None.
- **Deviation:** None.
```

- The header carries the finding ID and its title.
- **Reference** copies the finding's Reference. Leave out the finding's Issue, so the next round doesn't anchor on it.
- **Status** is `Completed` or `Dismissed`. A dismissed finding has `Did: None.` and the user's reason in Notes.

### Cap check

Purpose: stop an endless review loop by handing control to the user after 3 rounds without a PASS.

Before each new round:

1. Count the review-round commits since the most recent of: the branch start, the last replan commit, the last escalation commit. Rounds before a replan stay in REVIEW.md but don't count.
2. Fewer than 3 → start the round (§4).
3. 3 → stop, show the user the latest round's findings, and wait. Record their decision in an escalation entry, committed on its own: `<slug>: escalation after round N`. Then follow the decision.

```markdown
## Escalation: 3 rounds without PASS
- **Open findings:** R3.1, R3.2
- **Ruling:** {what the user decided and why}
```

---

## 6. Wrap-up and handoff

Purpose: produce the summary and learning candidates, then hand the finished branch to the user to merge.

1. Spawn `summarizer` and `extractor` in parallel, each with the prompt `slug: <slug>` and nothing else.
2. Check that `features/<slug>/SUMMARY.md` and `learnings/<slug>.candidates.md` exist.
   - **Both exist:** commit both: `<slug>: summary + learning candidates`.
   - **One is missing:** These files are written by subagents. Check to see if it's not still being written. If not, alert the user to which one, and stop. 
3. Hand off. Show the "Before you merge" checklist from SUMMARY.md, and tell the user:
   - Check each `(you)` DoD yourself; decide on each "Not verified" claim.
   - Review `git diff main...<slug>` and SUMMARY.md.
   - Merge (yours to run; never run it yourself):
     ```bash
     git switch main
     git merge --no-ff <slug> -m "Merge <slug>"
     ```
   - After merging, `/reflection` whenever you like.