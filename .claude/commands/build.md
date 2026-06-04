---
description: Execute one step of the approved plan under TDD discipline. Reads `.claude/state.md` to find the active feature and the next step.
---

Drive the `engineer` skill through **exactly one** plan step. One `/build` invocation completes one step and then pauses for the user.

## Preconditions

Before doing anything:

1. Read `.claude/state.md`.
2. Verify all of the following — if any is false, **stop and report what is missing** (do not work around it):
   - `Active feature` is not `none`.
   - `Plan` points to an existing file.
   - The plan's front-matter contains `status: approved`.
   - `Next step` names a step in the plan.

### How state.md gets set up

The normal flow is `/feature-start` (sets `Active feature`, `Spec`, `Phase: spec-draft`) → user approves spec (flips spec front-matter `status` to `approved`) → `/plan` (sets `Plan`, `Phase: plan-draft`, `Next step: Step 1 — <name>`) → user approves plan (flips plan front-matter `status` to `approved`). At that point `/build` is runnable.

If `/feature-start` / `/plan` weren't used (hand-authored spec and plan), the user is responsible for state.md being populated the same way: `Active feature`, `Spec`, `Plan`, `Next step` all set, and both the spec and plan front-matter `status: approved`. Phase is descriptive — `/build` does not gate on it.

If state isn't set up that way, explain what's missing and stop.

## Run

Load the `engineer` skill. Execute its TDD loop (red → green → refactor → verify) against **only** the step named in `Next step`. The engineer skill governs the discipline; this command just scopes the work to one step.

Hand off to the `tester` subagent in fresh context for the red phase, exactly as the engineer skill describes. **Do not pass implementation source to the tester** — only the spec, the current step's test list, and type signatures of code under test.

## After the step

When the step is complete:

- Update `.claude/state.md`:
  - Record the step under a `## Completed steps` section (create the section on first use).
  - Set `Next step` to the next plan item, or `—` if this was the last step.
  - If the plan is finished, set `Phase: idle` and note that the plan is fully built. Slice 1 has no `/feature-merge` yet; the user closes out manually.
- Pause for the user. Silent unless there is something to address.

## Halt conditions

Stop and surface to the user (do not auto-recover) if:

- The `tester` returns findings instead of tests.
- Red tests pass on first run.
- Green tests regress after a refactor and the cause is not obvious.
- The linter or type-checker reports an issue that cannot be resolved within the current step.
- A precondition is violated mid-run.
