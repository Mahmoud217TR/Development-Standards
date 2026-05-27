# Action

## Definition

A class representing one specific business operation the system performs. One use case, one entry point, one outcome. Actions orchestrate work: they validate preconditions, persist changes inside a transaction, fire events for side effects, and return the result. They are the **top-level write path** of the application.

## Keywords

Operation • Use case • Command • Write • Synchronous business logic

## Purpose

To localize business logic into named, testable units. Each Action represents one thing a user or system can *do*, and contains everything required to do it consistently.

## Rules

1. One Action per use case. No "manager" Actions with multiple methods.
2. Exactly one public method: `handle()`.
3. Dependencies via constructor injection only.
4. **Synchronous.** Actions never implement `ShouldQueue`.
5. All persistence inside `DB::transaction()` when multiple writes are involved.
6. Events fired AFTER the transaction commits.
7. Returns the canonical result of the operation (usually a Model or a Data class).
8. Calls allowed: Services, Queries, other Actions (sparingly), Models.
9. **`final` by default.**

## Shape

```php
final class {Verb}{Object} 
{
    public function __construct(/* injected dependencies */) {}

    public function handle(/* typed inputs */): /* return type */
    {
        // 1. (optional) preconditions / business rule checks
        // 2. DB::transaction(fn() => persist())
        // 3. event(new SomethingHappened(...))
        // 4. return result
    }
}
```

## Parameters

- **A Data class** (preferred) carrying validated input
- **Domain models** when operating on existing entities (`Order $order`, `User $user`)
- **Primitives** only when the input is genuinely a single value (`int $orderId`)

Never:
- Accept `Request` directly. The controller hands off a Data class.
- Accept arrays of mixed values. Use a Data class.

## Returns

- The canonical entity the operation produced or affected (e.g. `Order`, `User`)
- A Data class when returning a composite result
- `void` only when the operation truly has no result worth communicating

Never return raw arrays from Actions.

## Is it `final`?

**Yes. Always.** Extension is not supported. To customize behavior, write a new Action that composes the existing one or shares a Service.

## When to use

- A user performs a discrete action via HTTP (place an order, register, change a setting)
- A console command triggers a business operation
- A scheduled job orchestrates a write-heavy operation (use a Job if async; Action stays sync)
- Logic involves 2+ steps that must succeed together
- The same logic is needed from multiple entry points (HTTP controller + console command + listener)

## When NOT to use

- **Pure reads** → use a Query
- **Wrapping an external API** (Stripe, SMS provider) → use a Service
- **Async/scheduled work** → use a Job
- **Reacting to an event** → use a Listener
- **Trivial CRUD** without business rules (e.g., toggling a boolean) → can live in the controller; promote to Action when logic emerges
- **Model invariants** (UUIDs, timestamps) → use a model trait hook

## Lifecycle

1. Container resolves the Action when the controller (or other entry point) declares it as a parameter
2. Constructor runs, dependencies injected
3. Caller invokes `handle(...)` with typed inputs
4. Action runs synchronously
5. Returns; container discards the instance

## What controls it

- **Triggered by:** controllers, console commands, jobs, listeners, other Actions
- **Registered by:** Laravel's auto-resolved container — no explicit registration needed
- **Configured by:** constructor signature

## Example

```php
namespace App\Actions\Orders;

use App\Data\Orders\CreateOrderData;
use App\Events\OrderPlaced;
use App\Models\Order\Order;
use App\Services\OrderNumberGenerator;
use App\Services\InventoryService;
use Illuminate\Support\Facades\DB;

final class PlaceOrder
{
    public function __construct(
        private InventoryService $inventory,
        private OrderNumberGenerator $numbers,
    ) {}

    public function handle(CreateOrderData $data): Order
    {
        $this->inventory->assertAvailable($data->items);

        $order = DB::transaction(function () use ($data) {
            $order = Order::create([
                'number' => $this->numbers->next(),
                'customer_name' => $data->customer_name,
                'phone' => $data->phone,
            ]);

            $order->items()->createMany($data->items->toArray());

            return $order;
        });

        event(new OrderPlaced($order));

        return $order;
    }
}
```

Controller usage:

```php
public function store(
    StoreOrderRequest $request,   // authorization
    CreateOrderData $data,         // validation
    PlaceOrder $action,            // operation
) {
    $order = $action->handle($data);
    return OrderData::from($order);
}
```

## Anti-examples

### ❌ Multiple public methods

```php
final class OrderService
{
    public function create(...) {}
    public function cancel(...) {}
    public function ship(...) {}
}
```

This is a Service in name only — it's an accumulator. Each method is its own use case and deserves its own Action.

### ❌ Action accepting `Request`

```php
public function handle(Request $request): Order
{
    $name = $request->input('customer_name');
    // ...
}
```

Now the Action is coupled to HTTP. It can't be called from a console command or listener cleanly. Use a Data class.

### ❌ Side effects inside the transaction

```php
DB::transaction(function () use ($order) {
    Order::create([...]);
    Sms::send($phone, 'Order placed');  // external HTTP call inside tx
});
```

If SMS fails, the order rolls back. If SMS succeeds but DB commit fails, the customer got an SMS for an order that doesn't exist. Fire an Event after the transaction commits.

### ❌ Action implementing `ShouldQueue`

```php
final class PlaceOrder implements ShouldQueue { /* ... */ }
```

Actions are synchronous. If you need this work to run async, dispatch a Job that internally calls the Action, or model the work as a Job to begin with.
