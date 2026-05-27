# Service

## Definition

A class wrapping a single capability or external system. Services are **low-level building blocks** used by Actions, Queries, Listeners, and Jobs. They encapsulate the details of how a capability works (which API, which library, what configuration) so callers don't have to know.

## Keywords

Capability • Adapter • Gateway • Client • Wrapper • Utility

## Purpose

To isolate the team from external dependencies and library details. When the SMS provider changes, when the payment gateway updates its SDK, when the search backend swaps — only the Service changes. Callers (Actions, etc.) remain stable.

## Rules

1. One Service per capability or external system. `SmsService`, `StripeService`, `PdfRenderer`.
2. Multiple public methods OK if they belong to the same capability.
3. Dependencies via constructor injection only.
4. **No business logic.** A Service does not decide *whether* to send an SMS — it sends one when asked.
5. Services do not fire events. Events are an orchestration concern; orchestration lives in Actions.
6. Services should not call Actions or Queries. Direction of dependencies is downward.
7. **`final` by default**; if it needs a test double, define an interface and have the Service implement it.

## Shape

```php
final class {Subject}{Capability}Service implements {OptionalInterface}
{
    public function __construct(/* injected client / config */) {}

    public function {verb}(/* ... */): /* ... */ {}
    public function {anotherVerb}(/* ... */): /* ... */ {}
}
```

## Parameters

Whatever the capability needs. Typically primitives, Data classes, or domain objects. Services are pragmatic — they take what they need.

## Returns

Whatever the capability produces:
- Primitives (`bool` from `sent()`, `int` from `count()`)
- DTOs representing external responses (`StripeChargeResult`, `SmsDeliveryStatus`)
- Domain entities when the Service produces one (`Pdf::class`, `Invoice::class`)
- `void` for fire-and-forget operations

## Is it `final`?

**Yes**, with the caveat that mockable Services define an interface and the concrete class implements it. Tests mock the interface, not the class.

## When to use

- Wrapping an external API (Stripe, Mailgun, Twilio, Meilisearch)
- Wrapping a library that has complex setup (PDF, image processing)
- Encapsulating a piece of infrastructure (filesystem, cache, queue config)
- Encapsulating an internal capability used across many Actions (`OrderNumberGenerator`, `MoneyFormatter`)

## When NOT to use

- **A use case** — that's an Action
- **A read** — that's a Query (a Query may *use* a Service to talk to a search engine)
- **A model** — domain entities are Models, not Services
- **Pure functions** — if it's stateless, no dependencies, and trivial, a helper or static method on a value object may suffice
- **A "logic dumping ground"** named `OrderService` with 15 unrelated methods — that's the anti-pattern Actions exist to solve

## Lifecycle

1. Container resolves the Service when injected
2. Construction may set up clients, config, connections
3. Methods called as needed
4. Container caches the instance if bound as a singleton (recommended for Services that hold expensive resources)

## What controls it

- **Triggered by:** Actions, Queries, Listeners, Jobs, console commands
- **Registered by:** Laravel's container; bind interface to implementation in a Service Provider when using interfaces
- **Configured by:** constructor (usually from config files via `config()` helper or injected `Repository`)

## Examples

### External API wrapper

```php
namespace App\Services;

final class SmsService implements SmsServiceContract
{
    public function __construct(
        private TwilioClient $client,
        private string $fromNumber,
    ) {}

    public function send(string $to, string $message): SmsDeliveryStatus
    {
        $response = $this->client->messages->create($to, [
            'from' => $this->fromNumber,
            'body' => $message,
        ]);

        return new SmsDeliveryStatus(
            id: $response->sid,
            status: $response->status,
        );
    }

    public function getBalance(): int
    {
        return $this->client->account->balance;
    }
}
```

Provider binding:

```php
$this->app->bind(SmsServiceContract::class, function () {
    return new SmsService(
        client: new TwilioClient(config('services.twilio.key')),
        fromNumber: config('services.twilio.from'),
    );
});
```

### Internal capability

```php
final class OrderNumberGenerator
{
    public function next(): string
    {
        $year = now()->year;
        $sequence = OrderSequence::increment($year);
        return sprintf('ORD-%d-%05d', $year, $sequence);
    }
}
```

## Anti-examples

### ❌ Service containing business logic

```php
final class OrderService
{
    public function place(array $data): Order
    {
        // checks inventory, calculates total, creates order, fires event
    }
}
```

This is a misnamed Action. Each business operation should be its own Action class.

### ❌ Service calling an Action

```php
final class StripeService
{
    public function refund(string $chargeId): void
    {
        $result = $this->stripe->refunds->create([...]);
        app(MarkOrderAsRefunded::class)->handle($result);  // ❌ Service calling Action
    }
}
```

This reverses the dependency direction. The caller should be an Action that uses `StripeService::refund()` and then orchestrates the rest.

### ❌ Service that needs to be mocked but isn't `final`

```php
class SmsService { /* ... */ }  // not final
```

If you need a test double, define `SmsServiceContract` and mock the interface. Keep the class final.
