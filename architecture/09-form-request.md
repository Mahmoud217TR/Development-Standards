# Form Request

## Definition

A Laravel class that handles **authorization** and **validation** for a specific HTTP endpoint. FormRequests are the gatekeepers of write endpoints — they decide whether the user is allowed to perform the operation and whether the input is well-formed before the controller runs.

## Keywords

Validation • Authorization • Endpoint gate • Input contract

## Purpose

To centralize per-endpoint input rules and access control in one declarative class. Controllers stay thin; validation and auth never leak into Actions; rules are reusable, testable, and discoverable.

## Rules

1. **Every write endpoint has a FormRequest.**
2. `authorize()` returns `true` if the request is allowed, `false` otherwise. Cannot perform side effects.
3. `rules()` returns validation rules. Mandatory for write endpoints.
4. `messages()` returns custom error messages (optional, but encouraged for user-facing endpoints).
5. `prepareForValidation()` may transform inputs before validation (e.g., trim, normalize phone format).
6. `withValidator()` may attach after-validation hooks for cross-field rules that don't fit standard rules.
7. No business logic in FormRequests. Auth check is allowed to query the DB/policies; it must remain a yes/no decision with no state changes.
8. **`final` by default.**
9. Naming: `{Verb}{Object}Request` — `StoreOrderRequest`, `UpdateOrderRequest`, `UpdateUserEmailRequest`, `DestroyOrderRequest`.

## Shape

```php
use Illuminate\Foundation\Http\FormRequest;

final class {Verb}{Object}Request extends FormRequest
{
    public function authorize(): bool
    {
        return /* permission check */;
    }

    public function rules(): array
    {
        return [
            /* validation rules */
        ];
    }

    public function messages(): array
    {
        return [
            /* custom messages */
        ];
    }
}
```

## Parameters

N/A — FormRequests are injected by Laravel, not called directly.

## Returns

- `authorize()` returns `bool`. `false` → 403.
- `rules()` returns `array`. Validation failures → 422.
- `validated()` (called by the controller) returns the clean validated array.

## Is it `final`?

**Yes.**

## When to use

- Any HTTP write endpoint (POST, PUT, PATCH, DELETE)
- Any read endpoint with restricted access (admin-only listings, for example) — though many read endpoints don't need a FormRequest
- Endpoints where input shape and access policy are both worth naming

## When NOT to use

- For routing or HTTP semantics — that's controller territory
- For business logic — that's an Action's job
- For output formatting — that's a Resource
- Inside Actions, Queries, Jobs — those don't deal with HTTP

## Lifecycle

1. Request matches route
2. Controller method is resolved
3. FormRequest parameter triggers Laravel to construct it
4. `prepareForValidation()` runs (optional input transformation)
5. `authorize()` runs → 403 response if `false`
6. `rules()` evaluated against the input → 422 response if any rule fails
7. `withValidator()` hooks run (cross-field validation)
8. Controller method receives the FormRequest with validated data accessible via `$request->validated()`

## What controls it

- **Triggered by:** controller method declaring it as a parameter
- **Configured by:** overrides of `authorize()`, `rules()`, `messages()`, `prepareForValidation()`, `withValidator()`
- **May delegate to:** Laravel Policies (`$this->user()->can('update', $this->route('order'))`)

## Examples

### Standard write FormRequest

```php
namespace App\Http\Requests\Orders;

use App\Models\Order\Order;
use Illuminate\Foundation\Http\FormRequest;

final class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('create', Order::class) ?? false;
    }

    public function rules(): array
    {
        return [
            'customer_name' => ['required', 'string', 'max:100'],
            'phone' => ['required', 'string', 'regex:/^09\d{8}$/'],
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'integer', 'exists:products,id'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
            'notes' => ['nullable', 'string', 'max:500'],
        ];
    }

    public function messages(): array
    {
        return [
            'phone.regex' => 'Phone must start with 09 and be 10 digits.',
            'items.required' => 'Order must contain at least one item.',
        ];
    }
}
```

### Per-field-group update FormRequest

```php
namespace App\Http\Requests\Users;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

final class UpdateUserEmailRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user() !== null;
    }

    public function rules(): array
    {
        return [
            'email' => [
                'required',
                'email',
                'max:255',
                Rule::unique('users')->ignore($this->user()->id),
            ],
            'current_password' => ['required', 'current_password'],
        ];
    }
}
```

### FormRequest with prepared input

```php
final class StoreOrderRequest extends FormRequest
{
    protected function prepareForValidation(): void
    {
        $this->merge([
            'phone' => str_replace([' ', '-'], '', $this->phone),
            'customer_name' => trim($this->customer_name),
        ]);
    }

    public function authorize(): bool { /* ... */ }
    public function rules(): array { /* ... */ }
}
```

### FormRequest with cross-field validation

```php
final class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'items' => ['required', 'array', 'min:1'],
            // ...
        ];
    }

    public function withValidator($validator): void
    {
        $validator->after(function ($validator) {
            $totalQuantity = collect($this->items ?? [])->sum('quantity');
            if ($totalQuantity > 100) {
                $validator->errors()->add('items', 'Total quantity exceeds order limit of 100 items.');
            }
        });
    }
}
```

### Authorization via Policy with route model

```php
final class UpdateOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('order'));
    }

    public function rules(): array { /* ... */ }
}
```

### Used with a DTO and Action in a controller

```php
public function store(StoreOrderRequest $request, PlaceOrder $action)
{
    $dto = CreateOrderDto::from($request->validated());
    $order = $action->handle($dto);
    return new OrderResource($order);
}
```

Order of operations:
1. `StoreOrderRequest::authorize()` → 403 if not allowed
2. `StoreOrderRequest` validation runs → 422 if invalid
3. `$request->validated()` returns the clean array
4. `CreateOrderDto::from(...)` produces a typed DTO
5. Action runs with the DTO
6. Controller wraps the result in a Resource
