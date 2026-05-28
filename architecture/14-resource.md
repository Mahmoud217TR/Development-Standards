# JsonResource

## Definition

A Laravel class that transforms a domain Model (or Collection) into a JSON-serializable array for HTTP output. JsonResources are the **output-shaping layer** of the application — they decide what fields are exposed, in what format, with what conditional logic.

## Keywords

Output • Serializer • Response shaper • Presentation

## Purpose

To separate "what an Order is in the database" (Model) from "what an Order looks like in API responses" (Resource). Resources keep controllers free of field-by-field array building and centralize all output formatting decisions per resource type.

## Rules

1. **Every model that crosses the HTTP boundary has a Resource.**
2. Resources do **output formatting only**. No business logic, no database writes, no Action invocation.
3. Use `when()` / `whenLoaded()` / `whenCounted()` for conditional fields rather than `if/else` blocks.
4. Resources may format values (dates, money, enums) but never compute business values (totals with tax, derived statuses).
5. **`final` by default.**
6. Naming: `{X}Resource` — `OrderResource`, `UserResource`, `ProductResource`.
7. Use `collection()` for paginated/list responses — never instantiate the resource in a loop manually.

## Shape

```php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

final class {X}Resource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            // typed exposed fields
        ];
    }
}
```

## Parameters

The Resource constructor receives the Model (or array) being wrapped. The `toArray(Request $request)` method receives the current HTTP Request for context-aware decisions.

Inside `toArray()`, `$this->` proxies to the underlying Model's properties.

## Returns

`toArray()` returns an `array` that Laravel serializes to JSON.

## Is it `final`?

**Yes.**

## When to use

- **Every HTTP write endpoint** returns a Resource (`new OrderResource($order)`)
- **Every HTTP read endpoint** returns a Resource or `Resource::collection(...)`
- **Inertia responses** also use Resources for the data payload (since the data is the same; only the response wrapper differs)

## When NOT to use

- For input validation → FormRequest
- For inter-layer transport → DTO
- For computing business values → Action (compute in Action, expose result in Resource)
- For raw exports (CSV, Excel) → dedicated export classes, not Resources

## Conditional output

| Helper | Use |
|---|---|
| `when($condition, $value)` | Include field only if condition is true |
| `whenLoaded('relation')` | Include relation only if eager-loaded |
| `whenCounted('relation')` | Include count only if `withCount()` was used |
| `whenNotNull($value)` | Include only if value is not null |
| `mergeWhen($condition, [...])` | Merge multiple fields conditionally |

## Lifecycle

1. Controller returns `new XResource($model)` or `XResource::collection($items)`
2. Laravel resolves the Resource to a Response
3. `toArray($request)` runs to produce the data payload
4. Laravel wraps in `{ "data": ... }` (configurable)
5. Response serialized to JSON and sent

## What controls it

- **Constructed by:** Controllers
- **Configured by:** `toArray()` method, optional `with()` method for top-level metadata, `additional()` for extra response data
- **Wrapping:** controlled per Resource via `static $wrap = 'order'` or globally via `JsonResource::withoutWrapping()` in a Service Provider

## Examples

### Basic Resource

```php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

final class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'number' => $this->number,
            'customer_name' => $this->customer_name,
            'phone' => $this->phone,
            'status' => $this->status->value,
            'total_amount' => $this->total_amount,
            'created_at' => $this->created_at->toIso8601String(),
            'items' => OrderItemResource::collection($this->whenLoaded('items')),
            'items_count' => $this->whenCounted('items'),
        ];
    }
}
```

### Resource with viewer-aware fields

```php
final class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        $viewer = $request->user();
        $isOwnProfile = $viewer?->id === $this->id;
        $isAdmin = $viewer?->isAdmin() ?? false;

        return [
            'id' => $this->id,
            'name' => $this->name,
            'avatar_url' => $this->avatar_url,

            // Email visible only to the user themselves or admins
            'email' => $this->when($isOwnProfile || $isAdmin, $this->email),

            // Phone visible only to the user themselves
            'phone' => $this->when($isOwnProfile, $this->phone),

            // Admin-only fields
            $this->mergeWhen($isAdmin, [
                'last_login_at' => $this->last_login_at?->toIso8601String(),
                'is_verified' => $this->email_verified_at !== null,
            ]),

            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

### Collection / list response

```php
public function index(Request $request, ListOrdersQuery $query)
{
    $filters = OrderFiltersDto::from($request->query());
    return OrderResource::collection($query->handle($filters));
}
```

Laravel automatically handles pagination metadata when `$query->handle()` returns a `LengthAwarePaginator`:

```json
{
  "data": [...],
  "links": { ... },
  "meta": { "current_page": 1, "total": 47, ... }
}
```

### Resource with computed display values

Formatting is OK; business computation is not.

```php
final class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'number' => $this->number,

            // ✅ formatting — OK
            'created_at_human' => $this->created_at->diffForHumans(),
            'total_formatted' => number_format($this->total_amount / 100, 2) . ' SYP',

            // ❌ would be business logic — DON'T do this here
            // 'total_with_tax' => $this->total_amount * 1.10,
            // (compute in Action, expose pre-computed in Resource)
        ];
    }
}
```

### Wrapping a different key

```php
final class OrderResource extends JsonResource
{
    public static $wrap = 'order';

    public function toArray(Request $request): array { /* ... */ }
}
```

Response: `{ "order": { ... } }` instead of `{ "data": { ... } }`.

### Single Resource used across many endpoints

The same `OrderResource` is used by every endpoint that returns an order:

```php
final class OrderController
{
    public function store(StoreOrderRequest $request, PlaceOrder $action)
    {
        $dto = CreateOrderDto::from($request->validated());
        $order = $action->handle($dto);
        return new OrderResource($order->load('items'));
    }

    public function show(Order $order)
    {
        return new OrderResource($order->load('items'));
    }

    public function update(UpdateOrderRequest $request, Order $order, UpdateOrder $action)
    {
        $dto = UpdateOrderDto::from($request->validated());
        $order = $action->handle($order, $dto);
        return new OrderResource($order->load('items'));
    }

    public function index(Request $request, ListOrdersQuery $query)
    {
        $filters = OrderFiltersDto::from($request->query());
        return OrderResource::collection($query->handle($filters));
    }
}
```

Output shape is consistent across `store`, `show`, `update`, `index` — one Resource defines it for the whole controller.
