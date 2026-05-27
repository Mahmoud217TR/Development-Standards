# State & Transition

## Definition

Classes representing the **states** an entity can be in (`Pending`, `Shipped`, `Cancelled`) and the **transitions** between them. Built on `spatie/laravel-model-states`. States are persisted as strings on the model and allow the model to behave differently per state. Transitions are the discrete operations that move between states, optionally carrying their own data and logic.

## Keywords

State machine • FSM (finite state machine) • Status • Lifecycle stage • Transition

## Purpose

To make state changes explicit and enforceable. Instead of `$order->status === 'pending'` checks scattered across the codebase, the model holds a `State` object that knows its allowed transitions and per-state behavior.

## Rules

### States

1. Each state is a class extending an abstract base (`OrderState`).
2. Located co-located with the model: `app/Models/{Model}/States/`.
3. The abstract base declares the allowed transitions and default state on the model.
4. State classes can define methods that vary per state (e.g., `label()`, `color()`, `canBeCancelled()`).
5. **`final` by default** (the abstract base is `abstract`).

### Transitions

1. A transition class is created **only when the transition has its own logic** (sets extra fields, fires specific events, requires extra data).
2. Located in `app/Models/{Model}/States/Transitions/`.
3. Extends `Spatie\ModelStates\Transition`.
4. Constructor takes the model and any transition-specific data.
5. `handle()` returns the model after the state change.
6. Transition logic is **atomic** — runs inside `DB::transaction()` if multiple fields change.
7. Transition classes do not fire events; the **Action** that invokes the transition fires the event.
8. **`final` by default.**

### Use of states in business logic

1. Actions orchestrate state changes: they call the transition, then fire events.
2. Trivial transitions (just a status field change, no extra data) don't need a Transition class — call `$model->status->transitionTo(NewState::class)` directly in the Action.

## Shape

### Abstract base state

```php
namespace App\Models\{Model}\States;

use Spatie\ModelStates\State;
use Spatie\ModelStates\StateConfig;

abstract class {Model}State extends State
{
    abstract public function label(): string;

    public static function config(): StateConfig
    {
        return parent::config()
            ->default(Pending::class)
            ->allowTransition(Pending::class, Shipped::class)
            ->allowTransition(Shipped::class, Delivered::class)
            ->allowTransition(Pending::class, Cancelled::class);
    }
}
```

### Concrete state

```php
final class Pending extends {Model}State
{
    public function label(): string
    {
        return 'Pending';
    }
}
```

### Transition (when extra logic is needed)

```php
namespace App\Models\{Model}\States\Transitions;

use Spatie\ModelStates\Transition;

final class PendingToShipped extends Transition
{
    public function __construct(
        private {Model} $model,
        private string $trackingNumber,
    ) {}

    public function handle(): {Model}
    {
        $this->model->status = new Shipped($this->model);
        $this->model->tracking_number = $this->trackingNumber;
        $this->model->shipped_at = now();
        $this->model->save();

        return $this->model;
    }
}
```

## Parameters

- States: none (they describe a position, not an operation)
- Transitions: the model + any data the transition needs

## Returns

- States: N/A
- Transitions: the model after the state change

## Is it `final`?

**Yes**, for concrete states and transitions. Abstract base states are `abstract`.

## When to use

### Use state machines when:
- The entity has 3+ distinct states
- Transitions between states must be restricted (Pending → Cancelled OK, Shipped → Pending NOT OK)
- State affects what operations are allowed
- State affects what fields are required/relevant

### Use Transition classes when:
- The transition sets additional fields beyond the status (tracking number, refunded_at, etc.)
- The transition has validation logic of its own
- The transition is non-trivial enough to deserve a name

### Don't use Transition classes when:
- The state change is purely a status field update
- The transition has no logic beyond `$model->status = new NewState($model); $model->save();`

## When NOT to use state machines at all

- Simple booleans (`is_active`, `is_published`)
- 2-state flags that are unlikely to grow
- Status fields with no rules on transitions

## Lifecycle

### State persistence
1. Model has a `status` column (string)
2. Eloquent casts it to a State subclass via the abstract base's `morphMap` or `cast`
3. `$model->status` returns a State instance, not a string
4. Saving the model serializes the state back to a string

### Transition flow
1. Action receives the model
2. Action invokes the transition: `$model->status->transitionTo(Shipped::class, $trackingNumber)`
   OR `app(PendingToShipped::class, ['model' => $model, 'trackingNumber' => $tn])->handle()`
3. Spatie checks if the transition is allowed → throws `CouldNotPerformTransition` if not
4. Transition class runs `handle()` → updates model fields + saves
5. Action fires the relevant Event (e.g., `OrderShipped`)
6. Listeners react

## What controls it

- **State persistence:** Eloquent + Spatie's casting
- **Allowed transitions:** declared in the abstract base's `config()` method
- **Default state:** declared in `config()->default(...)`
- **Transition execution:** Action calls `transitionTo()` or instantiates a Transition class

## Examples

### Full setup

```php
// app/Models/Order/States/OrderState.php
namespace App\Models\Order\States;

use Spatie\ModelStates\State;
use Spatie\ModelStates\StateConfig;

abstract class OrderState extends State
{
    abstract public function label(): string;
    abstract public function color(): string;

    public static function config(): StateConfig
    {
        return parent::config()
            ->default(Pending::class)
            ->allowTransition(Pending::class, Shipped::class, Transitions\PendingToShipped::class)
            ->allowTransition(Shipped::class, Delivered::class)
            ->allowTransition(Pending::class, Cancelled::class);
    }
}
```

```php
// app/Models/Order/States/Pending.php
final class Pending extends OrderState
{
    public function label(): string { return 'Pending'; }
    public function color(): string { return 'amber'; }
}

// app/Models/Order/States/Shipped.php
final class Shipped extends OrderState
{
    public function label(): string { return 'Shipped'; }
    public function color(): string { return 'blue'; }
}
```

```php
// app/Models/Order/States/Transitions/PendingToShipped.php
namespace App\Models\Order\States\Transitions;

use App\Models\Order\Order;
use App\Models\Order\States\Shipped;
use Spatie\ModelStates\Transition;

final class PendingToShipped extends Transition
{
    public function __construct(
        private Order $order,
        private string $trackingNumber,
    ) {}

    public function handle(): Order
    {
        $this->order->status = new Shipped($this->order);
        $this->order->tracking_number = $this->trackingNumber;
        $this->order->shipped_at = now();
        $this->order->save();

        return $this->order;
    }
}
```

```php
// app/Models/Order/Order.php
namespace App\Models\Order;

use App\Models\Order\States\OrderState;
use Illuminate\Database\Eloquent\Model;
use Spatie\ModelStates\HasStates;

final class Order extends Model
{
    use HasStates;

    protected $casts = [
        'status' => OrderState::class,
    ];
}
```

### Action invoking the transition

```php
final class MarkOrderAsShipped
{
    public function handle(Order $order, string $trackingNumber): Order
    {
        $order = $order->status->transitionTo(Shipped::class, $trackingNumber);
        event(new OrderShipped($order));
        return $order;
    }
}
```

### Trivial transition (no Transition class)

```php
final class CancelOrder
{
    public function handle(Order $order): Order
    {
        $order->status->transitionTo(Cancelled::class);
        event(new OrderCancelled($order));
        return $order;
    }
}
```

## Anti-examples

### ❌ Transition firing events

```php
final class PendingToShipped extends Transition
{
    public function handle(): Order
    {
        $this->order->status = new Shipped($this->order);
        $this->order->save();
        event(new OrderShipped($this->order));   // ❌
        return $this->order;
    }
}
```

Transitions are atomic state changes. Events belong in the Action that orchestrates the use case. Otherwise an Action that wraps the transition can't control whether the event fires (e.g., during a bulk operation).

### ❌ Transition calling Services or making decisions

```php
public function handle(): Order
{
    if ($this->order->total > 1000) {            // ❌ business decision
        $this->order->priority = 'high';
    }
    $this->shippingApi->createShipment(...);     // ❌ external call
    $this->order->save();
    return $this->order;
}
```

Transitions update fields and save. Decisions and side effects belong in the Action.

### ❌ Using a state machine for a boolean flag

```php
abstract class PublishedState extends State { /* ... */ }
final class Published extends PublishedState {}
final class Unpublished extends PublishedState {}
```

Overkill. Use a boolean column.
