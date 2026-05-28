# Model Lifecycle Hooks

## Definition

Code attached to Eloquent model events — `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restoring`, `restored`. Used for model-level plumbing: generating UUIDs, slugs, ordered numbers, audit logging, cascade cleanups. Two mechanisms are allowed; the choice depends on scope.

## Keywords

Plumbing • Lifecycle hook • Model invariant • Boot • Trait • Concern

## Two mechanisms

### Mechanism 1 (primary) — `boot()` / `booted()` directly in the model

Lives in the model class itself. Used for model-specific hooks.

```php
final class Order extends Model
{
    protected static function booted(): void
    {
        static::creating(function (Order $order) {
            $order->uuid ??= (string) Str::uuid();
            $order->number ??= app(OrderNumberGenerator::class)->next();
        });
    }
}
```

### Mechanism 2 (secondary) — Trait with `boot{Name}()` in `app/Concerns/`

Used only for **cross-cutting concerns** that genuinely apply to multiple models. Avoids copy-paste across many models.

```php
namespace App\Concerns;

trait HasUuid
{
    protected static function bootHasUuid(): void
    {
        static::creating(function ($model) {
            $model->uuid ??= (string) Str::uuid();
        });
    }
}
```

Used by:

```php
final class Order extends Model { use HasUuid; }
final class Product extends Model { use HasUuid; }
final class Customer extends Model { use HasUuid; }
```

## When to use which

| Situation | Mechanism |
|---|---|
| Hook is specific to ONE model | `boot()` / `booted()` in the model |
| Hook is shared by 3+ models | Trait in `app/Concerns/` |
| One-off behavior (audit on a specific model) | `boot()` / `booted()` in the model |
| Generic invariants used across many entities (UUIDs) | Trait |

The decision rule: **start with `boot()` / `booted()` in the model**. Promote to a trait only when the third model needs the same hook.

## Rules

1. Hooks do **plumbing only**: UUIDs, slugs, ordered numbers, audit logs, cascade cleanups, model invariants.
2. **No business logic, side effects, or external calls** in hooks. No sending SMS, no HTTP calls, no event firing.
3. Hooks should not depend on the authenticated user, current request, or any contextual state. They run *always*, regardless of who triggered the save.
4. Hooks should be obvious in 5 lines or less.
5. Hooks should not call Actions, Queries, or external Services.
6. Observers are **not used** in this standard.
7. `app()` / `resolve()` is permitted inside `boot{Name}()` static closures (no DI available there).

## Shape

### In the model

```php
final class {Model} extends Model
{
    protected static function booted(): void
    {
        static::creating(function ({Model} $model) {
            // plumbing
        });

        static::updated(function ({Model} $model) {
            // plumbing
        });
    }
}
```

Note: `booted()` runs once when the model class boots; the static event hooks register inside it. Use `booted()` over the legacy `boot()` unless you need to call `parent::boot()` explicitly.

### In a trait

```php
namespace App\Concerns;

trait Has{Concern}
{
    protected static function bootHas{Concern}(): void
    {
        static::creating(function ($model) {
            // plumbing
        });
    }

    public function initializeHas{Concern}(): void
    {
        // optional: runs in model constructor, before booting
        // e.g., add a field to $fillable or $casts dynamically
    }
}
```

## When to use

Lifecycle hooks are the right tool for:
- Generating UUIDs on `creating`
- Generating slugs on `creating`
- Generating ordered numbers (`ORD-2026-00001`)
- Audit logging on `updated` (storing diffs in an audit table)
- Cascade soft-deletes to related records
- Computing derived fields that depend only on the model's own data

## When NOT to use

- **Sending notifications** (SMS, email) — that's a side effect of a business operation, not a model invariant
- **External API calls** — too unreliable for lifecycle hooks
- **Business decisions** ("if total > X, mark VIP") — belongs in Actions
- **Context-dependent logic** (different behavior per tenant, per user role) — hooks run regardless of context
- **Logic that must be skippable** in some scenarios — hooks always run

## Lifecycle

### `boot()` / `booted()` in model
1. Model class loads (first time per request)
2. Laravel calls `booted()` once
3. Static event hooks register with the model's event dispatcher
4. When the model triggers an event, registered hooks run

### Trait `boot{Name}()`
1. Model class with the trait loads
2. Laravel calls `boot{Name}()` once per request (cached after)
3. Hooks register
4. `initialize{Name}()` runs on every model instance construction

## Examples

### UUID generation as a trait (used across many models)

```php
namespace App\Concerns;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

trait HasUuid
{
    protected static function bootHasUuid(): void
    {
        static::creating(function (Model $model) {
            $model->uuid ??= (string) Str::uuid();
        });
    }
}
```

### Order number generation as a model-specific hook

```php
final class Order extends Model
{
    use HasUuid;

    protected static function booted(): void
    {
        static::creating(function (Order $order) {
            $order->number ??= app(OrderNumberGenerator::class)->next();
        });
    }
}
```

### Audit logging as a trait

```php
namespace App\Concerns;

trait Auditable
{
    protected static function bootAuditable(): void
    {
        static::updated(function (Model $model) {
            $changes = $model->getChanges();
            $original = array_intersect_key($model->getOriginal(), $changes);

            $model->audits()->create([
                'changes' => $changes,
                'original' => $original,
                'changed_at' => now(),
            ]);
        });
    }
}
```

### Cascade soft-delete as a model-specific hook

```php
final class Order extends Model
{
    use SoftDeletes;

    protected static function booted(): void
    {
        static::deleted(function (Order $order) {
            $order->items()->delete();
            $order->shipments()->delete();
        });
    }
}
```
