# Job

## Definition

A class representing **deferred or asynchronous work**. Jobs are dispatched to a queue and run later by a worker process. Unlike Actions (synchronous), Jobs run with no caller waiting for a result. Used for bulk operations, scheduled work, retry-required external calls, and long-running tasks triggered by webhooks or admins.

## Keywords

Async • Deferred • Background work • Queued task • Worker job

## Purpose

To move work off the HTTP request thread, to enable retries with backoff, to handle long-running batch operations, and to run scheduled or webhook-triggered work without blocking other parts of the system.

## Rules

1. One Job per coherent unit of async work.
2. **Implements `ShouldQueue`** (always).
3. Dispatched via `Job::dispatch(...)`, never called directly except in tests.
4. Single public method: `handle()` (Laravel resolves constructor args from `dispatch()`, then injects dependencies into `handle()`).
5. Constructor receives only **serializable** state (primitives, models, simple DTOs). Models are serialized via `SerializesModels` — the queue stores the ID and refetches.
6. Jobs MAY fire Events from inside `handle()`.
7. Jobs MAY use Services and Queries.
8. Jobs SHOULD NOT call Actions directly; if a Job needs an Action's logic, reconsider whether the work belongs as a Job at all.
9. **`final` by default.**
10. Configure retries, backoff, timeout, queue per Job class.

## Shape

```php
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

final class {VerbObject} implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60;
    public int $timeout = 120;
    public string $queue = 'default';

    public function __construct(/* serializable state */) {}

    public function handle(/* injected dependencies */): void
    {
        // do the work
    }

    public function failed(\Throwable $e): void
    {
        // optional: handle final failure
    }
}
```

## Parameters

### Constructor
- Models (auto-serialized to ID and refetched on run)
- Primitives
- Data classes (must be serializable)
- File paths (not File instances — store file then pass path)

### `handle()` method
- Services / other dependencies (injected from container)

## Returns

`void`. Jobs do not return results to their dispatcher.

## Is it `final`?

**Yes.**

## When to use

- **Bulk operations** (`ImportProductsFromCsv`, `ExportOrdersToXlsx`)
- **Scheduled work** dispatched from the scheduler (`SendDailyDigest`, `ChargeRecurringSubscriptions`)
- **Webhook handlers** (return 200 immediately, process in background)
- **Retry-required external calls** (sync to third-party APIs with exponential backoff)
- **Long-running tasks** that shouldn't block the request thread
- **Delayed work** (`->delay(now()->addDay())`)
- **Chained or batched operations** (`Bus::chain([...])`, `Bus::batch([...])`)

## When NOT to use

- **Synchronous business operations** → use an Action
- **Reactions to events** → use a Listener (Listeners are technically queue jobs but live in a different mental category)
- **Trivial work** completing in milliseconds — overhead of queuing exceeds the benefit
- **Anything that needs to return data to the caller** → Jobs can't; restructure

## Lifecycle

1. Caller invokes `Job::dispatch(...)` with constructor args
2. Job is serialized and pushed to the configured queue (Redis, database, etc.)
3. HTTP request returns immediately
4. Queue worker picks up the job
5. Worker deserializes — models refetched from DB by ID
6. Container resolves dependencies and calls `handle()`
7. On exception → retry per `$tries` / `$backoff`
8. On final failure → optional `failed()` method runs; entry written to `failed_jobs` table
9. On success → job removed from queue

## What controls it

- **Triggered by:**
  - Actions (`Job::dispatch(...)`)
  - Controllers (when work should run async)
  - Console commands
  - Scheduler (`$schedule->job(...)`)
  - Webhook handlers
  - Other Jobs (chained or batched)
- **Configuration:**
  - Per-Job: `$tries`, `$backoff`, `$timeout`, `$queue`
  - Per-queue: `config/queue.php`
  - Worker behavior: `php artisan queue:work` flags or Horizon config
- **Monitoring:** Laravel Horizon (Redis-backed queues only)

## Examples

### Bulk import

```php
namespace App\Jobs;

use App\Events\ProductsImported;
use App\Models\User;
use App\Services\CsvParser;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

final class ImportProductsFromCsv implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 1;          // imports shouldn't auto-retry
    public int $timeout = 600;      // 10 min for large files
    public string $queue = 'imports';

    public function __construct(
        public string $filePath,
        public User $merchant,
    ) {}

    public function handle(CsvParser $parser): void
    {
        $imported = collect();
        foreach ($parser->rows($this->filePath) as $row) {
            $product = $this->merchant->products()->create($row);
            $imported->push($product);
        }

        event(new ProductsImported($this->merchant, $imported));
    }
}
```

### Scheduled job

```php
final class SendDailyDigestToMerchants implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function handle(): void
    {
        User::merchants()->chunk(100, function ($merchants) {
            foreach ($merchants as $merchant) {
                SendMerchantDigest::dispatch($merchant);
            }
        });
    }
}
```

Scheduled in `routes/console.php`:

```php
Schedule::job(new SendDailyDigestToMerchants)->dailyAt('08:00');
```

### Delayed job (dispatched from an Action)

```php
// Inside an Action
public function handle(CreateOrderData $data): Order
{
    $order = DB::transaction(fn () => Order::create([...]));
    event(new OrderPlaced($order));

    SendUnpaidOrderReminder::dispatch($order)->delay(now()->addDay());

    return $order;
}
```

### Webhook handler dispatching a Job

```php
public function stripe(Request $request, StripeService $stripe)
{
    $payload = $stripe->verifyWebhook($request);
    ProcessStripeEvent::dispatch($payload);
    return response('', 200);
}
```

## Anti-examples

### ❌ Job called directly

```php
$job = new SendEmail($user);
$job->handle();   // ❌ defeats async, no retry, no error handling
```

Always dispatch.

### ❌ Job for trivial work

```php
final class IncrementCounter implements ShouldQueue
{
    public function handle(): void
    {
        Cache::increment('counter');
    }
}
```

Queue overhead exceeds benefit. Inline.

### ❌ Job constructor with non-serializable state

```php
public function __construct(public \PDO $connection) {}  // ❌ not serializable
```

Constructor args go through serialization. Use IDs, paths, or scalar values.

### ❌ Job that's really an Action

```php
final class PlaceOrder implements ShouldQueue { /* ... */ }
```

If a user is waiting for the result of placing an order, it shouldn't be a Job. Actions are synchronous. Jobs are for work no one is waiting for.
