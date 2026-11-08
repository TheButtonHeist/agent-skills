# DSL reference for planning heists

Button Heist parses canonical source into a durable `HeistPlan`.
Canonical source is the Swift-shaped DSL that the runtime accepts.
The runtime does not send this source to `swiftc`.

For one action or standalone `WaitFor`, use `perform(step:)`.
For a complete plan, use `run_heist(plan:)`.

## Targets

Start with meaningful labels. Add only the detail that resolves an ambiguity:

```swift
.label("Checkout")
.element(.label("Checkout"), .traits([.button]))
.within(container: .label("Milk"), .label("Remove"))
```

If an existing app contract requires an identifier, use that identifier.
Do not add identifiers to compensate for missing accessible labels or actions.

## String matching

Strings match exactly by default, with case and typography folding.
Broader matches are explicit. The runtime does not broaden a match after an
exact miss.

| Form | Meaning | Example |
|---|---|---|
| Exact string | Matches the complete text | `.label("Review PR, High priority")` |
| `.prefix(...)` | Matches the start of the text | `.label(.prefix("Review PR,"))` |
| `.suffix(...)` | Matches the end of the text | `.label(.suffix("items remaining"))` |
| `.contains(...)` | Matches text within the complete string | `.label(.contains("High priority"))` |

If the complete text is stable, prefer an exact match.
If part of the text can vary legitimately, broaden only that part.
Each broader matcher requires a non-empty string.

A broad matcher can select several elements. For example, the demo has several
rows that start with `Review PR,`. If one element is required, resolve the
ambiguity with a more specific label or semantic scope.
For a deliberate collection operation, use an element loop.

String matchers apply to labels, values, and identifiers. Supported screen and
notification predicates also accept string matchers. Update expectations use
the same matchers for their before and after values.

For example, `.value(after: .prefix("Saved at "))` accepts a status that contains
a variable save time. The expected status must still start with `Saved at `.
If the required value is known, use that exact value.
A broad matcher must not hide an incorrect result.

## Expectations

If the behavior is an in-place value change, use an observed update:

```swift
Increment(.label("Quantity"))
    .expect(.elementsChanged([
        .updated(.label("Quantity"), .value(before: "2", after: "3")),
        .updated(.label("Subtotal"), .value(before: "$4.00", after: "$6.00"))
    ]))
```

Every assertion must hold. `updated` names its own target.
It does not inherit the action's target.
If only the destination update matters, omit `before:`:
`.updated(.label("Quantity"), .value("3"))`.

Updates can also match changes in traits, hints, actions, custom content, and rotors.
For their property matchers, read the installed language documentation.

If the expected behavior includes an appearance or disappearance, use lifecycle evidence:

```swift
Activate(.label("Add to Cart"))
    .expect(.elementsChanged([
        .appeared(.label("Added to cart")),
        .updated(.label("Cart count"), .value(before: "0", after: "1"))
    ]))
```

`appeared`, `disappeared`, and `updated` require observed events.
A matching final state alone cannot satisfy these predicates.
`updated` requires evidence from the same screen generation.
Screen replacement produces departure and arrival evidence instead.

Navigation and feedback have their own predicates:

```swift
Activate(.label("Checkout"))
    .expect(.screenChanged("Checkout"))

Activate(.label("Save"))
    .expect(.notification("Saved"))
```

Use the screen name from observed screen evidence.
Do not assume that a visible heading is the semantic screen name.
`screenChanged` proves a screen transition.
`exists` establishes a current condition.
Screen predicates accept an optional name matcher, not an array of element assertions.
For additional destination content, use `WaitFor`.

If the exact transition is irrelevant, use state predicates for prerequisites or settled results:

```swift
WaitFor(.exists(.label("Cart")))

TypeText("milk", into: .label("Search"))
    .expect(.exists(.element(.label("Search"), .value("milk"))))

WaitFor(.missing(.label("Loading")), timeout: 10)
```

`missing` does not prove that an element previously existed.
For a text field that can reflow, assert its settled value.
An update expectation requires evidence that the same element changed.

Each action or invocation has one `.expect(...)`.
A subsequent `WaitFor` can evaluate current state.
It cannot recover transition evidence from a completed action.
Attach required transition evidence to the action that must produce it.

## Definitions and passed values

Define related behavior once. Supply the data through a parameter:

```swift
HeistPlan("browseCategories") {
    HeistDef<String>("Catalog.openCategory", parameter: "category") { category in
        Activate(.label(category))
            .expect(.screenChanged(category))

        Activate(.label("Back"))
            .expect(.screenChanged("Catalog"))
    }

    ForEach("Books", "Music", "Games") { category in
        RunHeist("Catalog.openCategory", category)
    }
}
```

This example assumes that category labels are also the observed destination
screen names. Every category provides Back.
The loop preserves the behavior and its expectations for each iteration.

If a behavior operates on a control that the caller selects, use a target parameter:

```swift
HeistPlan("removeMilk") {
    HeistDef<AccessibilityTarget>("Cart.remove", parameter: "item") { item in
        CustomAction("Remove", on: item)
            .expect(.elementsChanged([.disappeared(item)]))
    }

    RunHeist("Cart.remove", .label("Milk"))
}
```

This example requires a Milk element that exposes the custom action Remove.
Pass semantic targets. Do not retain live element objects as arguments.

Root plan names are single Swift-style identifiers.
Dotted paths such as `Cart.remove` name definitions and invocations.
Definitions must precede executable steps in their block.
Parameterless definitions use `HeistDef<Void>`.

A root plan can also take a value from the caller:

```swift
HeistPlan("openCategory", parameter: "category") { category in
    Activate(.label(category))
        .expect(.screenChanged(category))
}
```

For this root, pass `argument: {"type":"string","value":"Books"}` to both
`validate_heist` and `run_heist`.
A root target parameter uses `targetParameter: "target"`.
Its caller supplies an `accessibilityTarget` argument in the tool's advertised schema.
A reusable definition accepts one declared argument.
It does not accept arbitrary Swift objects or a variadic parameter list.

## Loops and branches

For a matched collection, use element iteration:

```swift
ForEach(.label("Dismiss"), limit: 20) { target in
    Activate(target)
        .expect(.elementsChanged([.disappeared(target)]))
}
```

This example assumes that each matched control disappears after activation.
Every action resolves its target live.
The initial match count and explicit limit bound execution.

After each iteration, the runtime examines the collection again.
If identity or order changes, the next iteration uses ordinal zero.
State changes alone do not restart the ordinal sequence.
The loop does not consume newly arriving controls indefinitely.
Collection loops cannot nest, including through invoked definitions.

If another action is required, use repetition toward an outcome:

```swift
If(.missing(.label("Inbox empty"))) {
    RepeatUntil(.exists(.label("Inbox empty")), timeout: 10) {
        Activate(.label("Delete next"))
            .expect(.elementsChanged([
                .updated(.label("Message count"), .value())
            ]))
    }
}
```

This example assumes one Delete next control.
Each deletion updates Message count in place, including the last deletion.
The unspecified value matcher establishes that the value changed.
If the required values are known, use concrete before and after values.

`RepeatUntil` executes its body at least once.
If the inbox can already be empty, keep the guard.
The mandatory timeout bounds the whole loop.
A timeout fails the step.
If another action is unnecessary, use `WaitFor`.

Branch on supported alternate states.
For a required condition that the app cannot fulfill, use `Fail("reason")`.
For an intentionally nonfatal finding, use `Warn("reason")`.
`WaitFor(...).else { ... }` can express timeout recovery.
Do not use a branch to bypass a regression.

## Runtime and artifact boundaries

The language supports composition and conditional execution.
Finite data, limits, or deadlines bound repetition.
Canonical runtime source excludes native Swift `if`, `for`, and `while`.
It also excludes imports, arbitrary functions, interpolation, and unrestricted computation.
In trusted local Swift authoring, host computation can flatten into fixed steps
before a plan exists.

Normal semantic actions own reveal and settlement.
Viewport/debug/session commands are outside the durable semantic journey.
Spatial gestures are explicit exceptions.
Strict test lint flags these gestures.
Timeouts are positive, finite seconds within the configured maximum.
Use outcome predicates instead of sleeps.

A generated `.heist` is a directory that contains `manifest.json` and `plan.json`.
The `plan.json` file contains generated storage IR.

Author canonical source.
Preserve the source.
Regenerate artifacts with supported tooling.
Do not edit the IR.
Record required app setup separately.

An artifact does not select the app.

For version-specific details, consult the installed tool schemas and the
project's [MCP agent guide](https://github.com/TheButtonHeist/TheButtonHeist/blob/main/docs/MCP-AGENT-GUIDE.md)
and [language specification](https://github.com/TheButtonHeist/TheButtonHeist/blob/main/docs/HEIST-LANGUAGE-SPEC.md).
