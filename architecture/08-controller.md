# Controller

## Definition

A class translating HTTP requests into application operations. Controllers are **thin adapters** between the HTTP layer and the business logic — they receive the FormRequest, construct a DTO from validated data, hand it to an Action or Query, then wrap the result in a JsonResource for output.

## Keywords

HTTP adapter • Endpoint • Route handler • Entry point

## Purpose

To handle HTTP protocol concerns (status codes, response format, headers) and to wire HTTP inputs to the right business operation. Controllers are *not* where business logic lives.

## Rules

1. Controllers contain **no business logic**, **no query building**, **no validation rules**.
2. Each method is 3–6 lines.
3. Dependencies (FormRequests, Actions, Queries) received via method-level dependency injection.
4. DTOs constructed inside the controller via `XDto::from($request->validated())`.
5. Responses returned via `new XResource(...)` or `XResource::collection(...)`, or via Laravel's response helpers for non-Resource responses (downloads, redirects, no-content).
6. Resource controllers preferred for standard CRUD; single-action invokable controllers for non-CRUD endpoints.
7. **`final` by default.**

## Shape

### Resource controller

```php
final class {Resource}Controller
{
    public function index(/* DI */): /* response */ {}
    public function show(/* DI */): /* response */ {}
    public function store(/* DI */): /* response */ {}
    public function update(/* DI */): /* response */ {}
    public function destroy(/* DI */): /* response */ {}
}
```

### Invokable single-action controller

```php
final class {VerbObject}Controller
{
    public function __invoke(/* DI */): /* response */
    {
        // single endpoint
    }
}
```

## Parameters

Method-level DI for each handler:
- **FormRequest** for write endpoints (authorize + validate)
- **`Request`** (base) for read endpoints with query parameters
- **Route-bound models** (`Order $order`)
- **Action / Query** for the operation

DTOs are **constructed inside the method body**, not injected:

```php
$dto = CreateOrderDto::from($request->validated());
```

## Returns

- A JsonResource: `new XResource($model)` or `XResource::collection($items)`
- `Illuminate\Http\JsonResponse` for non-Resource JSON
- A redirect, view (Inertia), or download response for browser flows
- `response()->noContent()` for DELETE endpoints

## Is it `final`?

**Yes.**

## When to use

- One Resource controller per HTTP resource for standard CRUD
- One invokable controller per non-CRUD endpoint
- Webhook handlers (invokable)

## When NOT to use

- For business logic — that goes in an Action or Query
- For complex query building — that goes in a Query
- For multi-step orchestration — that's an Action's job
- For shaping output — that's a Resource's job
- For validation rules — that's a FormRequest's job

## Lifecycle

1. Request matches a route
2. Route resolves to controller method
3. Method-level DI runs:
   a. FormRequest constructed → `prepareForValidation()` → `authorize()` → `rules()`
   b. Route models resolved (`Order $order` triggers DB lookup)
   c. Other dependencies (Actions, Queries) injected
4. Controller method body runs (a few lines)
5. Returns Resource → Laravel serializes to JSON
6. Response sent

## What controls it

- **Triggered by:** the Laravel router on incoming HTTP requests
- **Registered by:** route definitions in `routes/web.php` or `routes/api.php`
- **Resource routes:** `Route::resource('orders', OrderController::class)`
- **Invokable:** `Route::post('orders/{order}/ship', MarkOrderAsShippedController::class)`

## Examples

### Standard Resource controller

```php
namespace App\Http\Controllers;

use App\Actions\Orders\PlaceOrder;
use App\Actions\Orders\UpdateOrder;
use App\Actions\Orders\CancelOrder;
use App\Data\Orders\CreateOrderDto;
use App\Data\Orders\UpdateOrderDto;
use App\Data\Orders\OrderFiltersDto;
use App\Http\Requests\Orders\StoreOrderRequest;
use App\Http\Requests\Orders\UpdateOrderRequest;
use App\Http\Requests\Orders\DestroyOrderRequest;
use App\Http\Resources\OrderResource;
use App\Models\Order\Order;
use App\Queries\Orders\ListUserOrdersQuery;
use Illuminate\Http\Request;

final class OrderController
{
    public function index(Request $request, ListUserOrdersQuery $query)
    {
        $filters = OrderFiltersDto::from($request->query());
        return OrderResource::collection(
            $query->handle($request->user(), $filters)
        );
    }

    public function show(Order $order)
    {
        return new OrderResource($order->load('items'));
    }

    public function store(StoreOrderRequest $request, PlaceOrder $action)
    {
        $dto = CreateOrderDto::from($request->validated());
        $order = $action->handle($dto);
        return new OrderResource($order->load('items'));
    }

    public function update(UpdateOrderRequest $request, Order $order, UpdateOrder $action)
    {
        $dto = UpdateOrderDto::from($request->validated());
        $order = $action->handle($order, $dto);
        return new OrderResource($order->load('items'));
    }

    public function destroy(DestroyOrderRequest $request, Order $order, CancelOrder $action)
    {
        $action->handle($order);
        return response()->noContent();
    }
}
```

### Invokable controller for a state transition

```php
namespace App\Http\Controllers\Orders;

use App\Actions\Orders\MarkOrderAsShipped;
use App\Data\Orders\ShipOrderDto;
use App\Http\Requests\Orders\ShipOrderRequest;
use App\Http\Resources\OrderResource;
use App\Models\Order\Order;

final class MarkOrderAsShippedController
{
    public function __invoke(
        ShipOrderRequest $request,
        Order $order,
        MarkOrderAsShipped $action,
    ) {
        $dto = ShipOrderDto::from($request->validated());
        $order = $action->handle($order, $dto);
        return new OrderResource($order);
    }
}
```

Routes:

```php
Route::resource('orders', OrderController::class);
Route::post('orders/{order}/ship', MarkOrderAsShippedController::class);
```

### Per-field-group update controller

When updating distinct slices of a resource through separate endpoints:

```php
namespace App\Http\Controllers;

use App\Actions\Users\UpdateUserName;
use App\Actions\Users\UpdateUserEmail;
use App\Actions\Users\UpdateUserAddress;
use App\Data\Users\UpdateUserNameDto;
use App\Data\Users\UpdateUserEmailDto;
use App\Data\Users\UpdateUserAddressDto;
use App\Http\Requests\Users\UpdateUserNameRequest;
use App\Http\Requests\Users\UpdateUserEmailRequest;
use App\Http\Requests\Users\UpdateUserAddressRequest;
use App\Http\Resources\UserResource;

final class ProfileController
{
    public function updateName(UpdateUserNameRequest $request, UpdateUserName $action)
    {
        $dto = UpdateUserNameDto::from($request->validated());
        return new UserResource($action->handle($request->user(), $dto));
    }

    public function updateEmail(UpdateUserEmailRequest $request, UpdateUserEmail $action)
    {
        $dto = UpdateUserEmailDto::from($request->validated());
        return new UserResource($action->handle($request->user(), $dto));
    }

    public function updateAddress(UpdateUserAddressRequest $request, UpdateUserAddress $action)
    {
        $dto = UpdateUserAddressDto::from($request->validated());
        return new UserResource($action->handle($request->user(), $dto));
    }
}
```

Routes:

```php
Route::patch('/profile/name', [ProfileController::class, 'updateName']);
Route::patch('/profile/email', [ProfileController::class, 'updateEmail']);
Route::patch('/profile/address', [ProfileController::class, 'updateAddress']);
```
