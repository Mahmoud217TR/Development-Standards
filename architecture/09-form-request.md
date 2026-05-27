# Form Request

## Definition

A Laravel class that handles **authorization** for a specific endpoint. In this team's convention, Form Requests do NOT handle validation — that lives in Data classes. Form Requests exist purely to centralize the question "is this user allowed to do this?"

## Keywords

Authorization gate • Permission check • Endpoint policy

## Purpose

To separate "are you allowed to do this?" from "is your input shaped correctly?" — the former is endpoint-specific business policy, the latter is data shape. Keeping them separate makes both clearer.

## Rules

1. Every write endpoint has a Form Request.
2. `authorize()` returns `true` if the request is allowed, `false` (or throws) otherwise.
3. **No `rules()` method.** Validation lives in Data classes.
4. No business logic. `authorize()` may call Policies, but the decision must be a simple "allowed/not allowed."
5. **`final` by default.**
6. Name reflects the operation: `StoreOrderRequest`, `UpdateOrderRequest`, `DestroyOrderRequest`, `ShipOrderRequest`.

## Shape

```php
use Illuminate\Foundation\Http\FormRequest;

final class {Verb}{Object}Request extends FormRequest
{
    public function authorize(): bool
    {
        return /* permission check */;
    }
}
```

## Parameters

N/A — Form Requests are injected by Laravel, not called.

## Returns

`authorize()` returns `bool`. `false` causes Laravel to throw a 403.

## Is it `final`?

**Yes.**

## When to use

- Any HTTP write endpoint (POST, PUT, PATCH, DELETE)
- Any read endpoint where access is restricted (e.g., admin-only)
- Endpoints with non-trivial auth logic that benefits from being named and reusable

## When NOT to use

- For validation — use Data classes
- For unauthenticated public endpoints with no access control — skip
- Inside Actions, Queries, Jobs — those don't deal with HTTP

## Lifecycle

1. Request matches route
2. Controller method is resolved
3. Form Request parameter triggers Laravel to construct it
4. `authorize()` runs
5. If `false` → 403 response, controller never runs
6. If `true` → control passes to the next injected dependency (typically the Data class)

## What controls it

- **Triggered by:** controller method declaring it as a parameter
- **Configured by:** override of `authorize()`, optionally `failedAuthorization()` for custom responses
- **May delegate to:** Laravel Policies (`$this->user()->can('update', $this->route('order'))`)

## Examples

### Basic authorization

```php
namespace App\Http\Requests;

use App\Models\Order\Order;
use Illuminate\Foundation\Http\FormRequest;

final class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('create', Order::class) ?? false;
    }
}
```

### Authorization with route model

```php
final class UpdateOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        $order = $this->route('order');
        return $this->user()->can('update', $order);
    }
}
```

### Authorization with custom logic

```php
final class ShipOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        $order = $this->route('order');

        return $this->user()->can('update', $order)
            && $order->status->canBe(Shipped::class);
    }
}
```

### Used together with Data class

```php
public function store(
    StoreOrderRequest $request,    // authorization
    CreateOrderData $data,          // validation + typed input
    PlaceOrder $action,             // operation
) {
    $order = $action->handle($data);
    return OrderData::from($order);
}
```

Order of operations:
1. `StoreOrderRequest::authorize()` runs → 403 if not allowed
2. `CreateOrderData` validates → 422 if invalid
3. Controller body runs

## Anti-examples

### ❌ Validation rules in Form Request

```php
final class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return ['customer_name' => 'required|string'];   // ❌ validation here
    }
}
```

Validation is the Data class's job. Form Requests handle authorization only.

### ❌ Business logic in `authorize()`

```php
public function authorize(): bool
{
    $order = $this->route('order');
    if ($order->status === 'shipped') {
        $order->update(['notified' => true]);   // ❌ side effect
    }
    return $this->user()->can('view', $order);
}
```

`authorize()` is a pure check. No state changes.

### ❌ `authorize()` returning `true` by default

```php
public function authorize(): bool
{
    return true;
}
```

If there's no auth check, skip the Form Request entirely. Returning `true` is misleading — it suggests the question was considered when it wasn't.
