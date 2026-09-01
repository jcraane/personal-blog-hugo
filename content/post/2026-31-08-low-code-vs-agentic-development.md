---
layout: post

title: "Agentic Development Is Eating Low-Code"
subtitle: "The friction low-code was invented to remove is the friction agents remove for free"
author: Jamie Craane
date: 2026-08-31
description: "Low-code traded boilerplate syntax for visual abstractions. Now that AI agents write the boilerplate, that trade no longer pays for itself. Here is where low-code breaks down, what changes with agents, and the narrow cases where low-code still wins."
image: "/img/posts/low-code-vs-agentic-development.jpg"
showtoc: false
tags:
- AI
- AI-Assisted Development
- Low-Code
- Architecture
- Kotlin

categories: [ AI ]
URL: "/2026/31/08/low-code-vs-agentic-development/"
---

### Introduction

Every couple of years I end up in the same conversation. A team needs an internal tool: a form, an approval flow, a few integrations. Someone proposes a low-code platform, because writing it "properly" means a repo, a build, a pipeline, and a sprint. The platform wins on time-to-first-demo, and it deserves to.

Then the tool succeeds. A year later it has forty screens, a branching approval rule nobody can read, and a change request that takes two days of clicking because there is no way to find every place a field is referenced.

The core promise of low-code was democratizing software creation by trading boilerplate syntax for visual abstractions. That trade made sense when writing the first thousand lines was genuinely expensive. Two questions are worth asking now that it isn't:

- What was low-code actually charging you for?
- Does that charge still buy anything once an agent writes the boilerplate?

### What Low-Code Was Actually Solving

Low-code never eliminated complexity. It moved it. The business rules, the state transitions, the error handling: all still there, just expressed in proprietary UI panels, clumsy JSON blobs, and rigid workflow engines instead of in a language with a type checker.

What it genuinely removed was the *startup cost* of a codebase. No project scaffolding, no dependency management, no deployment story, no CRUD layer written by hand. You got a running thing on day one. Everything else about low-code, the constraints, the lock-in, the untestability, was the price of that day one.

### Where Low-Code Breaks Down

The bill arrives later, and it arrives in four places.

- **The expressiveness ceiling.** Visual logic builders are great for linear triggers, A to B to C. Expressing complex business rules, edge-case branching, or asynchronous concurrency visually becomes an unmaintainable web of spaghetti boxes. The escape hatch is always the same: a text field where you write a real expression, in a dialect nobody documents.
- **No software lifecycle parity.** Testing is brittle or retrofitted. You rarely get true headless unit testing, mocking, or a deterministic CI pipeline. What you get is a staging tenant and a human clicking through it.
- **Environment drift and governance.** Migrating across dev, staging, and production relies on platform-specific export and import tools, or on opaque database state, rather than clean diffable commits. "What changed last Tuesday" is a question the platform often cannot answer.
- **Vendor lock-in and portability.** You do not own the runtime. If the platform alters pricing, deprecates a connector, or quietly changes the behavior of a node, you have no escape hatch. Your business logic is a row in someone else's database.

### The Same Rule, Twice

Here is a discount rule as a workflow engine exports it:

```json
{
  "nodes": [
    { "id": "n1", "type": "trigger.http", "path": "/orders" },
    { "id": "n2", "type": "condition",
      "expr": "{{ $node.n1.body.total }} > 500 && {{ $node.n1.body.customer.tier }} == 'gold'" },
    { "id": "n3", "type": "action.setField", "field": "discount", "value": "0.15" }
  ],
  "connections": { "n1": ["n2"], "n2": { "true": ["n3"] } }
}
```

That `expr` field is code. It has no type checker, no autocomplete, no reference to what `tier` may legally contain, and no test that fails when someone renames a field upstream. The `0.15` is a string. Reviewing a change to this file means reading node IDs.

The same rule as code:

```kotlin
fun discountFor(order: Order): Discount = when {
    order.customer.tier == Tier.GOLD && order.total > Money.eur(500) -> Discount.percent(15)
    order.customer.tier == Tier.GOLD -> Discount.percent(5)
    else -> Discount.NONE
}

@Test
fun `gold customers below the threshold get the base discount`() {
    val order = order(tier = Tier.GOLD, total = Money.eur(499))
    assertEquals(Discount.percent(5), discountFor(order))
}
```

The interesting part is not that the second version is nicer to look at. It is that in 2020 the second version cost meaningfully more to produce than the first, and today it costs less. An agent writes the data classes, the branch, and the test in one pass, and the compiler checks the result. The boilerplate that made low-code attractive is exactly the work agents are best at.

### What Changes With Agents

| Dimension | Low-code platforms | Agentic development |
| --- | --- | --- |
| Interface | Proprietary GUI, drag and drop | Natural language plus plain code in a standard IDE |
| Artifact | Platform-locked metadata and schemas | Portable source code, Git-native |
| Testing | Manual UI testing, bolted-on suites | Native unit, integration, and property-based tests |
| Extensibility | Constrained by platform plugins | Any library in the ecosystem |
| Refactoring | Manual clicks across dozens of screens | Automated diffs across the whole codebase |

Agentic workflows remove the primary friction low-code was invented to solve. Because agents produce standard, linted, testable code, you get something close to low-code prototyping speed without giving up version control, modular architecture, or a real test harness. Refactoring is the clearest win: renaming a concept across a codebase is a mechanical job an agent does in minutes, and the compiler tells you when it is wrong. There is no equivalent operation in a click-based builder.

### Where Low-Code Still Wins

Two constraints keep low-code alive, and neither is about developer productivity.

**Non-technical stakeholder editing.** When the requirement is that an operations manager changes the approval threshold on a Tuesday afternoon without involving engineering, a repo and a pipeline are the wrong shape. Note that the actual requirement here is a safe editing surface for a small number of parameters, not a general-purpose programming environment. Teams reach for a low-code platform and end up with a whole application in it, which is how the forty-screen situation happens. A configuration UI backed by your own code covers the real need without the rest.

**Compliance and platform governance.** Regulated environments often accept a vendor's certifications, audit logging, and access model as a package. Rebuilding that assurance around a custom codebase is real work, and "the platform is SOC 2 certified and our auditor already knows it" is a legitimate reason to stay. The same goes for organizations where a central IT function has standardized on one platform, and anything else needs an exception process longer than the project.

Both of these are organizational constraints, not technical limits of code. Worth being honest about that when someone argues low-code is the better *engineering* choice.

### A Few Caveats

**Agent-generated code is not free.** You still own it. A codebase produced quickly is still a codebase to review, run, patch, and keep dependencies current on. Low-code at least outsources the runtime; with your own code, that operational burden is yours. It buys real optionality, but it is a cost, not a bonus.

**"Standard, testable code" describes the good case.** Agents happily produce plausible code with no tests and quietly wrong edge cases. The advantage over low-code is that the failure is *inspectable*: a compiler, a linter, and a test suite can catch it. That advantage is only worth something if you actually run them.

**Existing low-code estates are not going anywhere.** Migrating a mature platform application means recovering business rules encoded in node graphs that nobody has read in three years. The argument here is about what to build next, not a case for rewriting what is running.

### Conclusion

Low-code was a rational answer to a real cost: writing the first version of everything by hand was slow. That cost has dropped sharply, and the compensations low-code demanded in return, the ceiling, the missing lifecycle, the lock-in, have not dropped at all.

So the calculation has flipped. For actual application development, standard code with an agent doing the heavy lifting now wins on the dimension low-code used to own, which is speed, while keeping everything low-code never had. What is left for low-code is narrower and mostly organizational: simple internal forms wired to a spreadsheet, operational dashboards owned by non-technical teams, and environments where a vendor's compliance posture is the deliverable.

If you are reaching for a low-code platform today, the question worth asking first is whether you are buying speed or buying an editing surface for people who do not write code. Only one of those is still a good deal.
