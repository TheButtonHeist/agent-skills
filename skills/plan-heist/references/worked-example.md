# Plan a heist in BH Demo

This example uses the **Todo List** screen in the Button Heist demo app.
Its behavior comes from the [TodoListView implementation](https://github.com/TheButtonHeist/TheButtonHeist/blob/7a35532abc95001f84ab46e4931d13e180d559a2/TestApp/Sources/TodoListView.swift).
Labels, grouping, or behavior can change in another version.

Before a run, read the live interface.

## Establish the starting conditions

Start at the home screen of the demo with its default settings.
Make sure that Show Completed Todos is enabled.
Open Todo List.
Leave Filter set to All.

The screen resets its sample tasks each time it opens.
Two Work tasks start active:

- `Review PR, High priority`
- `Fix bug, High priority`

The app also contains other Review PR and Fix bug tasks with different priorities.
Each task row combines its children into one accessibility element.
The label contains the title and priority.
The value is `Active` or `Completed`.
Custom actions include `Toggle`, `Edit`, and `Delete`.
Completed rows also carry the selected trait.

The expected behavior is a change from active to completed and back.
The value and selected trait change together.

Keep the same named task as the target across both changes.

## Read and choose a target

Use `get_interface` to inspect Todo List.
Make sure that the row labels, values, traits, and custom actions match the expected interface.

The reusable plan starts on Todo List.
It evaluates row behavior independently of navigation.
The verification note records the separate navigation result.

Use `.label("Review PR, High priority")` for the first row.

The complete label distinguishes this row from the medium-priority Review PR tasks.
The target needs no ordinal or additional traits.
The combined row exposes Toggle directly.
The small visual toggle button is not necessary as a separate target.

Keep `Active` out of the target.
This value is the state that the action must change.

## Predict and try one interaction

The expected value changes from Active to Completed.
The row also gains the selected trait.

Send this action through `perform(step:)`:

```swift
CustomAction("Toggle", on: .label("Review PR, High priority"))
    .expect(.elementsChanged([
        .updated(
            .label("Review PR, High priority"),
            .value(before: "Active", after: "Completed")
        ),
        .updated(
            .label("Review PR, High priority"),
            .traits(
                before: .init(exclude: [.selected]),
                after: .init(include: [.selected])
            )
        )
    ]))
```

Inspect the evidence.
Make sure that observed updates satisfy both assertions.

A Completed row in the final state does not prove the transition or selected-state change.

If the value changes without the trait, investigate the failure.
Do not weaken the expectation.

## Preserve the behavior and pass values

The same behavior applies to both named tasks.

Define the behavior once.
Pass the row label into the targets and expectations.
Use a loop to supply the two labels:

```swift
HeistPlan("todoToggleRoundTrip") {
    HeistDef<String>("Todo.toggleRoundTrip", parameter: "task") { task in
        WaitFor(.exists(.element(.label(task), .value("Active"))))

        CustomAction("Toggle", on: .label(task))
            .expect(.elementsChanged([
                .updated(.label(task), .value(before: "Active", after: "Completed")),
                .updated(.label(task), .traits(
                    before: .init(exclude: [.selected]),
                    after: .init(include: [.selected])
                ))
            ]))

        CustomAction("Toggle", on: .label(task))
            .expect(.elementsChanged([
                .updated(.label(task), .value(before: "Completed", after: "Active")),
                .updated(.label(task), .traits(
                    before: .init(include: [.selected]),
                    after: .init(exclude: [.selected])
                ))
            ]))
    }

    ForEach("Review PR, High priority", "Fix bug, High priority") { task in
        RunHeist("Todo.toggleRoundTrip", task)
    }
}
```

The initial state predicate establishes a prerequisite.
Each action expectation proves two consequences.
The finite-value loop preserves the shared behavior and its inputs.
The runtime evaluates expectations on every iteration.
Successful completion leaves both tasks active again.
The program does not depend on the number or order of unrelated tasks.

The plan requires the All filter and the Show Completed Todos setting.
Otherwise, a completed row disappears before the second toggle.

Exercise that behavior in a separate journey with disappearance expectations.

## Validate and replay

Call `validate_heist` with the complete source and `lint: "strict_test"`.
Inspect the plan, invocation, and lint results.
Return to the home screen.
Open Todo List again to reset the exploratory change.
Run the complete plan with `run_heist`.
Make sure that both tasks end active without the selected trait.

If a run fails or stops, reset the screen before the next run.
Do not assume that the restoring toggle ran.
Deliver the source, starting conditions, validation result, and actual replay outcome.

Language validation alone does not establish a live pass.

If the user requests a `.heist` package, generate it with supported tooling.
Retain the editable source.

## Verification note

The row plan passed live replay on iOS 18.5 with Button Heist version
`0.7.1-0b8b2548b79e`.
The run passed four toggle actions, eight property assertions, and two prerequisite waits.

In the same build, navigation reached Todo List.
Exact-name and prefix screen-transition expectations timed out.
This result does not establish a screen-transition pass.
The navigation failure is separate from the row plan.
