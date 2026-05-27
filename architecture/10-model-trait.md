# Model Lifecycle Trait

## Definition

A PHP trait used by Eloquent models to attach **lifecycle hooks** (creating, updated, deleted, etc.). Implements the `boot{TraitName}()` convention, which Laravel automatically calls when the model boots. This is the team's chosen mechanism for model-level plumbing — UUIDs, slugs, audit logs, cascade cleanups.

## Keywords

Plumbing • Lifecycle hook • Model invariant • Trait • Concern

## Purpose

To attach behavior to Eloquent events (`creating`, `updated`, `deleting`, etc.) in a reusable, cross-model way. One trait per concern; models opt-in via `use HasUuid;`.

## Rules

1. One trait per concern: `HasUuid`, `HasOrderNumber`, `Auditable`, `HasSlug`.
2. Located in `app/Concerns/`.
3. Method name follows Laravel's convention: `bootHasUuid()`, `bootAuditable()` etc.
4. **Plumbing only** — no business logic, no external calls, no side effects beyond the model itself.
5. No dependency on the authenticated user, current request, or any contextual state. Traits run *always*, no matter who triggered the model save.
6. Logic should be obvious in 5 lines or less; if not, reconsider whether it belongs as a trait.
7. Traits should not call Actions, Queries, or external Services.

## Shape

```php
namespace App\Concerns;

use Illuminate\Database\Eloquent\Model;

trait Has{Concern}
{
    protected static function bootHas{Concern}(): void
    {
        static::creating(function (Model $model) {
            // plumbing
        });

        static::updated(function (Model $model) {
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

## Parameters

Traits hook into Eloquent events; the model instance is the only "parameter."

## Returns

N/A — hooks are void.

## Is it `final`?

Traits cannot be `final` in PHP (the concept doesn't apply to traits). The classes USING the trait are still `final`.

## When to use

- Generating UUIDs on `creating`
- Generating slugs on `creating`
- Generating ordered numbers (`ORD-2026-00001`)
- Audit logging on `updated` (storing diffs in an audit table)
- Cascade soft-deletes to related records
- Setting "created_by" / "updated_by" foreign keys (provided the context is reliable — usually NOT a good fit for traits)

## When NOT to use

- **Sending notifications** (SMS, email) — that's a side effect of a business operation, not a model invariant
- **External API calls** — too unreliable for lifecycle hooks
- **Business decisions** ("if total > X, mark VIP") — belongs in Actions
- **Context-dependent logic** (different behavior per tenant, per user role) — traits run regardless of context
- **Logic that must be skippable** in some scenarios — observers/traits always run

## Lifecycle

1. Model class loads
2. Laravel calls `boot{TraitName}()` once per request (cached after)
3. Hooks register with the model's event dispatcher
4. When the model triggers an event (`creating`, `updated`, etc.), registered hooks run
5. `initialize{TraitName}()` runs on every model instance construction (rare; usually skipped)

## What controls it

- **Triggered by:** Eloquent events fired during model lifecycle
- **Registered by:** `use HasUuid;` on the model class
- **Configured by:** the trait code itself; no external configuration

## Examples

### UUID generation

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

    public function initializeHasUuid(): void
    {
        $this->mergeCasts(['uuid' => 'string']);
    }
}
```

Used by:

```php
final class Order extends Model
{
    use HasUuid;
}
```

### Audit logging

```php
namespace App\Concerns;

use Illuminate\Database\Eloquent\Model;

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

        static::deleted(function (Model $model) {
            $model->audits()->create([
                'changes' => ['__deleted__' => true],
                'original' => $model->getOriginal(),
                'changed_at' => now(),
            ]);
        });
    }
}
```

### Slug generation

```php
namespace App\Concerns;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

trait HasSlug
{
    protected static function bootHasSlug(): void
    {
        static::creating(function (Model $model) {
            $model->slug ??= Str::slug($model->{$model->slugSource()});
        });
    }

    public function slugSource(): string
    {
        return 'name';
    }
}
```

## Anti-examples

### ❌ Business logic in a trait

```php
trait NotifiesOnOrder
{
    protected static function bootNotifiesOnOrder(): void
    {
        static::created(function (Model $order) {
            Sms::send($order->phone, 'Order placed');   // ❌
            Mail::send(new OrderConfirmation($order));  // ❌
        });
    }
}
```

This sends SMS and email *every* time an Order is created — including factories in tests, seeders, console commands, retries. Side effects belong in Actions and Listeners.

### ❌ External API calls

```php
trait SyncToErp
{
    protected static function bootSyncToErp(): void
    {
        static::updated(function (Model $model) {
            Http::post('https://erp.example.com/sync', $model->toArray());  // ❌
        });
    }
}
```

External HTTP inside a lifecycle hook means the API call happens on every test, every seeder, every retry. And if it fails, the whole save fails. Dispatch a Job from an Action instead.

### ❌ Context-dependent logic

```php
trait TracksCreator
{
    protected static function bootTracksCreator(): void
    {
        static::creating(function (Model $model) {
            $model->created_by = auth()->id();   // ❌ context-dependent
        });
    }
}
```

Fails in console commands, queue jobs, factories — anywhere `auth()` is null. Pass the creator into the Action explicitly.

### ❌ One mega-trait

```php
trait HasOrderLifecycle
{
    protected static function bootHasOrderLifecycle(): void
    {
        static::creating(function ($m) { /* uuid, number, slug, audit */ });
        static::updating(function ($m) { /* audit, sync, notify */ });
        static::deleting(function ($m) { /* cleanup, archive */ });
    }
}
```

Multiple concerns crammed into one trait. Split into `HasUuid`, `HasOrderNumber`, `Auditable`. Reuse across models.
