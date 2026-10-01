# Claude Code Harness

A plan, build, review workflow for Claude Code. Each feature is agreed up front, built step by step with proof, independently reviewed, and handed to you to merge.

## THESIS → PLAN → EXECUTION → REVIEW → SUMMARY
## + LEARNINGS → REFLECTION

## Workflow

| Step | Command | What happens |
| --- | --- | --- |
| 1 | `/planning <intent>` | Two-stage discussion produces `features/<slug>/THESIS.md` (goal, risks, decisions) and `PLAN.md` (steps with DoD + Verify lines). Committed to `main` once you agree. |
| 2 | `/clear`
| 3 | `/execute <slug>` | Builds each step on branch `<slug>`, writes tests from the Verify lines, logs to `EXECUTION.md`, then runs review rounds, fixes, and wrap-up. Re-run to resume. |
| 4 | You | Check `(you)` DoDs, read `SUMMARY.md` and the diff, then merge yourself. |
| 5 | `/reflection` | Optional, after merging: fold learning candidates into `learnings/`. |

If `/execute` hits a confirmed problem with the plan, run `/planning <slug>` again to replan, then resume.

## Layout

```
.claude/
  skills/planning/   # /planning
  skills/execute/    # /execute
  agents/            # reviewer, summarizer, extractor subagents
  settings.json      # permissions
CLAUDE.md            # project rules and check commands
features/<slug>/     # THESIS, PLAN, EXECUTION, REVIEW, SUMMARY
learnings/           # candidates and LEARNINGS.md
```

## Safety

Claude Code works in branches.

`git push` and `git merge` are denied; `git reset`, `clean`, `rebase` and `branch -D` ask first. Merging is always yours.

## Status

5/7 tested. `/reflection` is referenced but not yet included.
