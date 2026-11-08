---
name: plan-heist
description: Plan, author, and refine reusable Button Heist journeys from an app interface or described behavior. Use semantic targets, precise expectations, parameters, and bounded control flow.
---

# Plan a heist

A heist is a reusable accessibility journey that proves intended behavior.
The Button Heist DSL supports actions, branches, parameters, definitions, and
bounded loops.

For syntax, expectations, parameters, and control flow, read
[the DSL reference](references/dsl.md). For a complete example, read
[the BH Demo walkthrough](references/worked-example.md).

## Define the expected behavior

Identify the behavior to prove. Define the starting screen and required data.
Define the successful end state.

Keep the journey within the user's goal.
For a screen walkthrough, identify meaningful interactions and their consequences.
Choose a coherent journey. Do not activate every control indiscriminately.

Separate intended behavior from observed behavior. Use the user's requirements
and known product behavior to choose expectations.

A successful recording shows
what the app did. It does not establish that the behavior is correct.

Report uncertainties that affect the expected behavior.

## Read, predict, act, inspect

If an app is connected, use `get_interface` to inspect its accessibility interface.
The interface includes labels, values, traits, groups, and available actions.
If visual context helps, use `get_screen`.

If only a description or saved evidence is available, author from that evidence.
Identify the behavior that still requires live replay.

Before each meaningful action, define the expected changes.
Express these changes in an expectation.

For one action or standalone wait, use `perform(step:)`.
For a multi-step or composed program, use `run_heist(plan:)`.
Inspect the returned evidence before the next action.
Read the action delta first.

The delta summarizes the observed changes.

If the delta leaves a question unanswered, request another interface.

Retain the intended journey in the source. Observation calls, failed attempts,
and exploratory detours do not automatically belong in the saved heist.

## Choose concise semantic targets

Use the shortest target that identifies the intended control unambiguously.
Start with a meaningful label. If the label is ambiguous, add a trait or
container scope. Prefer natural accessibility properties.
If a returned canonical target identifies the intended control, reuse that target.

Do not copy every observed property into a target. Mutable state usually belongs
in the expectation. This separation lets the target identify the control across
its change. If order is part of the intended interaction, use an ordinal.
Capture-local IDs, geometry, and generated container names are not durable identity.
If accessibility information is missing, report the omission.

## Expect behavior across the interaction

Expectations can prove both current state and observed changes.
Choose the claim that expresses the expected behavior:

- Use `exists` or `missing` to establish a current condition.
- Use `appeared`, `disappeared`, or `updated` inside `elementsChanged` to prove
  observed transitions. For updates, name the property and relevant before and
  after values.
- Use `screenChanged` to prove navigation. If the destination name is known,
  include that name.
- If delivered accessibility feedback is part of the expected behavior, use
  `notification`.

One action can have several consequences. A quantity change can update both the
quantity and the subtotal. Put related element assertions in one
`elementsChanged([...])` expectation.

An action has one `.expect(...)`. For an additional condition outside that
predicate form, use `WaitFor(...)`.

Choose the most precise evidence appropriate to the behavior. Generic
`elementsChanged` or an unnamed screen change provides less evidence than a
specific consequence. Current-state evidence is appropriate if the transition
itself is irrelevant. It is also appropriate if reflow can replace the interface.
Do not assert incidental content. Do not weaken an expectation merely to pass.

When an action has no additional durable outcome, use
`.withoutExpectation("reason")` to explain the omission.
Do not use this waiver to omit a known consequence.

## Preserve reusable behavior

Use `HeistDef` and `RunHeist` to name reusable behavior.
If only the input changes, pass a string or semantic target.
Use that input in both targets and expectations where relevant.

- Use finite-value `ForEach` for supplied data.
- Use element `ForEach` for a matched collection.
- Use `RepeatUntil` for repeated action toward an outcome.

Keep expectations inside the repeated behavior.
Use `If` for legitimate alternate states.
Do not use a branch to skip a missing requirement.
If the app can finish without another action, use `WaitFor`.
Let the runtime reveal targets and settle the interface.
Avoid manual scroll setup and arbitrary delays.

Canonical source is the Swift-shaped DSL that Button Heist accepts.
Its explicit loops, definitions, and references persist in the durable plan.
The language does not execute arbitrary Swift.
Keep collection sizes, limits, and deadlines appropriate to the journey.

## Validate and deliver

Validate newly authored source with `validate_heist` and `lint: "strict_test"`.
Supply the same root argument intended for execution.
Inspect the plan, invocation, and lint results.

Plan validation establishes that the language and runtime structure are valid.
Invocation validation establishes that the supplied argument matches the root parameter.
Lint identifies problems in the authored test.
These results do not establish live target resolution or correct app behavior.

When live execution is in scope and available, restore the stated starting conditions.
Then replay the complete plan.
Until you identify the cause, retain the failure evidence.
Distinguish app defects from errors in the expectation or target.
Preserve the canonical DSL source for editing and regeneration.

If the user requests `.heist` packages, generate them with supported tooling.
Never write their internal JSON by hand.
Never rename a JSON file to `.heist`.

Deliver the source or requested artifact. Include its starting conditions and
inputs. State what the heist proves. Report the validation and replay outcomes.
State any unverified behavior or required external setup, including system UI
outside the app.
