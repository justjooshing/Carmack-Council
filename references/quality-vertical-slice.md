# Vertical Slice Quality Reference — Carmack × Cockburn

Philosophy: John Carmack. Delivery methodology: Alistair Cockburn (Crystal, use cases), Mike Cohn (user stories, INVEST criteria), Kent Beck (walking skeleton, XP).
Stack context: TanStack Start / TanStack Query / React / TypeScript / Drizzle / Postgres / Redis / Shopify.

Every finding must describe the **concrete delivery failure** — not just "this is wrong methodology."
This doc is about PLAN STRUCTURE and TASK DECOMPOSITION, not code correctness. Code quality is in quality-backend.md, quality-frontend.md, etc. This doc covers one question: is this plan structured so that working software can be shipped at each task boundary?

---

## Principle 1: A vertical slice is the unit of shippable value — never plan by layer

*Carmack: "You should be able to ship a vertical slice of your game — not all the menus, not all the enemies, but one complete level that demonstrates the whole experience." Applied to software: each task should produce a working, testable end-to-end capability.*
*Cockburn: "A vertical slice cuts through all layers — UI, API, database — to deliver a thin but complete piece of functionality."*

A plan built by horizontal layers looks like:
1. Define all Drizzle schema
2. Build all server functions
3. Build all TanStack Query hooks
4. Build all UI components
5. Wire it together and test

Nothing works until Step 5. Integration failures are discovered at the end when the entire feature is at risk. The developer has no signal that the architecture is sound until everything is built.

A plan built by vertical slices looks like:
1. User can view an empty list (schema + server function + query hook + UI — all layers, just the read path)
2. User can create one item (add mutation — all layers again, but thin)
3. User can edit an item
4. User can delete an item
5. Add validation, error states, edge cases

Each step ships. Each step is testable. Each step proves the integration. If Step 1 reveals an architectural problem, it's discovered before the other four were built on top of it.

### What to check

**Plan structured as sequential horizontal layers**
- Tasks that are purely schema work, followed by purely server function work, followed by purely UI work, with no end-to-end slice until the final task.
- Concrete failure: the integration is only tested at the end. Data shape decisions made in Task 1 (schema) may be incompatible with what the UI needs by Task 4. Rework is expensive and demoralising.
- Fix: reorder so that Task 1 produces a complete (thin) working slice end-to-end. Subsequent tasks extend depth, not width.
- Severity: **P1** if the plan has more than ~4 tasks with no runnable slice until the last one.

**Tasks that deliver no user-visible value**
- "Add Drizzle schema for products table" as a standalone task with no corresponding server function, query hook, or UI.
- Concrete failure: at the end of this task, nothing works for the user. The task cannot be validated except by reading the code. It is invisible progress.
- Fix: combine the schema change with the simplest server function and UI that makes the schema observable — even if it's just a hardcoded empty list rendered from a real query. That is a vertical slice.
- Severity: **P2** for isolated infrastructure tasks. **P1** if multiple consecutive tasks are infrastructure-only.

**Missing "walking skeleton" as the first task**
- The first task in the plan does not establish end-to-end connectivity across all architectural layers that the feature will use.
- Concrete failure: when the first task is pure schema work, the developer doesn't know until much later whether the route, server function, query invalidation, and UI rendering all wire together correctly. Walking skeleton failures (wrong API shape, missing auth context, query key mismatch) are cheapest to fix before any other code is written.
- Fix: make the first task the thinnest possible slice that touches every layer: schema → server function → TanStack Query hook → route → rendered output. It can show an empty state. The data doesn't need to be complete. The architecture does.
- Severity: **P2** when a walking skeleton is missing. **P1** for features that introduce a new architectural pattern (new server function shape, new Shopify API integration, new auth boundary).

---

## Principle 2: Slice by user action, not by technical concern

*Carmack: "Work along feature dimensions, not implementation dimensions. A feature is something the user can do."*
*Cockburn: "Each use case is a complete interaction — start with the basic course of action, then add alternatives."*

Technical concerns (types, schemas, validation, caching) are not user actions. They are implementation details of user actions. Planning by technical concern (all types first, then all schemas, then all validation) means the plan is optimised for developer convenience in the wrong direction — it's easy to write but impossible to ship incrementally.

Slice by what the user can DO:
- "User can view their orders" is a user action → vertical slice
- "Add order types to the Drizzle schema" is a technical concern → horizontal layer (belongs inside the "view orders" slice, not as its own task)

### What to check

**Tasks named after technical artefacts rather than user capabilities**
- Task names like: "Set up Drizzle schema", "Create server functions for X", "Build TanStack Query hooks", "Add TypeScript types".
- These are horizontal layers named directly.
- Fix: rename and restructure to: "User can [see/do/submit/view] X". Then verify that the task actually delivers that capability end-to-end.
- Severity: **P2** — rename and restructure as horizontal layer tasks are identified.

**User-visible behaviour deferred to "wiring" tasks at the end**
- A final task called "Wire UI to API" or "Connect components to data".
- Concrete failure: this task is doing what should have happened in every preceding task. It means none of the earlier tasks actually integrated across layers. All integration risk has been deferred to the last task. That task will take longer than estimated and will uncover the most surprises.
- Severity: **P1** — restructure the plan so integration happens in each task, not at the end.

**Validation and error handling as the last tasks**
- Error states, loading states, and form validation deferred until after "core functionality" is complete.
- This is sometimes correct — validation genuinely can come after the happy path slice works. But it's a smell if the happy path tasks never render an error state at all.
- Fix: ensure at minimum that each slice renders correctly when the server function returns an error. You don't need full validation logic — just that the error is observable. Full validation can be a later slice.
- Severity: **P3** — acceptable if the earlier slices at least surface errors. **P2** if error states are completely absent from the plan.

---

## Principle 3: INVEST — each task must be independently completable

*Carmack: a task that can't be finished until another task is done isn't really a task — it's half a task.*
*Cohn: "Independent slices can be developed in any order, tested in isolation, and prioritised freely."*

INVEST criteria applied to council plan tasks:

| Criterion | Applied to Council Plan Tasks |
|-----------|------------------------------|
| **I**ndependent | The task can be built and verified without the next task being done |
| **N**egotiable | The scope of the task can be adjusted without breaking the plan |
| **V**aluable | A user, developer, or the system is measurably better after this task |
| **E**stimable | The developer can assess whether the task is done |
| **S**mall | The task fits within a focused implementation session (roughly half a day) |
| **T**estable | There is a clear definition of done — something observable proves completion |

### What to check

**Tasks with circular or ambiguous dependencies**
- "Task 4 depends on Task 3 which depends on Task 4" — occurs when tasks aren't cleanly separated along user-action boundaries.
- Concrete failure: the developer doesn't know where to start. They build part of Task 3, part of Task 4, then finish both — which is not two tasks, it's one large task with a misleading split.
- Severity: **P2** — restructure dependencies so they are strictly one-directional.

**Tasks that are too large to verify**
- A single task that covers schema, multiple server functions, multiple UI components, AND validation.
- Concrete failure: "done" is ambiguous. The developer marks it complete, but the reviewer finds three things that weren't implemented. The task wasn't a slice — it was the whole feature in one shot.
- Fix: split at natural user-action boundaries. If the task covers more than one user action, it's at least two tasks.
- Severity: **P2** for tasks covering multiple user actions.

**Tasks with no observable completion signal**
- A task where "done" is not verifiable by running the application — e.g., "add types and schema for the new feature".
- Concrete failure: the implementing agent has no mechanical way to know it's complete. It ships when it thinks it's done, which may not match what was needed.
- Fix: every task must have a sentence starting with "When complete, you should be able to..." that describes an observable end state. If you can't write that sentence, the task is not a vertical slice.
- Severity: **P2**.

---

## Principle 4: The first slice proves the riskiest assumption

*Carmack: "Identify the hardest problem first. If the hard problem can't be solved, the easy problems don't matter."*
*Beck: "Build the walking skeleton first. Prove that the skeleton can walk before adding flesh."*

The hardest assumption in most features is integration: does the auth context flow correctly to the server function? Does the Shopify API respond with the expected shape? Does the TanStack Query invalidation trigger the right refetch? These integration questions don't need a full feature to test — they need ONE thin slice that exercises the full path.

The first slice should be chosen to surface the riskiest unknown, not the easiest win. Starting with a trivial UI component because "it's already designed" defers the real risk. Starting with the route → server function → Shopify API call → rendered result proves the hardest part first.

### What to check

**First task is the easiest task, not the riskiest**
- The plan starts with "Add Drizzle schema" or "Create TypeScript types" — mechanical tasks with no integration risk.
- Concrete failure: integration risks are deferred. By the time the developer discovers that the Shopify Storefront API returns a different shape than expected, three other tasks have been built on top of a wrong assumption.
- Fix: ask "what is the riskiest assumption in this feature?" and make the first task the one that proves or disproves it.
- Severity: **P2** when the first task is infrastructure only. **P1** when there is a known integration risk (new API, new auth boundary, new architectural pattern) that isn't addressed until late in the plan.

**External integrations treated as later tasks**
- A Shopify webhook integration, a new Admin API call, or a Redis caching layer deferred to Task 5+ when it's actually the core of the feature.
- Concrete failure: the earlier tasks build UI and server functions that assume a specific data shape from the external system. When the integration is implemented, the shape is wrong. Everything built before must be revised.
- Severity: **P1** for external integrations that define the data shape of the entire feature.

---

## Principle 5: Horizontal concerns are added to existing slices, not added as new slices

*Carmack: "Performance, validation, and error handling are depth, not breadth. Add them to what exists."*

Some concerns are genuinely horizontal — they apply to every slice. Validation is not a new feature on top of "create item"; it's the same "create item" slice with an additional depth of correctness. Error handling for a TanStack Query mutation is not a new task; it's depth added to the same task that built the mutation.

The failure mode is treating these as separate "polish" tasks at the end:
- Task 6: Add validation
- Task 7: Add error handling
- Task 8: Add loading states

This structure means the feature is "working" (by some definition) for Tasks 1-5, but not shippable until Task 8. Nothing in Tasks 1-5 is actually complete. The plan has created a false sense of progress.

The correct structure: each vertical slice task INCLUDES the validation, error handling, and loading states needed for that slice's user action. Validation for "create item" lives in the "user can create an item" task, not in a separate "add validation" task.

### What to check

**"Polish" tasks clustered at the end of the plan**
- Final tasks named: "Add validation", "Handle errors", "Add loading states", "Clean up".
- Concrete failure: Tasks 1-N are not actually shippable — they're missing the error handling and loading states that make them production-ready. The plan has N fake "done" tasks and one massive "make it real" task at the end.
- Fix: move validation and error handling into the task where the user action is built. Each task is shippable when it's done.
- Severity: **P2** for polish tasks at the end. **P1** if the plan has more than 2 such tasks, suggesting the earlier tasks aren't slices at all.

**Accessibility, analytics, and observability entirely absent**
- No task mentions ARIA labels, error boundaries, or any signal that the feature is working in production.
- These don't need their own tasks — they belong inside the relevant slice task.
- Severity: **P3** — flag it but don't restructure the plan for it. Note in Risks & Watchpoints.

---

## Gaps: What This Doc Doesn't Cover

- **Feature scoping**: what to build is covered by the council's discovery process. This doc covers how to structure what the council has already agreed to build.
- **Code quality within a slice**: correctness, patterns, and architecture within each task are the other experts' domains.
- **Project management**: sprint planning, velocity, story points. This is a code planning doc, not a Scrum doc.
- **Always horizontal concerns**: some work genuinely can't be a vertical slice — a database migration enabling a new feature, a configuration change unblocking multiple slices. Flag these as prerequisites or External Setup items, not as plan tasks.

---

## Quick Reference: Slice Health Check

| Signal | Healthy | Unhealthy |
|--------|---------|-----------|
| Task names | "User can view X", "User can create X" | "Add schema", "Build types", "Create hooks" |
| First task | Thin slice through all layers | Schema or types only |
| Dependencies | One-directional, clear | Circular, ambiguous |
| Final tasks | Extend existing slices | "Wire everything together", "Polish" |
| Each task's "done" | Observable, runnable | "Code complete", "types added" |
| External integrations | First or second task | Last tasks |

### The Overriding Filter

Before accepting the plan structure, apply the Cockburn–Carmack synthesis:

1. **Can I ship Task 1 to production and have a user see something real?** If no, Task 1 is not a vertical slice. Restructure.
2. **Does any task have "Wire", "Connect", or "Integrate" in its title?** If yes, integration is being deferred. It should have happened in every preceding task.
3. **What is the riskiest assumption?** Is it addressed in Task 1 or Task 2? If it's addressed in Task 5, the plan is ordered by comfort, not by risk.
4. **Are validation and error handling inside the action tasks, or are they their own tasks at the end?** They should be inside.
5. **Could the developer demo progress after each task?** Yes = vertical slices. No = horizontal layers.
