# Architecture Reference

This folder is the deep reference for each architectural building block. Each file follows the same template so they're easy to compare.

## Template

Every building block file contains:

1. **Definition** — what it is in one paragraph
2. **Keywords** — synonyms / mental model words (Reaction, Operation, etc.)
3. **Purpose** — what problem it solves
4. **Rules** — must-follow conventions
5. **Shape** — class signature, method signature, naming
6. **Parameters** — what it takes as input
7. **Returns** — what it produces
8. **Is it `final`?** — yes/no/depends
9. **When to use** — concrete scenarios
10. **When NOT to use** — anti-patterns and red flags
11. **Lifecycle** — when it's instantiated, when it runs, when it ends
12. **What controls it** — what triggers it, what registers it
13. **Examples** — at least one complete, runnable example
14. **Anti-examples** — common mistakes with explanations

## Flow Reference

A single request from HTTP to response:

```
1. HTTP Request arrives
2. Router resolves the Controller method
3. Form Request boots → runs authorize() → 403 if denied
4. Data class boots → validates input → 422 if invalid → hydrated as typed DTO
5. Controller method receives Form Request, Data class, and any Actions/Queries via DI
6. Controller calls Action::handle($data) or Query::handle($filters)
7. Action runs:
   a. Opens DB transaction
   b. Calls Services / other Actions / Queries as needed
   c. Persists changes via Eloquent (model lifecycle traits fire here)
   d. Commits transaction
   e. Fires Event(s) via event() helper
   f. Optionally dispatches Job(s) for delayed/scheduled work
   g. Returns result to controller
8. Listeners (queued) pick up the Event later
9. Each Listener runs in its own job, does one side effect
10. Controller serializes the Action's result via Data class
11. HTTP Response returned to client
```

## Files

- [`01-action.md`](./01-action.md)
- [`02-query.md`](./02-query.md)
- [`03-service.md`](./03-service.md)
- [`04-dto.md`](./04-dto.md)
- [`05-event.md`](./05-event.md)
- [`06-listener.md`](./06-listener.md)
- [`07-job.md`](./07-job.md)
- [`08-controller.md`](./08-controller.md)
- [`09-form-request.md`](./09-form-request.md)
- [`10-model-trait.md`](./10-model-trait.md)
- [`11-state-and-transition.md`](./11-state-and-transition.md)
- [`12-model.md`](./12-model.md)

## Glossary of Keywords

| Term | Meaning |
|---|---|
| **Operation** | A discrete, named thing the system does. Maps to an Action. |
| **Use case** | Same as Operation. |
| **Reaction** | Something that happens *because* an operation happened. Maps to a Listener. |
| **Capability** | A technical ability (send SMS, render PDF, talk to Stripe). Maps to a Service. |
| **Plumbing** | Model-level invariants (UUIDs, slugs, audit). Maps to model traits. |
| **Fan-out** | When one event produces many independent reactions. |
| **Side effect** | Any change outside the immediate operation's scope (notifications, external calls). |
| **Atomic** | Happens as one unit; succeeds or rolls back fully. Achieved via DB transactions. |
| **Synchronous** | Runs on the request thread, the caller waits for the result. |
| **Asynchronous** | Runs later on a queue, the caller doesn't wait. |
| **Dispatch** | Send a job to the queue (`Job::dispatch(...)`). |
| **Fire** | Emit an event (`event(new ...)`). |
| **Invoke** | Call a synchronous Action or Query. |
| **DTO** | Data Transfer Object — typed structure passed between layers. Maps to a Data class. |
| **Entry point** | Code that initiates a flow: controller, job, console command, listener. |
