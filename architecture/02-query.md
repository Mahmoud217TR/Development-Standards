# Query

## Definition

A class representing one read operation that is too complex or too important to inline. Queries fetch and shape data from one or more sources (usually Eloquent, sometimes search indexes or external APIs) and return it. They are the **top-level read path** of the application.

## Keywords

Read • Fetch • Lookup • Projection • View-model • Report

## Purpose

To localize complex read logic — filtering, aggregation, joins, search — into named, testable units. Prevents controllers from accumulating query-building code and enables the same read logic to be reused from multiple entry points (HTTP, console, jobs).

## Rules

1. One Query per read operation. Named after what it returns or who it serves.
2. Exactly one public method: `handle()`.
3. Dependencies via constructor injection only.
4. **Read-only.** Never writes to the database or triggers side effects.
5. Returns typed results: a Collection, a Paginator, a Data class, or a primitive count.
6. Never fires events.
7. **`final` by default.**

## Shape

```php
final class {Action}{Object}Query
{
    public function __construct(/* injected dependencies */) {}

    public function handle(/* typed inputs */): /* return type */
    {
        return /* the read */;
    }
}
```

## Parameters

- **A filter Data class** (preferred) for parameterized reads
- **Domain models** for scoped reads (`User $user` for "their orders")
- **Primitives** for simple lookups (`int $userId`, `string $term`)

Never accept `Request` directly.

## Returns

- `Illuminate\Database\Eloquent\Collection` for lists
- `Illuminate\Contracts\Pagination\LengthAwarePaginator` for paginated lists
- A Data class for composite reads (dashboard stats, projections)
- A Model for single-record reads (only when business rules are involved; otherwise use `Model::find()` directly)
- An `int` or `bool` for aggregate/lookup queries

## Is it `final`?

**Yes. Always.**

## When to use

- The read has 3+ filter parameters
- The same read logic is needed from multiple places (HTTP + export job, multiple controllers)
- The read involves multiple joins, subqueries, or aggregation
- The read includes scoping for security (e.g., "only this merchant's orders") that must be reused
- The read hits a non-Eloquent source (search index, external API)
- The result is a projection — a Data class assembled from multiple sources

## When NOT to use

- **Trivial lookups** (`Model::find($id)`, `User::where('email', $email)->first()`) — inline in controller
- **One-off reads** used in exactly one place with no complexity — inline
- **Writing data** — that's an Action
- **Side effects** — Queries never have side effects
- **Caching layer** alone — caching is a concern that wraps queries; not a reason to create one

## Lifecycle

1. Container resolves the Query when declared as a controller/action/job parameter
2. Constructor runs, dependencies injected
3. Caller invokes `handle(...)` with typed inputs
4. Query runs synchronously
5. Returns; container discards the instance

## What controls it

- **Triggered by:** controllers, Actions, jobs, console commands, listeners
- **Registered by:** Laravel's auto-resolved container
- **Configured by:** filter Data class structure

## Example

```php
namespace App\Queries\Orders;

use App\Data\Orders\OrderFilters;
use App\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

final class ListUserOrdersQuery
{
    public function handle(User $user, OrderFilters $filters): LengthAwarePaginator
    {
        return $user->orders()
            ->when($filters->status, fn ($q, $status) => $q->where('status', $status))
            ->when($filters->search, fn ($q, $search) => $q->where(function ($q) use ($search) {
                $q->where('number', 'like', "%{$search}%")
                  ->orWhere('customer_name', 'like', "%{$search}%");
            }))
            ->when($filters->date_from, fn ($q, $from) => $q->where('created_at', '>=', $from))
            ->when($filters->date_to, fn ($q, $to) => $q->where('created_at', '<=', $to))
            ->with(['items'])
            ->latest()
            ->paginate($filters->per_page ?? 20);
    }
}
```

### Aggregation/projection Query

```php
final class UserDashboardStatsQuery
{
    public function handle(User $user, Carbon $from, Carbon $to): DashboardStatsData
    {
        $orders = $user->orders()->whereBetween('created_at', [$from, $to]);

        return new DashboardStatsData(
            total_orders: $orders->count(),
            pending_orders: (clone $orders)->where('status', 'pending')->count(),
            total_revenue: (clone $orders)->sum('total_amount'),
        );
    }
}
```

### Search Query (calls a Service)

```php
final class SearchProductsQuery
{
    public function __construct(private SearchService $search) {}

    public function handle(string $term, ProductFilters $filters): Collection
    {
        $hits = $this->search->index('products')->search($term, [
            'filter' => $filters->toSearchFilter(),
        ]);

        return Product::whereIn('id', $hits->ids())->get();
    }
}
```

## Anti-examples

### ❌ Query with side effects

```php
final class GetUserAndLogAccessQuery
{
    public function handle(int $userId): User
    {
        $user = User::findOrFail($userId);
        AccessLog::create(['user_id' => $userId]);  // side effect
        return $user;
    }
}
```

Queries are pure reads. The logging belongs in middleware, an Action, or an Observer.

### ❌ Query using `auth()` internally

```php
public function handle(OrderFilters $filters): LengthAwarePaginator
{
    return auth()->user()->orders()->paginate();  // hidden dependency
}
```

The Query secretly depends on the authenticated user. It can't be used from a job or console command. Pass `User $user` as a parameter.

### ❌ Query that's a glorified wrapper

```php
final class GetOrderByIdQuery
{
    public function handle(int $id): Order
    {
        return Order::findOrFail($id);
    }
}
```

Zero value over `Order::findOrFail($id)`. Inline it.
