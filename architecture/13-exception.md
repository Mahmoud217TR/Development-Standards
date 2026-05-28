# Exception

## Definition

A class representing an error condition. In this standard, exceptions fall into two categories: **domain exceptions** (business-rule violations defined in `app/Exceptions/`) and **infrastructure exceptions** (failures from databases, external APIs, libraries). The team's approach is to throw at the source, translate at boundaries, and catch only at the outermost layer.

## Keywords

Error • Domain exception • Infrastructure exception • Handler • Boundary translation

## Purpose

To make failure modes part of the type system. A function that throws `InsufficientInventoryException` documents its failure cases in a way callers can reason about. Centralizing HTTP translation in the exception handler keeps controllers and Actions free of error-formatting code.

## Two categories

### Domain exceptions

Represent **business-rule violations**. Defined in `app/Exceptions/`. One class per failure type.

```php
namespace App\Exceptions;

class InsufficientInventoryException extends \DomainException
{
    public function __construct(
        public readonly array $unavailableItems,
    ) {
        parent::__construct('Some items are not available in the requested quantities.');
    }
}

class OrderAlreadyShippedException extends \DomainException {}
class PaymentDeclinedException extends \RuntimeException {}
class PaymentGatewayException extends \RuntimeException {}
```

### Infrastructure exceptions

Thrown by databases, external APIs, libraries, the filesystem. Examples: `\Stripe\Exception\ApiErrorException`, `\PDOException`, `Illuminate\Database\QueryException`, `Symfony\Component\HttpClient\Exception\TransportException`.

These should not propagate up to Actions unchanged. **Services catch them and translate to domain exceptions.**

## Rules

### Throwing

- **Throw at the source.** Code that detects a failure should throw immediately, not return null or false.
- Throw a **domain exception** (from `app/Exceptions/`) for business-rule violations.
- Use the framework's built-in exceptions (`ModelNotFoundException`, `ValidationException`, etc.) when they fit; don't invent custom equivalents.

### Catching

- **Catching is the exception, not the norm.** Most code lets exceptions propagate.
- **Services** catch vendor/external exceptions and re-throw as domain exceptions. Never catch and swallow.
- **Actions** catch only when there is an explicit business fallback decision. Narrow `try/catch` only — never `\Exception` or `\Throwable`.
- **Queries, Controllers, Form Requests, Data classes** never catch.
- **Jobs, Listeners** never catch inside `handle()` — rely on `$tries`, `$backoff`, `failed()`.

### Translating to HTTP

- All exception → HTTP response mapping lives in **one place**: `bootstrap/app.php` (Laravel 11+) or `App\Exceptions\Handler` (Laravel 10).
- Controllers never format error responses.
- Domain exceptions get mapped to appropriate status codes (422, 409, 402, etc.).
- Infrastructure exceptions get reported to monitoring and mapped to 500/502/503.

### Naming

- `*Exception` suffix mandatory: `InsufficientInventoryException`, not `InsufficientInventory`.
- Past tense for state-conflict errors: `OrderAlreadyShipped`, not `CannotShipOrder`.
- Imperative for action errors: `PaymentDeclined`, not `PaymentWasDeclined`.

## Shape

```php
namespace App\Exceptions;

class {Subject}{Failure}Exception extends \DomainException
{
    public function __construct(
        public readonly /* context fields */,
        string $message = '',
    ) {
        parent::__construct($message ?: 'Default message');
    }
}
```

## Is it `final`?

**Yes** for domain exceptions. Abstract bases (when you have a hierarchy of related exceptions) use `abstract`.

## Where each layer catches

| Layer | Catches? | What |
|---|---|---|
| Database / external library | N/A — they throw | — |
| Service | Yes | Vendor exceptions → re-throw as domain exceptions |
| Query | No | Let propagate |
| Action | Rare | Only with explicit business fallback |
| Listener / Job `handle()` | No | Queue's retry mechanism handles it |
| Listener / Job `failed()` | Yes | After all retries exhausted; log or notify |
| Controller | No | Let propagate to handler |
| Form Request | No | `authorize()` returns bool, doesn't throw |
| Exception Handler | **Yes** | Catches everything; renders + reports |

## Examples

### Domain exception with context

```php
namespace App\Exceptions;

final class InsufficientInventoryException extends \DomainException
{
    public function __construct(
        public readonly array $unavailableItems,
    ) {
        parent::__construct('Some items are not available in the requested quantities.');
    }
}
```

### Throwing from an Action

```php
final class PlaceOrder
{
    public function handle(CreateOrderDto $dto): Order
    {
        if (!$this->inventory->isAvailable($dto->items)) {
            throw new InsufficientInventoryException(
                $this->inventory->getUnavailable($dto->items)
            );
        }
        // ...
    }
}
```

### Service translating vendor exception

```php
final class StripeService
{
    public function refund(string $chargeId, int $amount): RefundDto
    {
        try {
            $response = $this->client->refunds->create([
                'charge' => $chargeId,
                'amount' => $amount,
            ]);
        } catch (\Stripe\Exception\CardException $e) {
            throw new PaymentDeclinedException($e->getMessage(), previous: $e);
        } catch (\Stripe\Exception\ApiErrorException $e) {
            throw new PaymentGatewayException($e->getMessage(), previous: $e);
        }

        return RefundDto::from($response);
    }
}
```

### Action with explicit fallback (narrow try/catch)

```php
final class PlaceOrder
{
    public function handle(CreateOrderDto $dto): Order
    {
        $order = DB::transaction(fn () => /* create order */);

        try {
            $this->shipping->createShipment($order);
        } catch (ShippingProviderUnavailableException $e) {
            $order->update(['shipping_status' => 'pending']);
            report($e);
        }

        event(new OrderPlaced($order));

        return $order;
    }
}
```

### Centralized handler (Laravel 11+)

```php
// bootstrap/app.php

use App\Exceptions\InsufficientInventoryException;
use App\Exceptions\OrderAlreadyShippedException;
use App\Exceptions\PaymentDeclinedException;
use App\Exceptions\PaymentGatewayException;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(/* ... */)
    ->withMiddleware(function (Middleware $middleware) { /* ... */ })
    ->withExceptions(function (Exceptions $exceptions) {

        // Domain exceptions → 422 with details
        $exceptions->render(function (InsufficientInventoryException $e, Request $request) {
            return response()->json([
                'message' => $e->getMessage(),
                'unavailable_items' => $e->unavailableItems,
            ], 422);
        });

        // State conflict → 409
        $exceptions->render(function (OrderAlreadyShippedException $e) {
            return response()->json([
                'message' => 'This order has already been shipped.',
            ], 409);
        });

        // Payment declined → 402
        $exceptions->render(function (PaymentDeclinedException $e) {
            return response()->json([
                'message' => 'Payment was declined.',
            ], 402);
        });

        // Gateway error → 502, report to monitoring
        $exceptions->render(function (PaymentGatewayException $e) {
            return response()->json([
                'message' => 'Payment processing failed. Please try again.',
            ], 502);
        });

        // Don't spam Sentry with business-rule violations
        $exceptions->dontReport([
            InsufficientInventoryException::class,
            OrderAlreadyShippedException::class,
            PaymentDeclinedException::class,
        ]);
    })
    ->create();
```

### Job using `failed()` for final-failure logic

```php
final class SendOrderConfirmationSms implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60;

    public function handle(OrderPlaced $event): void
    {
        $this->sms->send(...);  // if this throws, queue retries
    }

    public function failed(\Throwable $e): void
    {
        // Called after $tries exhausted
        report($e);
        // Optionally: notify admins, mark a flag in DB
    }
}
```
