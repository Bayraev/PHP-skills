# PDO and database patterns

PDO is the standard database abstraction layer. Its core safety property is prepared
statements with bound parameters — same API across drivers, injection-proof when used
correctly.

## Connecting

```php
<?php
declare(strict_types=1);

$pdo = new PDO(
    'mysql:host=127.0.0.1;dbname=shop;charset=utf8mb4',
    getenv('DB_USER') ?: 'shop',
    getenv('DB_PASS') ?: '',
    [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES => false,
        PDO::ATTR_STRINGIFY_FETCHES => false,
    ],
);
```

- `ERRMODE_EXCEPTION` is the default since PHP 8.0 and the only sane mode — the
  constructor *always* throws `PDOException` on connection failure regardless of the
  attribute. Other modes (silent/warning) exist for legacy code; new code shouldn't
  choose them because every call site must then inspect `errorCode()` manually.
- Put the charset in the DSN (`charset=utf8mb4` for MySQL) — SET NAMES after connect
  can miss the mark and historically enabled encoding-based injection.
- `ATTR_EMULATE_PREPARES => false` forces native prepares: the server sees parameters
  as typed values, native types survive round trips, and the parser compiles once.
  Emulation is sometimes useful for `LIMIT ?` on old setups, but keep it off by default.
- Set `ATTR_DEFAULT_FETCH_MODE` once instead of passing `PDO::FETCH_ASSOC` to every
  fetch — consistency and less noise.
- On 8.4+, `PDO::connect($dsn, ...)` returns driver-specific subclasses (`Pdo\MySQL`)
  exposing driver features; plain `new PDO(...)` remains fully supported.
- Connections are cheap to establish per request but expensive per *query batch* in
  long-runners — reuse the handle; persistent connections (`PDO::ATTR_PERSISTENT`)
  are a last resort with real footguns (transaction state, locks surviving requests).

## Prepared statements

```php
<?php
declare(strict_types=1);

$stmt = $pdo->prepare('SELECT id, name, email FROM users WHERE status = ? AND age > ?');
$stmt->bindValue(1, 'active', PDO::PARAM_STR);
$stmt->bindValue(2, 18, PDO::PARAM_INT);
$stmt->execute();
$users = $stmt->fetchAll();
```

Or pass values straight to `execute()` (all bound as strings — fine for most drivers,
but explicit `bindValue(..., PDO::PARAM_INT)` matters where MySQL would otherwise coerce
in surprising places, e.g. `LIMIT ?` under emulation):

```php
<?php
declare(strict_types=1);

$stmt = $pdo->prepare('UPDATE users SET last_seen = NOW() WHERE id = :id');
$stmt->execute(['id' => $userId]);
$affected = $stmt->rowCount();
```

- Positional `?` and named `:name` parameters cannot mix in one statement.
- IN-lists: one placeholder per element (see security reference) — never interpolate.
- Reuse one prepared statement for many executions inside a loop; re-preparing per row
  throws away the whole point.
- `rowCount()` on SELECT is not portable across drivers (MySQL returns rows touched);
  for counting, use `SELECT COUNT(*)`.

## Fetching

```php
<?php
declare(strict_types=1);

$stmt = $pdo->prepare('SELECT id, name FROM users WHERE tenant_id = ?');
$stmt->execute([$tenantId]);

foreach ($stmt as $row) {            // streams: constant memory for big results
    printf("%d %s\n", $row['id'], $row['name']);
}

$lookup = $stmt->fetchAll(PDO::FETCH_KEY_PAIR);   // exactly 2 columns → [id => name]
$users  = array_map(User::fromRow(...), $stmt->fetchAll()); // hydrate after fetch
```

`fetchAll` materializes everything — fine for pages, wrong for exports (iterate or use
`FETCH_FUNC`/generators; unbuffered MySQL queries via `MYSQL_ATTR_USE_BUFFERED_QUERY =>
false` trade connection freedom for memory).

## Transactions

```php
<?php
declare(strict_types=1);

$pdo->beginTransaction();
try {
    $chargeId = $charges->insert($order);
    $order->markPaid($chargeId);
    $orders->update($order);

    $pdo->commit();
} catch (Throwable $e) {
    if ($pdo->inTransaction()) {
        $pdo->rollBack();
    }

    throw $e;
}
```

- Keep transactions short: open after you have the data, commit before slow side
  effects (email, HTTP) — locks are held the whole time.
- Emulate nested transactions in app code (counter/savepoints) if needed; PDO throws
  `PDOException` ("There is already an active transaction") on naive nesting.
- SQLite/MySQL DDL, and some statements, cause implicit commits — schema changes do not
  belong inside data transactions.
- Serializability anomalies (lost updates under concurrent reads) are not solved by
  transactions alone — use row locking (`SELECT ... FOR UPDATE`) or optimistic version
  columns.
- `$pdo->inTransaction()` guards the rollback because commit failures or a prior
  implicit commit can end the transaction before your catch runs.

## Mapping rows to objects

Keep SQL rows out of your domain: fetch arrays/objects at the edge, hydrate into typed
constructors (`User::fromRow(array $row): self`) which validate and convert types.
Enum-backed columns: `OrderStatus::from($row['status'])` converts at the boundary so a
bad value fails there, with a clear message, instead of deep in business logic.

## Testing database code

Use a real engine (SQLite in-memory is fine when SQL is compatible; otherwise a
throwaway MySQL/Postgres container) via PDO — mocks of PDO verify nothing about SQL.
Wrap each test in a transaction rolled back in `tearDown`, or truncate between tests;
never run integration tests against shared persistent data.
