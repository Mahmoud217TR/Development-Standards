# Model

## Definition

An Eloquent class representing a **persistent domain entity**. Models define table mapping, relationships, casts, accessors/mutators, scopes, and lifecycle hooks (`boot()` / `booted()`). In this team's convention, Models hold **data shape, relational structure, and model-level invariants** — but not business logic.

## Keywords

Entity • Domain object • Eloquent model • Persistent record

## Purpose

To represent persistent entities in the database with type-safe access, relationships, and data transformations (casts). Models are *what an entity is* and *what is intrinsically true about it on save*, not *what the system does with it*.

## Rules

1. Models contain only:
   - Table configuration (`$table`, `$primaryKey`, `$keyType`, `$incrementing`)
   - Fillable / guarded (prefer fillable explicitly)
   - Casts (`$casts` array or property)
   - Relationships (`hasMany`, `belongsTo`, etc.)
   - Accessors / mutators (for derived attributes)
   - Scopes (for reusable query constraints)
   - Trait usage (`use HasUuid, HasFactory;`)
   - **`boot()` / `booted()` methods** for model-specific lifecycle hooks (UUID generation, audit logging, cascade cleanups)
2. Models do **NOT** contain:
   - Business logic (decisions about what to do)
   - External API calls
   - Side effects beyond the model itself (no SMS, email, webhooks, event firing)
3. **`final` by default** unless a specific package requires extension.
4. Located in `app/Models/{Model}/` (each model that has state classes etc. gets its own folder) OR `app/Models/` directly for simple models.
5. Casts use enum classes and Data classes where appropriate.

## Shape

```php
namespace App\Models\{Model};

use App\Concerns\HasUuid;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

final class {Model} extends Model
{
    use HasFactory, HasUuid;

    protected $fillable = [/* explicit list */];

    protected $casts = [
        /* type casts */
    ];

    protected static function booted(): void
    {
        static::creating(function ({Model} $model) {
            // model-specific plumbing
        });
    }

    // relationships, accessors, scopes
}
```

## Parameters

N/A — Models are persistent representations, instantiated by Eloquent or factories.

## Returns

Method-by-method: relationships return relation objects, accessors return computed values, scopes return query builders.

## Is it `final`?

**Yes** by default. Exceptions:
- A package requires you to extend (rare)
- You have a domain reason for inheritance (very rare, and usually wrong — prefer composition)

## When to use

- Anything persisted in the database
- Anything with relationships to other entities

## When NOT to use

- For ephemeral data → use Data classes
- For business operations → use Actions
- For complex reads → use Queries (which internally use Models)
- For external API responses → use a Data class

## Lifecycle

1. Model class loads → `booted()` runs once (registers static event hooks); traits' `boot{Name}()` methods also run
2. Instance created (via `new Model`, `Model::create()`, factory, or query result)
3. `initialize{TraitName}()` runs for each trait
4. Attributes set; relationships lazy-loaded as accessed
5. On save: lifecycle events fire (`creating`, `created`, etc.); registered hooks run
6. On delete: `deleting`, `deleted` events fire

## What controls it

- **Triggered by:** Actions (writes), Queries (reads), seeders, factories, framework operations
- **Configured by:** properties on the class itself
- **Schema controlled by:** migrations

## Examples

### Standard model with model-specific hooks

```php
namespace App\Models\Order;

use App\Concerns\HasUuid;
use App\Models\Order\States\OrderState;
use App\Models\User;
use App\Services\OrderNumberGenerator;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Spatie\ModelStates\HasStates;

final class Order extends Model
{
    use HasFactory, HasUuid, HasStates;

    protected $fillable = [
        'number',
        'customer_name',
        'phone',
        'notes',
        'total_amount',
        'status',
        'tracking_number',
        'shipped_at',
        'merchant_id',
    ];

    protected $casts = [
        'status' => OrderState::class,
        'total_amount' => 'integer',
        'shipped_at' => 'datetime',
    ];

    protected static function booted(): void
    {
        static::creating(function (Order $order) {
            // Order-specific: generate order number (not shared with other models)
            $order->number ??= app(OrderNumberGenerator::class)->next();
        });

        static::deleted(function (Order $order) {
            // Cascade cleanup
            $order->items()->delete();
        });
    }

    public function merchant(): BelongsTo
    {
        return $this->belongsTo(User::class, 'merchant_id');
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function audits(): HasMany
    {
        return $this->hasMany(OrderAudit::class);
    }

    public function scopeForMerchant($query, User $merchant)
    {
        return $query->where('merchant_id', $merchant->id);
    }
}
```

### Model with accessor (modern attribute style)

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

final class Order extends Model
{
    // ...

    protected function displayNumber(): Attribute
    {
        return Attribute::make(
            get: fn () => "#{$this->number}",
        );
    }
}
```

### Simple model with no folder

```php
namespace App\Models;

use App\Concerns\HasUuid;
use Illuminate\Database\Eloquent\Model;

final class Tag extends Model
{
    use HasUuid;

    protected $fillable = ['name', 'color'];
}
```

Located directly in `app/Models/Tag.php` — no folder needed because there are no state classes or sub-entities to organize.
