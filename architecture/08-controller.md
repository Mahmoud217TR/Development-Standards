# Controller

## Definition

A class translating HTTP requests into application operations. Controllers are **thin adapters** between the HTTP layer and the business logic — they receive the request, hand it to a Form Request (auth), a Data class (validation), and an Action or Query (operation), then format the response.

## Keywords

HTTP adapter • Endpoint • Route handler • Entry point

## Purpose

To handle the HTTP protocol concerns (status codes, response format, headers) and to wire HTTP inputs to the right business operation. Controllers are *not* where business logic lives.

## Rules

1. Controllers contain **no business logic**, **no query building**, **no validation beyond DI**.
2. Each method is 1–5 lines (after parameter declarations).
3. Dependencies, Form Requests, Data classes, Actions, and Queries are received via method-level dependency injection.
4. Return responses via Data classes (auto-serialized) or Laravel's response helpers.
5. Resource controllers are preferred for standard CRUD (`index`, `show`, `store`, `update`, `destroy`); single-action invokable controllers for non-CRUD endpoints.
6. **`final` by default.**

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
- **Form Request** for write endpoints (authorization)
- **Data class** for input validation + typed data
- **Filter Data class** for read endpoints (`OrderFilters`)
- **Route-bound models** (`Order $order`)
- **Action / Query** for the operation
- **Other Services** if directly needed (rare; usually go through Action)

## Returns

- A Data class instance (serializes to JSON automatically)
- `Illuminate\Http\JsonResponse` for non-Data responses
- A redirect, view (Inertia), or download response for browser flows

## Is it `final`?

**Yes.**

## When to use

- One controller per resource for standard CRUD
- One invokable controller per non-CRUD endpoint (e.g., `MarkOrderAsShippedController`)
- Webhook handlers (also invokable)

## When NOT to use

- For business logic — that goes in an Action or Query
- For complex query building — that goes in a Query
- For multi-step orchestration — that's an Action's job

## Lifecycle

1. Request matches a route
2. Route resolves to controller method
3. Method-level DI runs:
   a. Form Request boots → `authorize()` → throws 403 if denied
   b. Data class is resolved + validated → throws 422 if invalid
   c. Other dependencies injected
4. Controller method runs (a few lines)
5. Returns response

## What controls it

- **Triggered by:** the Laravel router on incoming HTTP requests
- **Registered by:** route definitions in `routes/web.php` or `routes/api.php`
- **Resource routes:** `Route::resource('orders', OrderController::class)`
- **Invokable:** `Route::post('orders/{order}/ship', MarkOrderAsShippedController::class)`

## Examples

### Resource controller

```php
namespace App\Http\Controllers;

use App\Actions\Orders\PlaceOrder;
use App\Actions\Orders\CancelOrder;
use App\Data\Orders\CreateOrderData;
use App\Data\Orders\OrderData;
use App\Data\Orders\OrderFilters;
use App\Http\Requests\StoreOrderRequest;
use App\Http\Requests\DestroyOrderRequest;
use App\Models\Order\Order;
use App\Queries\Orders\ListUserOrdersQuery;
use Illuminate\Support\Facades\Auth;

final class OrderController
{
    public function index(OrderFilters $filters, ListUserOrdersQuery $query)
    {
        return OrderData::collect(
            $query->handle(Auth::user(), $filters)
        );
    }

    public function show(Order $order)
    {
        return OrderData::from($order);
    }

    public function store(StoreOrderRequest $request, CreateOrderData $data, PlaceOrder $action)
    {
        $order = $action->handle($data);
        return OrderData::from($order);
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
use App\Data\Orders\OrderData;
use App\Data\Orders\ShipOrderData;
use App\Http\Requests\ShipOrderRequest;
use App\Models\Order\Order;

final class MarkOrderAsShippedController
{
    public function __invoke(
        ShipOrderRequest $request,
        Order $order,
        ShipOrderData $data,
        MarkOrderAsShipped $action,
    ) {
        $order = $action->handle($order, $data->tracking_number);
        return OrderData::from($order);
    }
}
```

Routes:

```php
Route::resource('orders', OrderController::class);
Route::post('orders/{order}/ship', MarkOrderAsShippedController::class);
```

## Anti-examples

### ❌ Business logic in controller

```php
public function store(Request $request)
{
    $request->validate([...]);
    $order = Order::create($request->all());
    $order->items()->createMany($request->items);
    Sms::send($request->phone, 'Order placed');
    Mail::to($request->email)->send(new OrderConfirmation($order));
    return $order;
}
```

Validation, persistence, side effects, all inline. Extract to Form Request + Data class + Action + Listeners.

### ❌ Query building in controller

```php
public function index(Request $request)
{
    $query = Order::query();
    if ($request->status) $query->where('status', $request->status);
    if ($request->search) $query->where('name', 'like', "%{$request->search}%");
    // ...20 more lines
    return $query->paginate();
}
```

Extract to a Query class.

### ❌ Controllers with many unrelated methods

```php
final class HomeController
{
    public function index() {}
    public function dashboard() {}
    public function settings() {}
    public function notifications() {}
    public function billing() {}
}
```

Group by resource, not by page. If endpoints don't share a resource, use invokable controllers.
