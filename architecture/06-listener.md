# Listener

## Definition

A class that reacts to one or more Events. Listeners do **one side effect** in response to a happening — send an SMS, fire a webhook, update an external index, log an analytics event. They are the **fan-out reactions** to Events fired by Actions and Jobs.

## Keywords

Reaction • Subscriber • Handler • Side effect

## Purpose

To keep Actions focused on the core operation while still triggering arbitrarily many side effects. Each Listener does one thing; they run in parallel, can fail independently, and can be added/removed without touching the Action.

## Rules

1. One Listener per reaction. Don't make one Listener that sends SMS AND emails AND fires a webhook.
2. **Implements `ShouldQueue` by default.** Listeners run async.
3. Exactly one public method: `handle({EventClass} $event)`.
4. Named after what it does, not what it reacts to: `SendOrderConfirmationSms`, not `OrderPlacedListener`.
5. Listeners may use Services and Queries. They MUST NOT call Actions.
6. Listeners SHOULD NOT fire further Events (avoid chains).
7. **`final` by default.**
8. Failures are retried per queue configuration; persistent failures go to the failed jobs table.

## Shape

```php
use Illuminate\Contracts\Queue\ShouldQueue;

final class {VerbObject} implements ShouldQueue
{
    public function __construct(/* injected dependencies */) {}

    public function handle({EventClass} $event): void
    {
        // one side effect
    }
}
```

## Parameters

The Event class is the sole parameter to `handle()`.

## Returns

`void`. Listeners are fire-and-forget.

## Is it `final`?

**Yes.**

## When to use

- A reaction to an Event that:
  - Is slow (external API call, file processing)
  - Can fail independently of the main operation
  - Is one of many possible reactions to the same Event
- The reaction doesn't need to be visible in the original Action's code

## When NOT to use

- **The reaction is required for the operation to be valid** → put it in the Action's transaction
- **You only have one reaction and it'll always be there** → dispatch a Job directly from the Action, skip the Event
- **The reaction returns data the Action needs** → not a fit for events; restructure
- **For business decisions** → Listeners do side effects, not business rules. Decisions belong in Actions

## Lifecycle

1. Event is fired
2. Laravel resolves all Listeners subscribed to this Event class
3. If `ShouldQueue` → Listener is pushed to the configured queue
4. Worker picks up the listener job
5. Container resolves the Listener instance with dependencies
6. `handle($event)` runs
7. On failure → retried per `$tries` / `$backoff` config; final failure goes to `failed_jobs`

## What controls it

- **Triggered by:** an Event being fired
- **Registered by:** Laravel auto-discovery (matches `handle({EventClass})` signature) or explicit registration in `EventServiceProvider::$listen`
- **Configured by:** properties on the Listener class:
  - `public int $tries = 3` — retry count
  - `public int $backoff = 60` — seconds between retries
  - `public string $queue = 'notifications'` — queue name
  - `public int $timeout = 30` — max execution seconds

## Example

```php
namespace App\Listeners;

use App\Events\OrderPlaced;
use App\Services\SmsService;
use Illuminate\Contracts\Queue\ShouldQueue;

final class SendOrderConfirmationSms implements ShouldQueue
{
    public int $tries = 3;
    public int $backoff = 60;
    public string $queue = 'notifications';

    public function __construct(private SmsService $sms) {}

    public function handle(OrderPlaced $event): void
    {
        $this->sms->send(
            $event->order->phone,
            "Order {$event->order->number} received. Thank you!",
        );
    }
}
```

Multiple Listeners on the same Event:

```php
final class NotifyMerchantOfNewOrder implements ShouldQueue
{
    public function handle(OrderPlaced $event): void { /* ... */ }
}

final class UpdateProductInventory implements ShouldQueue
{
    public function handle(OrderPlaced $event): void { /* ... */ }
}

final class RecordOrderAnalytics implements ShouldQueue
{
    public function handle(OrderPlaced $event): void { /* ... */ }
}
```

All run in parallel on the queue after `OrderPlaced` fires.

