# Section 9: Testing Strategy

## Goal

Define what we test, how we test, and where each kind of test lives. Tests serve two purposes: catching regressions and documenting expected behavior. They must be **fast, deterministic, and aligned with the architecture** locked in Section 4.

## Locked decisions

| Topic | Decision |
|---|---|
| **Framework** | Pest |
| **Philosophy** | Architecture-driven (School C) — feature tests for HTTP, unit tests for Actions/Queries/Services |
| **Coverage** | Soft target ~70%, advisory only |
| **Mandatory tests** | Critical paths must have tests; rest is judgment call |
| **Database** | Real Postgres + `RefreshDatabase` everywhere |
| **Browser / E2E** | Not used — feature tests cover enough |
| **External services** | Mix — `Http::fake()` for HTTP clients; interfaces + `Fake*` classes for vendor SDKs |
| **Parallelism** | Parallel by default (`php artisan test --parallel`) |

## Stack

- `pestphp/pest` — test framework
- `pestphp/pest-plugin-laravel` — Laravel helpers
- PHPUnit (under the hood — Pest builds on it)
- Postgres (test database, same engine as production)
- `Http::fake()`, `Event::fake()`, `Queue::fake()`, `Bus::fake()`, `Mail::fake()`, `Notification::fake()`, `Storage::fake()` — built-in Laravel fakes
- Custom `Fake*` Service classes (in `tests/Fakes/`) for vendor-SDK-based Services

## Folder structure

```
tests/
├── Pest.php                    # Pest configuration, global hooks
├── TestCase.php                # Base case (extends Laravel's TestCase)
├── CreatesApplication.php
├── Feature/                    # HTTP-level integration tests
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
├── Unit/                       # Direct class invocation, no HTTP
│   ├── Actions/
│   │   └── Orders/
│   │       ├── PlaceOrderTest.php
│   │       └── CancelOrderTest.php
│   ├── Queries/
│   │   └── Orders/
│   │       └── ListUserOrdersQueryTest.php
│   ├── Services/
│   │   ├── OrderNumberGeneratorTest.php
│   │   └── StripeServiceTest.php
│   ├── Listeners/
│   │   └── SendOrderConfirmationSmsTest.php
│   ├── Jobs/
│   │   └── ImportProductsFromCsvTest.php
│   └── States/
│       └── Orders/
│           └── PendingToShippedTest.php
└── Fakes/                      # Hand-written fake Services
    ├── FakeStripeService.php
    └── FakeSmsService.php
```

## What gets tested where

| Component | Test type | Location | Notes |
|---|---|---|---|
| Controllers | Feature | `tests/Feature/` | Full HTTP round-trip |
| FormRequests | Feature | (via controller test) | Asserted by hitting the endpoint with bad/unauthorized input |
| JsonResources | Feature | (via controller test) | Asserted on response JSON shape |
| Actions | Unit | `tests/Unit/Actions/` | Direct `handle()` invocation |
| Queries | Unit | `tests/Unit/Queries/` | Direct `handle()` with real DB |
| Services (Http-based) | Unit | `tests/Unit/Services/` | `Http::fake()` |
| Services (SDK-based) | Unit | `tests/Unit/Services/` | Mock the SDK client at construction |
| Listeners | Unit + Feature | both | Unit: `handle()`. Feature: `Event::fake()` to assert dispatch. |
| Jobs | Unit + Feature | both | Unit: `handle()`. Feature: `Queue::fake()` to assert dispatch. |
| State transitions | Unit | `tests/Unit/States/` | Atomic logic |
| Model `boot()` hooks | Implicit | (via feature/unit tests) | Hooks fire as side effects of any model operation |
| DTOs | Not tested | — | Plain data containers; testing them tests PHP itself |
| Domain exceptions | Not in isolation | — | Asserted on in Action tests via `expect(fn () => ...)->toThrow(...)` |
| Model traits | Implicit | — | Tested through any model that uses them |

## Configuration

### `phpunit.xml` (Pest reads this)

Key settings:

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
- **Postgres `testing` database** is created once locally and in CI; tests share it via transactions
- **QUEUE_CONNECTION=sync** so queued listeners run inline; override per-test with `Queue::fake()` when asserting dispatch
- **CACHE/SESSION=array** so tests don't share state via Redis

### `Pest.php`

```php
uses(Tests\TestCase::class, RefreshDatabase::class)->in('Feature', 'Unit');

// Helper functions
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

### CI test runner

```bash
php artisan test --parallel --coverage --min=70
```

- `--parallel` runs tests across multiple processes (database is automatically split into per-process schemas)
- `--coverage` enabled in CI but does not block (advisory)
- `--min=70` warns but does not fail PRs (`--stop-on-failure` separately controls hard failure)

In `composer.json`:

```json
"scripts": {
    "test": "php artisan test --parallel",
    "test:coverage": "php artisan test --parallel --coverage"
}
```

## Rules

### General

1. **Tests are mandatory for critical paths.** Critical = authentication, authorization, payment, order placement, anything touching money, anything affecting other users' data.
2. **Tests are encouraged but not required** for everything else. Judgment call per PR.
3. **Bug fixes that don't have a regression test in the PR require the reviewer to ask why.** Not a hard block; a culture norm.
4. **Tests must pass in parallel.** No test depends on another test's state. No shared mutable globals (use `Carbon::setTestNow()` per-test, not module-level).
5. **No `sleep()` or arbitrary waits.** If a test needs to wait for something, the design is wrong.
6. **No flaky tests committed to main.** A test that fails intermittently is broken; quarantine or fix immediately.

### Feature tests

1. **One feature test file per endpoint or per controller**, named after what's being tested: `PlaceOrderTest.php`, not `OrderControllerTest.php`.
2. **Assert at the right layers:**
   - HTTP status code
   - Response JSON shape (key fields)
   - Database state (specific records, specific columns)
   - Dispatched events / jobs (via `Event::fake()` / `Queue::fake()`)
3. **One scenario per test.** "It places an order successfully" + "It returns 422 when items are empty" = two tests.
4. **Use factory methods, not hand-crafted arrays.** `User::factory()->merchant()->create()` over `User::create([...long array...])`.
5. **Don't assert implementation details.** Asserting "the action was called once" via spying is a smell; assert the *observable outcome* (DB state, response, dispatched events).

### Unit tests (Actions, Queries, Services)

1. **Call `handle()` directly.** No HTTP. No routes.
2. **Resolve via the container** (`app(MyAction::class)`) so dependencies inject — not `new MyAction()` unless the constructor has no dependencies.
3. **Use `Event::fake()` and `Queue::fake()` in Action tests** to assert events/jobs are dispatched without running the full chain.
4. **DB state assertions** are fine in unit tests — they use `RefreshDatabase` too.

### Services with HTTP calls

```php
it('translates Stripe card errors to PaymentDeclinedException', function () {
    Http::fake([
        'api.stripe.com/v1/refunds' => Http::response(['error' => ['type' => 'card_error']], 402),
    ]);

    expect(fn () => app(StripeService::class)->refund('ch_123', 1000))
        ->toThrow(PaymentDeclinedException::class);
});
```

### Services with vendor SDKs

1. Define an interface (`StripeServiceContract`).
2. Real implementation lives in `app/Services/`.
3. Fake implementation lives in `tests/Fakes/`.
4. Bind in test: `app()->instance(StripeServiceContract::class, new FakeStripeService());`

```php
// tests/Fakes/FakeStripeService.php
final class FakeStripeService implements StripeServiceContract
{
    public array $refunds = [];
    public ?\Throwable $nextException = null;

    public function refund(string $chargeId, int $amount): RefundDto
    {
        if ($this->nextException) {
            throw $this->nextException;
        }
        $this->refunds[] = compact('chargeId', 'amount');
        return new RefundDto(id: 're_fake_' . count($this->refunds), amount: $amount);
    }

    public function failNextWith(\Throwable $e): void
    {
        $this->nextException = $e;
    }
}
```

```php
// In a test
it('marks order as refunded after Stripe refund', function () {
    $stripe = new FakeStripeService();
    app()->instance(StripeServiceContract::class, $stripe);

    $order = Order::factory()->paid()->create();
    app(RefundOrder::class)->handle($order, 1000);

    expect($stripe->refunds)->toHaveCount(1);
    expect($order->fresh()->status)->toBeInstanceOf(Refunded::class);
});
```

### Listeners

1. **Unit test the `handle()` method directly:**

```php
it('sends SMS via the SMS service when order placed', function () {
    $sms = new FakeSmsService();
    app()->instance(SmsServiceContract::class, $sms);

    $order = Order::factory()->create(['phone' => '0912345678']);
    app(SendOrderConfirmationSms::class)->handle(new OrderPlaced($order));

    expect($sms->sent)->toHaveCount(1);
    expect($sms->sent[0]['to'])->toBe('0912345678');
});
```

2. **Feature test asserts the listener is registered:**

```php
it('dispatches OrderPlaced event after placing order', function () {
    Event::fake([OrderPlaced::class]);

    actingAsUser();
    $this->postJson('/orders', validOrderPayload())->assertSuccessful();

    Event::assertDispatched(OrderPlaced::class);
});
```

Don't assert the listener ran in feature tests — that's the listener's own test's job. Just assert the event fired.

### Jobs

Same dual approach: unit-test `handle()` directly; feature-test that the job was dispatched.

```php
// Unit
it('imports products from a CSV file', function () {
    $merchant = User::factory()->create();
    $path = Storage::fake('local')->putFileAs('imports', new File(/* test csv */), 'products.csv');

    app(ImportProductsFromCsv::class)->handle($path, $merchant);

    expect($merchant->products()->count())->toBe(3);
});

// Feature
it('dispatches import job when CSV uploaded', function () {
    Queue::fake();
    actingAsUser();

    $this->postJson('/imports', ['file' => UploadedFile::fake()->create('products.csv')])
         ->assertSuccessful();

    Queue::assertPushed(ImportProductsFromCsv::class);
});
```

### State transitions

```php
it('transitions order from Pending to Shipped with tracking number', function () {
    $order = Order::factory()->pending()->create();

    $order->status->transitionTo(Shipped::class, 'TRACK123');

    expect($order->status)->toBeInstanceOf(Shipped::class);
    expect($order->tracking_number)->toBe('TRACK123');
    expect($order->shipped_at)->not->toBeNull();
});

it('throws when trying to ship an already-cancelled order', function () {
    $order = Order::factory()->cancelled()->create();

    expect(fn () => $order->status->transitionTo(Shipped::class, 'TRACK123'))
        ->toThrow(CouldNotPerformTransition::class);
});
```

## Test data — Factories

1. **Every model that gets created in tests has a factory.** This is non-negotiable.
2. **Factories use realistic data by default.** Use Faker; not `'name' => 'test'`.
3. **State methods for variations:** `User::factory()->merchant()`, `Order::factory()->pending()`, `Order::factory()->paid()`.
4. **Factories DO NOT trigger model lifecycle hooks that have external side effects** — but our model hooks are plumbing only (UUIDs, slugs), so factories work normally.

```php
// database/factories/OrderFactory.php
final class OrderFactory extends Factory
{
    protected $model = Order::class;

    public function definition(): array
    {
        return [
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
}
```

## Pest-specific conventions

1. **Use `it(...)` over `test(...)`** for readability: `it('places an order successfully', ...)` reads as a sentence.
2. **Group related tests with `describe(...)` blocks** when a file has 8+ tests:
   ```php
   describe('placing an order', function () {
       it('succeeds for valid input', function () { /* ... */ });
       it('returns 422 for empty items', function () { /* ... */ });
   });
   ```
3. **Use higher-order tests for trivial assertions:**
   ```php
   it('returns 401 when not authenticated')
       ->getJson('/orders')
       ->assertUnauthorized();
   ```
4. **Datasets for parametric tests:**
   ```php
   it('rejects invalid phone numbers', function (string $phone) {
       $this->postJson('/orders', ['phone' => $phone])->assertUnprocessable();
   })->with(['', '123', '+44 20 7946 0958', 'not-a-phone']);
   ```
5. **Custom expectations** for repeated assertions — define in `Pest.php`:
   ```php
   expect()->extend('toBeOrderInStatus', function (string $stateClass) {
       expect($this->value->status)->toBeInstanceOf($stateClass);
       return $this;
   });
   ```

## What NOT to test

1. **DTOs** — they're plain data containers; constructing one and reading it back tests PHP, not your code.
2. **Eloquent's behavior** — `User::create()` returning a User is Laravel's job to test, not yours.
3. **Framework behavior** — don't test middleware that Laravel ships with; test your own.
4. **Private methods** — test the public interface; private methods are implementation details.
5. **Getters and setters with no logic** — `$user->name` returning a string is not a test case.

## CI integration

In CI (`.github/workflows/ci.yml` or similar):

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
    with: { php-version: '8.3', coverage: pcov }
  - run: composer install --prefer-dist --no-interaction
  - run: php artisan test --parallel --coverage --min=70
```

Coverage threshold is advisory — the `--min=70` flag warns but the CI job won't fail PRs for being under 70% on its own. The hard PR-blocking jobs are Pint, Larastan, Rector (Section 3), and tests passing (this section).

## Summary — the testing decision tree

```
Is it a controller / HTTP endpoint?
  → Feature test in tests/Feature/

Is it an Action, Query, Service, Listener, Job, or State transition?
  → Unit test in tests/Unit/{Type}/

Does it call an external HTTP API via Laravel's Http client?
  → Use Http::fake() in the unit test

Does it call a vendor SDK?
  → Define an interface, bind a Fake* implementation

Does it fire events or dispatch jobs that we don't want to actually run?
  → Event::fake() / Queue::fake() / Bus::fake()

Is it a DTO, plain getter, framework behavior, or private method?
  → Don't test it
```
