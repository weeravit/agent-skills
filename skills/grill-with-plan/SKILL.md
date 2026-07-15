---
name: grill-with-plan
description: Orchestrator that chains grilling, ponytail, and superpowers:writing-plans — grills a plan in Thai, pressure-tests it against over-engineering, then writes the confirmed design to an English plan doc in-project.
disable-model-invocation: true
---

Grill the user about the design below, then write the plan.

## Design

$ARGUMENTS

## Steps

1. Explore the codebase with `codegraph` before and during the session — never grep/read raw when a codegraph query answers it.
2. Run the `grilling` skill on the design above. Conduct the entire session in Thai. Whenever a recommendation risks over-engineering, invoke `ponytail` to pressure-test it back to the simplest working shape.
3. When the session feels complete, ask the user (in Thai) whether they want to grill further or add questions. Loop step 2 until they explicitly confirm they're done — never advance on an assumed yes.
4. Only after that confirmation, run `superpowers:writing-plans` to write the plan. English, into the project's own docs location (not the harness scratch/plan path). Exclude tests from the plan. Do not run `dart format` on it. Do not perform any translation work.
