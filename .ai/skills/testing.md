# Testing Skill

You are writing tests in a Laravel codebase that uses **Pest**, **real Postgres**, and a strict **architecture-driven testing philosophy**. Every test you write must conform to these rules.

When in doubt: test observable behavior, never implementation details.

---

## Framework & stack

| Tool | Use |
|---|---|
| **Pest** | Test framework — use `it()`, `describe()`, `expect()` |
| **Postgres** | Real database in CI and locally; same engine as production |
| **`RefreshDatabase`** | Transaction-per-test rollback (default for all tests) |
| **`Http::fake()`** | For Services using Laravel's HTTP client |
| **`Event::fake()`, `Queue::fake()`, `Bus::fake()`, `Mail::fake()`, `Notification::fake()`, `Storage::fake()`** | Laravel built-in fakes |
| **Hand-written `Fake*` classes** | In `tests/Fakes/` for Services wrapping vendor SDKs |
| **Mockery** | Avoided — final classes block it; use interfaces + fakes instead |

---

## What gets tested where

| Component | Test type | Location |
|---|---|---|
| Controllers | Feature | `tests/Feature/{Domain}/` |
| FormRequests | Feature (via controller) | (same as above) |
| JsonResources | Feature (via response shape) | (same as above) |
| Actions | Unit | `tests/Unit/Actions/{Domain}/` |
| Queries | Unit | `tests/Unit/Queries/{Domain}/` |
| Services (HTTP-based) | Unit with `Http::fake()` | `tests/Unit/Services/` |
| Services (SDK-based) | Unit with interface + `Fake*` | `tests/Unit/Services/` |
| Listeners | Unit (`handle()`) + Feature (`Event::fake()` dispatch) | `tests/Unit/Listeners/` + Feature |
| Jobs | Unit (`handle()`) + Feature (`Queue::fake()` dispatch) | `tests/Unit/Jobs/` + Feature |
| State transitions | Unit | `tests/Unit/States/{Model}/` |
| Model `boot()` hooks | Implicit (tested through feature/unit tests touching those models) | — |
| DTOs | NOT tested in isolation | — (constructing one and reading it back tests PHP itself) |
| Domain exceptions | NOT tested in isolation | (asserted on in Action tests via `->toThrow(...)`) |
| Model traits | Implicit | — |

---

## Folder structure

```
tests/
├── Pest.php                    # global hooks, helpers
├── TestCase.php                # extends Laravel's TestCase
├── CreatesApplication.php
├── Feature/
│   ├── Auth/
│   │   ├── LoginTest.php
│   │   └── RegisterTest.php
│   ├── Orders/
│   │   ├── PlaceOrderTest.php
│   │   ├── ListOrdersTest.php
│   │   ├── ShowOrderTest.php
│   │   └── CancelOrderTest.php
│   └── Profile/
│       ├── UpdateNameTest.php
│       └── UpdateEmailTest.php
├── Unit/
│   ├── Actions/
│   │   └── Orders/
│   │       ├── PlaceOrderActionTest.php
│   │       └── CancelOrderActionTest.php
│   ├── Queries/
│   │   └── Orders/
│   │       └── ListUserOrdersQueryTest.php
│   ├── Services/
│   │   ├── OrderNumberGeneratorServiceTest.php
│   │   └── StripeServiceTest.php
│   ├── Listeners/
│   │   └── SendOrderConfirmationSmsListenerTest.php
│   ├── Jobs/
│   │   └── ImportProductsFromCsvJobTest.php
│   └── States/
│       └── Orders/
│           └── PendingToShippedTest.php
└── Fakes/
    ├── FakeStripeService.php
    └── FakeSmsService.php
```

**Naming:** `{SourceFile}Test.php` — `PlaceOrderActionTest.php` for `PlaceOrderAction`.

---

## Pest.php — base setup

```php
<?php

declare(strict_types=1);

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

uses(TestCase::class, RefreshDatabase::class)->in('Feature', 'Unit');

function actingAsUser(?User $user = null): User
{
    $user = $user ?? User::factory()->create();
    test()->actingAs($user);
    return $user;
}

function asMerchant(): User
{
    return actingAsUser(User::factory()->merchant()->create());
}
```

---

## phpunit.xml — environment

```xml
<env name="APP_ENV" value="testing"/>
<env name="DB_CONNECTION" value="pgsql"/>
<env name="DB_DATABASE" value="testing"/>
<env name="QUEUE_CONNECTION" value="sync"/>
<env name="MAIL_MAILER" value="array"/>
<env name="CACHE_DRIVER" value="array"/>
<env name="SESSION_DRIVER" value="array"/>
```

Notes:
- `QUEUE_CONNECTION=sync` runs queued listeners inline; override with `Queue::fake()` when asserting dispatch
- `CACHE/SESSION=array` prevents cross-test state via Redis

---

## CI command

```bash
php artisan test --parallel --coverage --min=70
```

- `--parallel` runs across multiple processes (Postgres schemas auto-split)
- Coverage threshold is **advisory only** — does not block PRs

---

## Pest conventions

### 1. Prefer `it(...)` over `test(...)` — reads as a sentence

```php
it('places an order successfully', function () {
    // ...
});
```

### 2. Group with `describe()` when a file has 8+ tests

```php
describe('placing an order', function () {
    it('succeeds for valid input', function () { /* ... */ });
    it('returns 422 when items are empty', function () { /* ... */ });
});

describe('listing orders', function () {
    it('returns the authenticated user orders only', function () { /* ... */ });
});
```

### 3. Higher-order tests for trivial assertions

```php
it('rejects unauthenticated requests')
    ->getJson('/orders')
    ->assertUnauthorized();
```

### 4. Datasets for parametric tests

```php
it('rejects invalid phone numbers', function (string $phone) {
    actingAsUser();
    $this->postJson('/orders', validOrderPayload(['phone' => $phone]))
        ->assertUnprocessable();
})->with([
    'empty' => '',
    'too short' => '123',
    'international format' => '+44 20 7946 0958',
    'non-numeric' => 'not-a-phone',
]);
```

### 5. Custom expectations for repeated assertions

In `tests/Pest.php`:

```php
expect()->extend('toBeOrderInStatus', function (string $stateClass) {
    expect($this->value->status)->toBeInstanceOf($stateClass);
    return $this;
});

// Usage in tests
expect($order)->toBeOrderInStatus(Shipped::class);
```

### 6. Use `beforeEach()` for shared setup

```php
beforeEach(function () {
    $this->merchant = User::factory()->merchant()->create();
    $this->actingAs($this->merchant);
});

it('lists merchant orders', function () {
    Order::factory()->for($this->merchant)->count(3)->create();
    $this->getJson('/orders')->assertOk()->assertJsonCount(3, 'data');
});
```

---

## Feature tests — full HTTP round-trip

**Location:** `tests/Feature/{Domain}/`
**Tests:** Controllers, FormRequests, JsonResources, full flow integration

### Skeleton — write endpoint

```php
<?php

declare(strict_types=1);

use App\Events\OrderPlacedEvent;
use App\Models\Order\Order;
use App\Models\Product;
use App\Models\User;
use Illuminate\Support\Facades\Event;

it('places an order successfully', function () {
    $merchant = User::factory()->merchant()->create();
    $product = Product::factory()->for($merchant)->create(['stock' => 10]);
    actingAsUser($merchant);

    Event::fake([OrderPlacedEvent::class]);

    $response = $this->postJson('/orders', [
        'customer_name' => 'Ahmad Ali',
        'phone' => '0912345678',
        'items' => [
            ['product_id' => $product->id, 'quantity' => 2],
        ],
    ]);

    // Assert HTTP response
    $response->assertCreated()
        ->assertJsonStructure(['data' => ['id', 'number', 'customer_name', 'status', 'items']])
        ->assertJsonPath('data.customer_name', 'Ahmad Ali')
        ->assertJsonPath('data.status', 'pending');

    // Assert DB state
    expect(Order::count())->toBe(1);
    $order = Order::first();
    expect($order->customer_name)->toBe('Ahmad Ali');
    expect($order->items)->toHaveCount(1);

    // Assert event fired
    Event::assertDispatched(OrderPlacedEvent::class, fn ($e) => $e->order->id === $order->id);
});

it('returns 422 when items are empty', function () {
    actingAsUser();

    $this->postJson('/orders', [
        'customer_name' => 'Ahmad Ali',
        'phone' => '0912345678',
        'items' => [],
    ])
        ->assertUnprocessable()
        ->assertJsonValidationErrors(['items']);
});

it('returns 401 when not authenticated', function () {
    $this->postJson('/orders', [])->assertUnauthorized();
});

it('returns 403 when user cannot create orders', function () {
    $user = User::factory()->create();   // not a merchant
    $this->actingAs($user);

    $this->postJson('/orders', validOrderPayload())->assertForbidden();
});
```

### Skeleton — read endpoint

```php
<?php

declare(strict_types=1);

use App\Models\Order\Order;
use App\Models\User;

beforeEach(function () {
    $this->merchant = User::factory()->merchant()->create();
    $this->actingAs($this->merchant);
});

it('lists the authenticated user orders', function () {
    Order::factory()->for($this->merchant, 'merchant')->count(3)->create();
    Order::factory()->count(2)->create();   // another user's orders, should NOT appear

    $this->getJson('/orders')
        ->assertOk()
        ->assertJsonCount(3, 'data')
        ->assertJsonStructure(['data' => [['id', 'number', 'status']], 'meta', 'links']);
});

it('filters orders by status', function () {
    Order::factory()->for($this->merchant, 'merchant')->pending()->count(2)->create();
    Order::factory()->for($this->merchant, 'merchant')->shipped()->count(1)->create();

    $this->getJson('/orders?status=pending')
        ->assertOk()
        ->assertJsonCount(2, 'data');
});

it('paginates with configurable per_page', function () {
    Order::factory()->for($this->merchant, 'merchant')->count(25)->create();

    $this->getJson('/orders?per_page=10')
        ->assertOk()
        ->assertJsonCount(10, 'data')
        ->assertJsonPath('meta.total', 25);
});
```

### Feature test rules

1. **One feature test file per endpoint or controller.** Named after the operation: `PlaceOrderTest.php`, not `OrderControllerTest.php`
2. **One scenario per test.** "places an order" + "returns 422 when items empty" = two tests
3. **Assert at the right layers:**
   - HTTP status code
   - Response JSON shape (key fields via `assertJsonStructure` / `assertJsonPath`)
   - Database state (specific records, specific columns)
   - Dispatched events / jobs (via `Event::fake()` / `Queue::fake()`)
4. **Use factory methods, not hand-crafted arrays.** `User::factory()->merchant()` over `User::create([...])`
5. **Don't assert implementation details.** No spying that "the Action was called." Assert the observable outcome (DB state, response, dispatched events)
6. **Test the four pillars of an endpoint:**
   - Happy path (200/201)
   - Validation errors (422)
   - Unauthenticated (401)
   - Forbidden (403)

---

## Unit tests — Action

**Location:** `tests/Unit/Actions/{Domain}/`
**Tests:** Direct `handle()` invocation, no HTTP

### Skeleton

```php
<?php

declare(strict_types=1);

use App\Actions\Orders\PlaceOrderAction;
use App\Data\Orders\CreateOrderData;
use App\Data\Orders\OrderItemData;
use App\Events\OrderPlacedEvent;
use App\Exceptions\InsufficientInventoryException;
use App\Models\Order\Order;
use App\Models\Product;
use App\Models\User;
use Illuminate\Support\Facades\Event;

it('creates an order with items', function () {
    $merchant = User::factory()->merchant()->create();
    $product = Product::factory()->for($merchant)->create(['stock' => 10]);

    Event::fake();

    $dto = new CreateOrderData(
        customer_name: 'Test Customer',
        phone: '0912345678',
        items: [new OrderItemData(product_id: $product->id, quantity: 2)],
    );

    $order = app(PlaceOrderAction::class)->handle($dto);

    expect($order)->toBeInstanceOf(Order::class);
    expect($order->customer_name)->toBe('Test Customer');
    expect($order->items)->toHaveCount(1);
});

it('throws when inventory is insufficient', function () {
    $merchant = User::factory()->merchant()->create();
    $product = Product::factory()->for($merchant)->create(['stock' => 1]);

    $dto = new CreateOrderData(
        customer_name: 'Test Customer',
        phone: '0912345678',
        items: [new OrderItemData(product_id: $product->id, quantity: 5)],
    );

    expect(fn () => app(PlaceOrderAction::class)->handle($dto))
        ->toThrow(InsufficientInventoryException::class);
});

it('fires OrderPlacedEvent after placing', function () {
    Event::fake([OrderPlacedEvent::class]);
    $merchant = User::factory()->merchant()->create();
    $product = Product::factory()->for($merchant)->create(['stock' => 10]);

    $dto = new CreateOrderData(
        customer_name: 'Test Customer',
        phone: '0912345678',
        items: [new OrderItemData(product_id: $product->id, quantity: 1)],
    );

    $order = app(PlaceOrderAction::class)->handle($dto);

    Event::assertDispatched(OrderPlacedEvent::class, fn ($e) => $e->order->id === $order->id);
});

it('rolls back when persistence fails mid-transaction', function () {
    $merchant = User::factory()->merchant()->create();
    $product = Product::factory()->for($merchant)->create(['stock' => 10]);

    // Force an exception inside the transaction
    Product::saving(function () {
        throw new \Exception('boom');
    });

    $dto = new CreateOrderData(
        customer_name: 'Test Customer',
        phone: '0912345678',
        items: [new OrderItemData(product_id: $product->id, quantity: 1)],
    );

    expect(fn () => app(PlaceOrderAction::class)->handle($dto))->toThrow(\Exception::class);
    expect(Order::count())->toBe(0);   // transaction rolled back
});
```

### Action test rules

1. **Resolve via the container** (`app(MyAction::class)`) so dependencies inject — NOT `new MyAction()` unless the constructor has no dependencies
2. **Call `handle()` directly** — no HTTP, no routes
3. **Use `Event::fake()` and `Queue::fake()`** to assert events/jobs without running the full chain
4. Assert observable outcomes (return value, DB state, fired events) — not implementation
5. Test happy path + each domain exception thrown + transaction rollback if relevant

---

## Unit tests — Query

**Location:** `tests/Unit/Queries/{Domain}/`

### Skeleton

```php
<?php

declare(strict_types=1);

use App\Data\Orders\OrderFiltersData;
use App\Models\Order\Order;
use App\Models\User;
use App\Queries\Orders\ListUserOrdersQuery;

it('returns only the given user orders', function () {
    $user = User::factory()->create();
    $otherUser = User::factory()->create();

    Order::factory()->for($user, 'merchant')->count(3)->create();
    Order::factory()->for($otherUser, 'merchant')->count(5)->create();

    $result = app(ListUserOrdersQuery::class)->handle($user, new OrderFiltersData());

    expect($result->total())->toBe(3);
});

it('filters by status', function () {
    $user = User::factory()->create();
    Order::factory()->for($user, 'merchant')->pending()->count(2)->create();
    Order::factory()->for($user, 'merchant')->shipped()->count(1)->create();

    $result = app(ListUserOrdersQuery::class)->handle(
        $user,
        new OrderFiltersData(status: 'pending')
    );

    expect($result->total())->toBe(2);
});

it('searches by order number', function () {
    $user = User::factory()->create();
    Order::factory()->for($user, 'merchant')->create(['number' => 'ORD-2026-00001']);
    Order::factory()->for($user, 'merchant')->create(['number' => 'ORD-2026-00002']);

    $result = app(ListUserOrdersQuery::class)->handle(
        $user,
        new OrderFiltersData(search: '00001')
    );

    expect($result->total())->toBe(1);
});
```

### Query test rules

1. Call `handle()` directly
2. Hit the real database (Postgres + `RefreshDatabase` per test)
3. Test the filter combinations and edge cases
4. Test scoping (e.g., "only returns the given user's data")

---

## Unit tests — Service with `Http::fake()`

```php
<?php

declare(strict_types=1);

use App\Exceptions\PaymentDeclinedException;
use App\Exceptions\PaymentGatewayException;
use App\Services\StripeService;
use Illuminate\Support\Facades\Http;

it('returns a RefundData on successful refund', function () {
    Http::fake([
        'api.stripe.com/v1/refunds' => Http::response([
            'id' => 're_123',
            'amount' => 1000,
            'status' => 'succeeded',
        ], 200),
    ]);

    $result = app(StripeService::class)->refund('ch_123', 1000);

    expect($result->id)->toBe('re_123');
    expect($result->amount)->toBe(1000);
});

it('throws PaymentDeclinedException on card errors', function () {
    Http::fake([
        'api.stripe.com/v1/refunds' => Http::response([
            'error' => ['type' => 'card_error', 'message' => 'Card was declined'],
        ], 402),
    ]);

    expect(fn () => app(StripeService::class)->refund('ch_123', 1000))
        ->toThrow(PaymentDeclinedException::class);
});

it('throws PaymentGatewayException on api errors', function () {
    Http::fake([
        'api.stripe.com/v1/refunds' => Http::response(['error' => 'server error'], 500),
    ]);

    expect(fn () => app(StripeService::class)->refund('ch_123', 1000))
        ->toThrow(PaymentGatewayException::class);
});
```

---

## Unit tests — Service with vendor SDK (interface + Fake class)

For Services wrapping a vendor SDK (e.g., the official Stripe PHP client) where `Http::fake()` can't intercept the calls, use an interface + hand-written fake.

### Fake class

```php
<?php

declare(strict_types=1);

namespace Tests\Fakes;

use App\Data\Payments\RefundData;
use App\Services\Contracts\PaymentGatewayContract;

final class FakeStripeService implements PaymentGatewayContract
{
    public array $refunds = [];
    public ?\Throwable $nextException = null;

    public function refund(string $chargeId, int $amount): RefundData
    {
        if ($this->nextException) {
            throw $this->nextException;
        }
        $this->refunds[] = compact('chargeId', 'amount');
        return new RefundData(id: 're_fake_' . count($this->refunds), amount: $amount);
    }

    public function failNextWith(\Throwable $e): void
    {
        $this->nextException = $e;
    }
}
```

### Usage in tests

```php
<?php

declare(strict_types=1);

use App\Actions\Orders\RefundOrderAction;
use App\Exceptions\PaymentDeclinedException;
use App\Models\Order\Order;
use App\Models\Order\States\Refunded;
use App\Services\Contracts\PaymentGatewayContract;
use Tests\Fakes\FakeStripeService;

it('marks order as refunded after Stripe refund succeeds', function () {
    $stripe = new FakeStripeService();
    app()->instance(PaymentGatewayContract::class, $stripe);

    $order = Order::factory()->paid()->create();

    app(RefundOrderAction::class)->handle($order, 1000);

    expect($stripe->refunds)->toHaveCount(1);
    expect($stripe->refunds[0]['amount'])->toBe(1000);
    expect($order->fresh())->toBeOrderInStatus(Refunded::class);
});

it('does not mark order as refunded if Stripe declines', function () {
    $stripe = new FakeStripeService();
    $stripe->failNextWith(new PaymentDeclinedException('declined'));
    app()->instance(PaymentGatewayContract::class, $stripe);

    $order = Order::factory()->paid()->create();

    expect(fn () => app(RefundOrderAction::class)->handle($order, 1000))
        ->toThrow(PaymentDeclinedException::class);

    expect($order->fresh()->status)->not->toBeInstanceOf(Refunded::class);
});
```

### Service test rules

1. For Services using Laravel's `Http::*` client → `Http::fake()`
2. For Services wrapping a vendor SDK → define interface, write `Fake*` class in `tests/Fakes/`, bind in test via `app()->instance(...)`
3. For internal Services with no external dependency (`OrderNumberGeneratorService`, `MoneyFormatterService`) → no fake; call directly
4. Test the **observable behavior** (return values, exceptions thrown), not the internal HTTP/SDK calls

---

## Unit tests — Listener

```php
<?php

declare(strict_types=1);

use App\Events\OrderPlacedEvent;
use App\Listeners\SendOrderConfirmationSmsListener;
use App\Models\Order\Order;
use App\Services\Contracts\SmsServiceContract;
use Tests\Fakes\FakeSmsService;

it('sends SMS via the SMS service when order placed', function () {
    $sms = new FakeSmsService();
    app()->instance(SmsServiceContract::class, $sms);

    $order = Order::factory()->create([
        'phone' => '0912345678',
        'number' => 'ORD-2026-00042',
    ]);

    app(SendOrderConfirmationSmsListener::class)->handle(new OrderPlacedEvent($order));

    expect($sms->sent)->toHaveCount(1);
    expect($sms->sent[0]['to'])->toBe('0912345678');
    expect($sms->sent[0]['message'])->toContain('ORD-2026-00042');
});
```

### Listener test rules

1. Unit-test `handle()` directly — instantiate via container, call with a fake event
2. The corresponding **Feature test** asserts the Listener gets dispatched (via `Event::fake()` + `Event::assertDispatched(...)`)
3. Don't run both Listener `handle()` AND the source Action in the same test — they're separate units

---

## Unit tests — Job

```php
<?php

declare(strict_types=1);

use App\Events\ProductsImportedEvent;
use App\Jobs\ImportProductsFromCsvJob;
use App\Models\User;
use Illuminate\Http\Testing\File;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Storage;

it('imports products from a CSV file', function () {
    $merchant = User::factory()->create();
    Storage::fake('local');
    $path = Storage::disk('local')->putFile('imports', new File(stub('products.csv')));

    Event::fake([ProductsImportedEvent::class]);

    app(ImportProductsFromCsvJob::class, [
        'filePath' => $path,
        'merchant' => $merchant,
    ])->handle(app(\App\Services\CsvParserService::class));

    expect($merchant->products()->count())->toBe(3);
    Event::assertDispatched(ProductsImportedEvent::class);
});
```

### Feature side (dispatch assertion)

```php
use App\Jobs\ImportProductsFromCsvJob;
use Illuminate\Http\Testing\File;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Queue;

it('dispatches import job when CSV uploaded', function () {
    Queue::fake();
    actingAsUser();

    $this->postJson('/imports', [
        'file' => UploadedFile::fake()->create('products.csv'),
    ])->assertSuccessful();

    Queue::assertPushed(ImportProductsFromCsvJob::class);
});
```

---

## Unit tests — State transition

```php
<?php

declare(strict_types=1);

use App\Models\Order\Order;
use App\Models\Order\States\Cancelled;
use App\Models\Order\States\Shipped;
use Spatie\ModelStates\Exceptions\CouldNotPerformTransition;

it('transitions Pending to Shipped with tracking number', function () {
    $order = Order::factory()->pending()->create();

    $order->status->transitionTo(Shipped::class, 'TRACK123');

    expect($order->status)->toBeInstanceOf(Shipped::class);
    expect($order->tracking_number)->toBe('TRACK123');
    expect($order->shipped_at)->not->toBeNull();
});

it('does not allow transition from Cancelled to Shipped', function () {
    $order = Order::factory()->cancelled()->create();

    expect(fn () => $order->status->transitionTo(Shipped::class, 'TRACK123'))
        ->toThrow(CouldNotPerformTransition::class);
});
```

---

## Factories — required for every model

### Skeleton

```php
<?php

declare(strict_types=1);

namespace Database\Factories;

use App\Models\Order\Order;
use App\Models\Order\States\Pending;
use App\Models\Order\States\Shipped;
use App\Models\Order\States\Cancelled;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

final class OrderFactory extends Factory
{
    protected $model = Order::class;

    public function definition(): array
    {
        return [
            'merchant_id' => User::factory()->merchant(),
            'customer_name' => fake()->name(),
            'phone' => '09' . fake()->numerify('########'),
            'total_amount' => fake()->numberBetween(1000, 100000),
            'status' => Pending::class,
        ];
    }

    public function pending(): static
    {
        return $this->state(['status' => Pending::class]);
    }

    public function shipped(): static
    {
        return $this->state([
            'status' => Shipped::class,
            'tracking_number' => fake()->bothify('TRK-#########'),
            'shipped_at' => now(),
        ]);
    }

    public function cancelled(): static
    {
        return $this->state([
            'status' => Cancelled::class,
            'cancelled_at' => now(),
        ]);
    }

    public function paid(): static
    {
        return $this->state(['paid_at' => now()]);
    }
}
```

### Factory rules

1. **Every model has a factory** — non-negotiable
2. Realistic data via Faker; not `'name' => 'test'`
3. State methods for variations: `Order::factory()->pending()`, `User::factory()->merchant()`
4. Factories DO NOT trigger external side effects — model hooks are plumbing only (UUIDs, slugs)

---

## General rules

1. **Tests are mandatory for critical paths.** Critical = auth, authorization, payments, orders, anything touching money, anything affecting other users' data.
2. **Tests are encouraged but not required** for everything else. Judgment call.
3. **Bug fixes get a regression test in the same PR.**
4. **Tests must pass in parallel.** No test depends on another. No shared mutable globals.
5. **No `sleep()` or arbitrary waits.** If a test needs to wait for something, the design is wrong.
6. **No flaky tests committed to main.** Intermittent failure = broken; quarantine or fix immediately.
7. **One scenario per test.** Don't bundle "creates" + "validates" + "authorizes" in one test.
8. **Use Carbon::setTestNow() for time-dependent code** — never depend on actual clock.
9. **Use UploadedFile::fake()** for file upload tests.
10. **Always `RefreshDatabase`** — never manually clean DB state.

---

## What NOT to test

1. **DTOs** — constructing one and reading it back tests PHP, not your code
2. **Eloquent's behavior** — `User::create()` returning a User is Laravel's job to test
3. **Framework behavior** — don't test middleware Laravel ships with
4. **Private methods** — test the public interface
5. **Getters and setters with no logic** — `$user->name` returning a string is not a test case
6. **Trivial accessors** — unless they have logic worth testing

---

## CI configuration

### GitHub Actions snippet

```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: testing
    ports: [5432:5432]
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5

steps:
  - uses: actions/checkout@v4
  - uses: shivammathur/setup-php@v2
    with:
      php-version: '8.3'
      coverage: pcov
  - run: composer install --prefer-dist --no-interaction
  - run: php artisan test --parallel --coverage --min=70
```

---

## Decision tree — what kind of test do I write?

```
Is it triggered by an HTTP request?
  → Feature test in tests/Feature/{Domain}/

Is it an Action, Query, Service, Listener, Job, or State transition?
  → Unit test in tests/Unit/{Type}/{Domain}/

Does the Service call an external HTTP API via Laravel's Http client?
  → Use Http::fake() in the unit test

Does the Service call a vendor SDK (e.g., Stripe PHP client)?
  → Define an interface, write a Fake* class in tests/Fakes/

Does the code fire events or dispatch jobs?
  → Use Event::fake(), Queue::fake(), Bus::fake() to assert dispatch without running them

Is it a DTO, plain getter, framework behavior, or private method?
  → Don't test it
```

---

## Generation checklist — when asked to write tests

For a new write endpoint (e.g., `POST /orders`):

1. **Feature test**: `tests/Feature/{Domain}/{Verb}{Object}Test.php`
   - Happy path (201 + DB state + event dispatched)
   - Validation errors (422 with specific fields)
   - Unauthenticated (401)
   - Forbidden (403)
2. **Unit test for the Action**: `tests/Unit/Actions/{Domain}/{Verb}{Object}ActionTest.php`
   - Happy path (return value, DB state, event fired via `Event::fake`)
   - Each domain exception thrown
   - Transaction rollback if applicable
3. **Unit tests for any new Listener** triggered by the Action's Event
4. **Unit test for any new Service** (with `Http::fake()` or Fake class)
5. **Factory updates** if new model states are introduced

For a new read endpoint:

1. **Feature test**: HTTP response shape + filtering + pagination
2. **Unit test for the Query**: filter combinations, scoping, edge cases

Always show complete test files with full namespaces, `<?php`, `declare(strict_types=1);`, and proper `use` statements. Never produce partial snippets unless explicitly asked.
