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
3. Dependencies via constructor injection only. No `app()` / `resolve()` inside method bodies.
4. **Synchronous.** Actions never implement `ShouldQueue`.
5. All persistence inside `DB::transaction()` when multiple writes are involved.
6. Synchronous work whose result the response needs runs **inside** the Action (inside the transaction if it modifies the database).
7. Events fired **AFTER** the transaction commits.
8. Returns the canonical result of the operation (usually a Model).
9. Throws domain exceptions for business-rule violations. Does not catch infrastructure exceptions (narrow `try/catch` allowed only for explicit business fallback decisions).
10. Calls allowed: Services, Queries, other Actions (sparingly), Models.
11. **`final` by default.**

## Shape

```php
final class {Verb}{Object}
{
    public function __construct(/* injected dependencies */) {}

    public function handle(/* typed inputs */): /* return type */
    {
        // 1. (optional) preconditions / business rule checks → throw domain exception if violated
        // 2. DB::transaction(fn() => persist())
        // 3. (sync) work whose result the response needs
        // 4. event(new SomethingHappened(...))
        // 5. return result
    }
}
```

## Parameters

- **A DTO** (preferred) carrying validated input
- **Domain models** when operating on existing entities (`Order $order`, `User $user`)
- **Primitives** only when the input is genuinely a single value (`int $orderId`)

Never:
- Accept `Request` directly. The controller hands off a DTO.
- Accept arrays of mixed values. Use a DTO.

## Returns

- The canonical entity the operation produced or affected (e.g. `Order`, `User`)
- A DTO when returning a composite result that includes data outside a single model (e.g., order + computed loyalty points earned)
- `void` only when the operation truly has no result worth communicating

Never return raw arrays from Actions.

## Is it `final`?

**Yes. Always.** Extension is not supported. To customize behavior, write a new Action that composes the existing one or shares a Service.

## When to use

- A user performs a discrete action via HTTP (place an order, register, change a setting)
- A console command triggers a business operation
- A scheduled task orchestrates a write-heavy operation (use a Job if async; Action stays sync)
- Logic involves 2+ steps that must succeed together
- The same logic is needed from multiple entry points (HTTP controller + console command + listener)

## When NOT to use

- **Pure reads** → use a Query
- **Wrapping an external API** (Stripe, SMS provider) → use a Service
- **Async/scheduled work** → use a Job
- **Reacting to an event** → use a Listener
- **Trivial CRUD** without business rules (e.g., toggling a boolean) → can live in the controller; promote to Action when logic emerges
- **Model invariants** (UUIDs, timestamps) → use a model `booted()` hook

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

## Examples

### Standard Action with async side effects

```php
namespace App\Actions\Orders;

use App\Data\Orders\CreateOrderDto;
use App\Events\OrderPlaced;
use App\Exceptions\InsufficientInventoryException;
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

    public function handle(CreateOrderDto $dto): Order
    {
        if (!$this->inventory->isAvailable($dto->items)) {
            throw new InsufficientInventoryException($dto->items);
        }

        $order = DB::transaction(function () use ($dto) {
            $order = Order::create([
                'number' => $this->numbers->next(),
                'customer_name' => $dto->customer_name,
                'phone' => $dto->phone,
            ]);

            $order->items()->createMany($dto->items->toArray());

            return $order;
        });

        // Async side effects: SMS, invoice generation, merchant notification
        event(new OrderPlaced($order));

        return $order;
    }
}
```

### Action with synchronous work whose result the response needs

When the response must include computed data (loyalty points earned, totals with tax, etc.), that work belongs inside the Action:

```php
final class PlaceOrder
{
    public function __construct(
        private InventoryService $inventory,
        private LoyaltyService $loyalty,
    ) {}

    public function handle(CreateOrderDto $dto): OrderConfirmationDto
    {
        if (!$this->inventory->isAvailable($dto->items)) {
            throw new InsufficientInventoryException($dto->items);
        }

        $result = DB::transaction(function () use ($dto) {
            $order = Order::create([...]);
            $order->items()->createMany($dto->items->toArray());

            // Loyalty points must be in the response — sync
            $pointsEarned = $this->loyalty->awardPointsFor($order);

            return new OrderConfirmationDto(
                order: $order,
                points_earned: $pointsEarned,
            );
        });

        // Async side effects fire AFTER the transaction commits
        event(new OrderPlaced($result->order));

        return $result;
    }
}
```

### Action with an explicit business fallback (narrow try/catch)

```php
final class PlaceOrder
{
    public function handle(CreateOrderDto $dto): Order
    {
        $order = DB::transaction(fn () => /* create order */);

        try {
            $this->shipping->createShipment($order);
        } catch (ShippingProviderUnavailableException $e) {
            // Business decision: accept order, retry shipping later
            $order->update(['shipping_status' => 'pending']);
            report($e);
        }

        event(new OrderPlaced($order));

        return $order;
    }
}
```

### Controller usage

```php
public function store(StoreOrderRequest $request, PlaceOrder $action)
{
    $dto = CreateOrderDto::from($request->validated());
    $order = $action->handle($dto);
    return new OrderResource($order);
}
```

The controller:
1. Receives the FormRequest (which has already authorized + validated)
2. Constructs the DTO from the validated payload
3. Invokes the Action with the typed DTO
4. Wraps the result in a Resource
