# DTO / Data Class

## Definition

A typed, immutable structure that carries data between layers. Built on `spatie/laravel-data`. Data classes serve as **input validators**, **output serializers**, **inter-layer DTOs**, and **TypeScript type sources** — all from one definition.

## Keywords

Data Transfer Object • DTO • Payload • Schema • Typed structure • Contract

## Purpose

To replace ad-hoc arrays passed between controllers, Actions, Queries, and frontend with explicit, typed contracts. A Data class is a single declaration of "this is the shape of this thing" that the type system can verify on both PHP and TypeScript sides.

## Rules

1. One Data class per shape. Don't reuse `OrderData` for both creation input and read output if they differ.
2. Naming:
   - `Create{X}Data` — input for creation
   - `Update{X}Data` — input for partial updates (uses `Optional`)
   - `{X}Data` — output / inter-layer
   - `{X}Filters` — filter parameters for Queries
3. Validation lives in attributes on the constructor properties.
4. **`#[TypeScript]` attribute** on every Data class meant to cross the PHP/JS boundary.
5. **`final` by default.**
6. No business logic methods. Data classes hold data, not behavior.
7. Hidden/internal fields use `#[Hidden]` for output control.

## Shape

```php
use Spatie\LaravelData\Data;
use Spatie\TypeScriptTransformer\Attributes\TypeScript;

#[TypeScript]
final class {X}Data extends Data
{
    public function __construct(
        // typed properties with validation attributes
        public string $field,
        public ?int $nullable_field,
    ) {}
}
```

## Parameters

The constructor IS the schema. Each parameter is a property of the data structure.

## Returns

N/A — Data classes are themselves the result. They serialize to JSON automatically when returned from controllers.

## Is it `final`?

**Yes. Always.**

## When to use

- **Validating HTTP input** → controller injects a Data class; Laravel auto-validates from the Request
- **Serializing HTTP output** → return `XData::from($model)` from a controller
- **Passing structured data between Actions, Services, Queries** → typed instead of arrays
- **Generating TypeScript types** for the frontend
- **Defining filter parameters** for Queries
- **Defining payloads** for Events and Jobs

## When NOT to use

- **Wrapping a single primitive** — just use the primitive (`int $userId`, not `UserIdData`)
- **Replacing a domain Model** — Models are persistent entities, Data classes are transient payloads
- **As a class with methods** — Data classes hold data; logic belongs in Actions or Services

## Lifecycle

### Input flow
1. HTTP Request arrives
2. Controller method declares `Create{X}Data $data` parameter
3. `spatie/laravel-data` resolves the parameter from the Request
4. Validation runs (from attributes or `rules()` method)
5. If invalid → 422 response with errors, controller never runs
6. If valid → typed Data instance passed to controller method
7. Controller hands it to an Action

### Output flow
1. Action/Query returns a Model or Collection
2. Controller calls `{X}Data::from($model)`
3. Properties extracted from the model into typed fields
4. Returned from controller → automatically JSON-serialized

### Inter-layer flow
1. Action receives input as a Data class
2. May construct intermediate Data classes for sub-operations
3. Returns a result Data class (or a Model)

## What controls it

- **Triggered by:** controllers (injection), Actions (manual construction)
- **Configured by:** `config/data.php` for defaults (date format, wrapping, etc.)
- **TypeScript generation:** `php artisan typescript:transform` reads `#[TypeScript]` attributes and writes a `.d.ts` file

## Capabilities Reference

| Capability | How |
|---|---|
| Validation from attributes | `#[Required]`, `#[StringType]`, `#[Max(100)]` on properties |
| Validation from method | `public static function rules(): array` |
| Custom messages | `public static function messages(): array` |
| Authorization | `public static function authorize(Request $r): bool` (we use Form Requests instead) |
| Casting | Built-in for Carbon, enums, nested Data; custom via `#[WithCast(...)]` |
| Nested Data | Type-hint as another Data class |
| Collections | `DataCollection` or `array<X>` with `#[DataCollectionOf(X::class)]` |
| Partial updates | Use `Type\|Optional` for fields that may be absent |
| Hidden fields | `#[Hidden]` excludes from output |
| Renamed output | `#[MapOutputName('camelCase')]` |
| Computed properties | Set in constructor body after parent properties |
| Wrapping | `Data::setDefaultWrap('data')` or per-response `->wrap('order')` |
| Pagination | `Data::collect($paginator, PaginatedDataCollection::class)` |
| TypeScript output | `#[TypeScript]` attribute |

## Examples

### Input Data class with validation

```php
namespace App\Data\Orders;

use Spatie\LaravelData\Data;
use Spatie\LaravelData\Attributes\Validation\{Required, StringType, Max, Regex, Min, ArrayType};
use Spatie\LaravelData\Attributes\DataCollectionOf;
use Spatie\LaravelData\DataCollection;
use Spatie\TypeScriptTransformer\Attributes\TypeScript;

#[TypeScript]
final class CreateOrderData extends Data
{
    public function __construct(
        #[Required, StringType, Max(100)]
        public string $customer_name,

        #[Required, Regex('/^09\d{8}$/')]
        public string $phone,

        #[Required, ArrayType, Min(1)]
        #[DataCollectionOf(OrderItemData::class)]
        public DataCollection $items,

        public ?string $notes = null,
    ) {}
}
```

### Output Data class

```php
#[TypeScript]
final class OrderData extends Data
{
    public function __construct(
        public string $id,
        public string $number,
        public string $customer_name,
        public string $phone,
        public OrderStatus $status,        // backed enum
        public Carbon $created_at,
        public int $total_amount,
        /** @var array<OrderItemData> */
        public array $items,

        #[Hidden]
        public ?string $internal_notes = null,  // never in JSON output
    ) {}
}
```

### Filter Data class for Queries

```php
#[TypeScript]
final class OrderFilters extends Data
{
    public function __construct(
        public ?string $status = null,
        public ?string $search = null,
        public ?Carbon $date_from = null,
        public ?Carbon $date_to = null,
        public ?string $sort = null,
        public ?string $direction = 'desc',
        public int $per_page = 20,
    ) {}
}
```

### Partial update with `Optional`

```php
use Spatie\LaravelData\Optional;

#[TypeScript]
final class UpdateOrderData extends Data
{
    public function __construct(
        public string|Optional $customer_name,
        public string|Optional $phone,
        public ?string|Optional $notes,
    ) {}
}
```

Only fields actually sent in the request are present; `Optional` ones are skipped on `toArray()`.

### TypeScript generation

Running `php artisan typescript:transform` produces:

```typescript
export type CreateOrderData = {
    customer_name: string;
    phone: string;
    items: Array<OrderItemData>;
    notes: string | null;
};

export type OrderData = {
    id: string;
    number: string;
    customer_name: string;
    phone: string;
    status: 'pending' | 'shipped' | 'delivered' | 'cancelled';
    created_at: string;
    total_amount: number;
    items: Array<OrderItemData>;
};

export type OrderFilters = {
    status: string | null;
    search: string | null;
    date_from: string | null;
    date_to: string | null;
    sort: string | null;
    direction: string | null;
    per_page: number;
};
```

Frontend (Vue/Inertia) imports these for guaranteed type alignment.

## Anti-examples

### ❌ Logic methods on Data classes

```php
final class OrderData extends Data
{
    public function totalWithTax(): int
    {
        return $this->total + $this->total * 0.1;
    }
}
```

Data classes carry data. Calculations belong in Actions or value objects.

### ❌ Same Data class for input and output

```php
final class OrderData extends Data
{
    public function __construct(
        public string $customer_name,
        public string $phone,
        // ... has both input fields AND output-only fields like `id`, `number`, `created_at`
    ) {}
}
```

Separate input (`CreateOrderData`) from output (`OrderData`). Their shapes are different and merging them creates confusion.

### ❌ Bypassing the Data class in controllers

```php
public function store(Request $request, PlaceOrder $action)
{
    $action->handle($request->all());  // raw array
}
```

Defeats the whole point. Inject the Data class.
