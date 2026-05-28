# Architecture Reference

This folder is the deep reference for each architectural building block. Each file follows the same template so they're easy to compare.

## Template

Every building block file contains:

1. **Definition** — what it is in one paragraph
2. **Keywords** — synonyms / mental model words
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

## Flow Reference

A single request from HTTP to response:

```
1. HTTP Request arrives
2. Router resolves the Controller method
3. FormRequest is constructed by Laravel
   a. prepareForValidation() runs (optional input transformation)
   b. authorize() → 403 if denied
   c. rules() runs against the input → 422 if invalid
   d. withValidator() after-hooks run (optional)
4. Controller method receives FormRequest with validated data
5. Controller calls Dto::from($request->validated()) to construct typed DTO
6. Controller invokes Action::handle($dto) or Query::handle($filtersDto)
7. Action runs:
   a. (optional) Preconditions / business rule checks → throw domain exception if violated
   b. Opens DB transaction
   c. Calls Services / other Actions / Queries as needed
   d. Persists changes via Eloquent (model lifecycle hooks fire here)
   e. (sync) Any work whose result the response needs runs here
   f. Commits transaction
   g. Fires Event(s) via event() helper — listeners run async
   h. Optionally dispatches Job(s) for delayed/scheduled work
   i. Returns result (Model) to controller
8. Listeners (queued) pick up the Event later, each runs in its own job
9. Controller wraps the result in a JsonResource: new XResource($model) or XResource::collection($items)
10. Laravel serializes the Resource to JSON
11. HTTP Response returned to client

If any step throws an exception that isn't caught:
  → bubbles up to bootstrap/app.php exception handler
  → handler translates to HTTP response (status + body)
  → optionally reports to monitoring
```

## Files

- [`01-action.md`](./01-action.md) — Action
- [`02-query.md`](./02-query.md) — Query
- [`03-service.md`](./03-service.md) — Service
- [`04-dto.md`](./04-dto.md) — DTO (transport)
- [`05-event.md`](./05-event.md) — Event
- [`06-listener.md`](./06-listener.md) — Listener
- [`07-job.md`](./07-job.md) — Job
- [`08-controller.md`](./08-controller.md) — Controller
- [`09-form-request.md`](./09-form-request.md) — FormRequest (authorization + validation)
- [`10-model-trait.md`](./10-model-trait.md) — Model Lifecycle (boot/booted + traits)
- [`11-state-and-transition.md`](./11-state-and-transition.md) — State & Transition
- [`12-model.md`](./12-model.md) — Model
- [`13-exception.md`](./13-exception.md) — Exception
- [`14-resource.md`](./14-resource.md) — JsonResource (output)

## The HTTP I/O triad

The HTTP boundary uses three distinct tools, each with one job:

| Tool | Job | Lives in | Suffix |
|---|---|---|---|
| **FormRequest** | Authorize + Validate input | `app/Http/Requests/` | `Request` |
| **DTO** | Carry validated data between layers | `app/Data/` | `Dto` |
| **JsonResource** | Format Model into JSON output | `app/Http/Resources/` | `Resource` |

The flow on every endpoint:

```
HTTP → FormRequest (validate) → DTO (transport) → Action (logic) → Model → Resource (serialize) → HTTP
```

## Glossary of Keywords

| Term | Meaning |
|---|---|
| **Operation** | A discrete, named thing the system does. Maps to an Action. |
| **Use case** | Same as Operation. |
| **Reaction** | Something that happens *because* an operation happened. Maps to a Listener. |
| **Capability** | A technical ability (send SMS, render PDF, talk to Stripe). Maps to a Service. |
| **Plumbing** | Model-level invariants (UUIDs, slugs, audit). Lives in model boot/booted or traits. |
| **Fan-out** | When one event produces many independent reactions. |
| **Side effect** | Any change outside the immediate operation's scope (notifications, external calls). |
| **Atomic** | Happens as one unit; succeeds or rolls back fully. Achieved via DB transactions. |
| **Synchronous** | Runs on the request thread, the caller waits for the result. |
| **Asynchronous** | Runs later on a queue, the caller doesn't wait. |
| **Dispatch** | Send a job to the queue (`Job::dispatch(...)`). |
| **Fire** | Emit an event (`event(new ...)`). |
| **Invoke** | Call a synchronous Action or Query. |
| **DTO** | Data Transfer Object — typed structure passed between layers. Transport only, no validation. |
| **Entry point** | Code that initiates a flow: controller, job, console command, listener. |
| **Domain exception** | An exception type representing a business-rule violation; defined in `app/Exceptions/`. |
