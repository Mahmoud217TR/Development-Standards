# Model

## Definition

An Eloquent class representing a **persistent domain entity**. Models define table mapping, relationships, casts, accessors/mutators, and scopes. In this team's convention, Models hold **data shape and relational structure** — not business logic.

## Keywords

Entity • Domain object • Eloquent model • Persistent record

## Purpose

To represent persistent entities in the database with type-safe access, relationships, and data transformations (casts). Models are *what an entity is*, not *what the system does with it*.

## Rules

1. Models contain only:
   - Table configuration (`$table`, `$primaryKey`, `$keyType`, `$incrementing`)
   - Fillable / guarded (prefer fillable explicitly)
   - Casts (`$casts` array or property)
   - Relationships (`hasMany`, `belongsTo`, etc.)
   - Accessors / mutators (for derived attributes)
   - Scopes (for reusable query constraints)
   - Trait usage (`use HasUuid, HasFactory;`)
2. Models do **NOT** contain:
   - Business logic
   - External API calls
   - Side effects beyond the model itself
   - Lifecycle hooks (these go in traits)
3. **`final` by default** unless a specific package requires extension.
4. Located in `app/Models/{Model}/` (each model gets its own folder for state classes etc.) OR `app/Models/` directly if no state machine is involved.
5. Casts use enum classes and Data classes where appropriate.

## Shape

```php
namespace App\Models\{Model};

use App\Concerns\HasUuid;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

final class {Model} extends Model
{
    use HasFactory, HasUuid;

    protected $fillable = [/* explicit list */];

    protected $casts = [
        /* type casts */
    ];

    public function {relationship}(): HasMany|BelongsTo|/*...*/
    {
        return $this->{relationType}(/* ... */);
    }
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
- For external API responses → use a DTO or value object

## Lifecycle

1. Model class loads → traits' `boot{Name}()` methods register hooks
2. Instance created (via `new Model`, `Model::create()`, factory, or query result)
3. `initialize{TraitName}()` runs for each trait
4. Attributes set; relationships lazy-loaded as accessed
5. On save: lifecycle events fire (`creating`, `created`, etc.); trait hooks run
6. On delete: `deleting`, `deleted` events fire

## What controls it

- **Triggered by:** Actions (writes), Queries (reads), seeders, factories, framework operations
- **Configured by:** properties on the class itself
- **Schema controlled by:** migrations

## Examples

### Standard model

```php
namespace App\Models\Order;

use App\Concerns\HasUuid;
use App\Models\Order\States\OrderState;
use App\Models\User;
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

### Model with accessor

```php
final class Order extends Model
{
    // ...

    public function getDisplayNumberAttribute(): string
    {
        return "#{$this->number}";
    }
}
```

Modern Laravel attribute style:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

protected function displayNumber(): Attribute
{
    return Attribute::make(
        get: fn () => "#{$this->number}",
    );
}
```

## Anti-examples

### ❌ Business logic in a model

```php
final class Order extends Model
{
    public function place(): void                              // ❌
    {
        $this->save();
        Sms::send($this->phone, 'Order placed');
        event(new OrderPlaced($this));
    }

    public function calculateTotalWithDiscount(): int          // ❌ if it involves business rules
    {
        $discount = $this->customer->loyalty_level === 'gold' ? 0.1 : 0;
        return $this->total - ($this->total * $discount);
    }
}
```

`place()` is an Action. The total calculation involves a business rule (loyalty tier discount) and belongs in an Action or value object. Models hold data; operations live elsewhere.

### ❌ Hooks defined in the model

```php
final class Order extends Model
{
    protected static function booted(): void                   // ❌
    {
        static::creating(function ($order) {
            $order->uuid = Str::uuid();
        });
    }
}
```

Hooks belong in traits (`HasUuid`).

### ❌ Non-explicit fillable

```php
final class Order extends Model
{
    protected $guarded = [];                                   // ❌ unsafe
}
```

Mass assignment vulnerability. Always be explicit with `$fillable`.

### ❌ Models with many query methods

```php
final class Order extends Model
{
    public static function pendingForMerchant(User $m, array $filters): Builder
    {
        // 30 lines of filtering
    }
}
```

Complex queries belong in Query classes. Scopes are fine for simple, reusable constraints; not for whole query operations.
