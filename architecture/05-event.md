# Event

## Definition

An immutable record of *something that happened*. Events carry the relevant context (usually a domain entity) and are emitted by Actions or Jobs after a meaningful occurrence. They are **broadcasts**, not commands — they say "this happened" and let any interested Listener react.

## Keywords

Notification • Broadcast • Signal • Past-tense fact • Domain event

## Purpose

To decouple the thing that happened from the things that react to it. The Action that placed an order does not need to know which Listeners will send SMS, update inventory, log analytics, or notify the merchant. It just fires the event.

## Rules

1. Named in past tense: `OrderPlaced`, `UserRegistered`, `PaymentFailed`. Never imperative.
2. Carry the **domain entity** that the event is about, not raw fields.
3. **Immutable.** Public readonly properties only.
4. No methods beyond the constructor.
5. **`final` by default.**
6. Fired only by Actions and Jobs.
7. Fired AFTER any database transaction commits.
8. Listeners (not the Event) decide what to do.

## Shape

```php
final class {Subject}{PastTenseVerb}
{
    public function __construct(
        public readonly /* domain entity */ ${name},
    ) {}
}
```

## Parameters

The constructor receives the domain context. Usually one entity, occasionally two (`PaymentReceived(Order $order, Payment $payment)`).

## Returns

N/A — Events don't return; they exist.

## Is it `final`?

**Yes.**

## When to use

- Multiple independent reactions to one occurrence (SMS, email, webhook, analytics)
- Reactions belong to different parts of the system that shouldn't know about each other
- The list of reactions may grow over time
- You want async by default (queued Listeners)
- A state transition occurred and other parts of the system care

## When NOT to use

- **There's only one reaction and it will always be there** → call the Service or dispatch the Job directly
- **The reaction is part of the operation itself** ("the order isn't valid unless inventory decrements") → keep it inside the Action's transaction
- **The HTTP response needs the result of the reaction** (loyalty points to display, computed totals) → run the work synchronously inside the Action
- **You want to chain operations** ("after A succeeds, do B, then C") → use job chaining, not events
- **You want a return value from the reaction** → events fire-and-forget; you won't get one
- **For audit/logging only** → use proper logging or a model lifecycle hook

## Lifecycle

1. Action (or Job) finishes its atomic work
2. Action commits transaction
3. Action calls `event(new SomethingHappened($entity))`
4. Laravel's event dispatcher resolves registered Listeners for this event class
5. Each Listener that implements `ShouldQueue` is dispatched to the queue
6. Synchronous Listeners run immediately
7. Action returns to its caller; Listeners run independently afterwards

## What controls it

- **Fired by:** Actions, Jobs, and (rarely) the framework itself
- **Registered Listeners:** Laravel auto-discovers listeners via reflection in modern Laravel, or via the `EventServiceProvider::$listen` array
- **Subscribed to by:** Listener classes (one Listener may subscribe to multiple events)

## Example

```php
namespace App\Events;

use App\Models\Order\Order;

final class OrderPlaced
{
    public function __construct(
        public readonly Order $order,
    ) {}
}
```

Firing it from an Action:

```php
public function handle(CreateOrderDto $dto): Order
{
    $order = DB::transaction(fn () => Order::create([...]));
    event(new OrderPlaced($order));
    return $order;
}
```

Subscribing listeners (via convention — Laravel auto-discovers):

```php
final class SendOrderConfirmationSms implements ShouldQueue
{
    public function handle(OrderPlaced $event): void { /* ... */ }
}

final class NotifyMerchantOfNewOrder implements ShouldQueue
{
    public function handle(OrderPlaced $event): void { /* ... */ }
}
```

