---
name: grill-with-plan
description: Orchestrator that chains grilling, ponytail, and superpowers:writing-plans — asks the user which language to grill in, pressure-tests the plan against over-engineering, then writes the confirmed design to an English plan doc in-project.
disable-model-invocation: true
---

Grill the user about the design below, then write the plan.

## Design

$ARGUMENTS

## Steps

1. Run `codegraph sync` first, always, before anything else.
2. Explore the codebase with `codegraph` before and during the session — never grep/read raw when a codegraph query answers it.
3. Ask the user which language to grill in before starting the session. Use that language for every step below until the plan is written.
4. Run the `grilling` skill on the design above, conducting the entire session in the chosen language. Invoke `ponytail` on any over-engineering risk.
5. When the session feels complete, ask the user (in the chosen language) whether they want to grill further or add questions. Loop step 4 until they explicitly confirm they're done — never advance on an assumed yes.
6. Only after that confirmation, run `superpowers:writing-plans` to write the plan:
   - Language: English.
   - Save path: `docs/superpowers/plans/` in the project (not the harness scratch/plan path).
   - Exclude tests from the plan.
   - Do not run `dart format` on it.
   - Do not perform any translation work.

## Missing skill fallback

`grilling`, `ponytail`, and `superpowers:writing-plans` are required for steps 4-6. If any of them does not resolve (not in the available-skills listing, under any namespace), run `npx skills find "<skill-name>"` to locate it — do this search on your own, without asking the user first. Present the matches found and ask the user which one to install before running any install command. Once installed, retry the step. Do not skip the step or improvise a substitute in place of the missing skill.
