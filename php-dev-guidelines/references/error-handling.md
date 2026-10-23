# Error handling: exceptions, errors, and diagnostics

PHP has two parallel failure channels. Knowing which channel a failure belongs to — and
which layer should handle it — is most of the design work.

## The two hierarchies

Both implement `Throwable` (so `catch (Throwable $e)` catches everything), but they
answer different questions:

```text
Throwable
├── Error                      engine-level: bugs in code — usually let it crash, fix the code
│   ├── TypeError              wrong type/return at a call boundary (incl. strict_types hits)
│   │   └── ArgumentCountError too few arguments
│   ├── ValueError             right type, unacceptable value (e.g. json_validate depth)
│   ├── ArithmeticError        e.g. intdiv(PHP_INT_MIN, -1), bit-shift by negative
│   │   └── DivisionByZeroError  / and % by zero — including 1.0/0.0; use fdiv()
│   │                            for IEEE semantics (INF/NaN, no throw)
│   ├── UnhandledMatchError    a match with no matching arm and no default
│   └── CompileError / ParseError
└── Exception                  runtime/environmental failures — catch and handle
    ├── LogicException         + subclasses (InvalidArgumentException, DomainException,
    │                            OutOfRangeException, LengthException...) — caller screwed up
    ├── RuntimeException       + subclasses (UnexpectedValueException, PDOException...) —
    │                            the world failed (I/O, network, external service)
    └── ErrorException         wrapper that turns native PHP warnings/notices into exceptions
```

Practical division:

- **Programmer error (Error, LogicException):** fail fast, fix the code. Don't catch
  `TypeError` to "handle" a wrong argument type — the caller should pass the right type.
  Catching these converts bugs into mysterious behavior.
- **Runtime failure (RuntimeException and domain exceptions):** expected, recoverable,
  or at least reportable — missing file, dead database, invalid user input. Catch at a
  boundary that can actually do something (retry, show a message, roll back).
- Domain code should mostly throw its own `Exception` subclasses for business-rule
  failures (`OrderAlreadyShipped`), built on `RuntimeException` or `LogicException`
  depending on blame.

## Throwing well

```php
<?php
declare(strict_types=1);

final class ShipmentNotFound extends \RuntimeException {}

final class ShipmentRepository
{
    public function find(string $id): Shipment
    {
        $row = $this->db->fetchOne($id);

        return $row
            ?? throw new ShipmentNotFound("Shipment {$id} does not exist");
    }
}
```

- Exception *type* is the API; the message is for humans. Code that catches should match
  the class, not parse the message.
- `throw` is an expression (8.0): `?? throw`, `?: throw`, and inside arrow functions all
  work — use it to make "can't continue" inline instead of three extra lines.
- Give exceptions context as structured fields, not just string interpolation:
  `previous:` for chained causes (JSON failures inside a parser error), dedicated
  properties for IDs the handler needs.
- Fail with the most specific built-in SPL exception when you don't have a domain type
  yet: `InvalidArgumentException` for caller mistakes, `UnexpectedValueException` for
  bad external data, `RuntimeException` for environment.

## Catching well

```php
<?php
declare(strict_types=1);

try {
    $body = json_decode($raw, true, 512, JSON_THROW_ON_ERROR);
    $order = Order::fromArray($body);
} catch (\JsonException $e) {
    throw new InvalidRequest('Malformed JSON body', previous: $e);
} catch (\OrderNotFound | \ShipmentNotFound $e) {   // multi-catch
    error_log($e->getMessage());
    throw new InvalidRequest('Referenced resource missing', previous: $e);
} finally {
    $this->meter->stop();   // always runs, even on rethrow/return
}
```

Rules of thumb:
- Catch the narrowest type that you can meaningfully handle. `catch (Throwable)` belongs
  only at true boundaries (job worker top frame, CLI entry, error-rendering middleware)
  where the job is logging-and-stop, not "handling."
- Don't swallow. An empty catch block is a lie to the next reader — at minimum log with
  context, or comment why ignoring is correct (`// best-effort metrics`).
- Multi-catch (`A | B`) instead of duplicated bodies; `finally` for cleanup that must run
  regardless (releases, connection returns).
- Re-throw wrapped when crossing layers: lower-level exceptions leaking upward couple
  caller code to implementation details (a controller shouldn't know about PDO).
- Convert warnings to exceptions in library/entry code when you want failures strict:
  `set_error_handler` throwing `ErrorException` — standard in modern frameworks, and it
  makes `file_get_contents` failures catchable.

## Configuration and reporting

Production-safe defaults:

```php
<?php
declare(strict_types=1);

error_reporting(E_ALL);          // let deprecations surface in logs/CI
ini_set('display_errors', '0');  // never render errors to users (leaks paths, secrets)
ini_set('log_errors', '1');      // ...write them somewhere you read
```

- `display_errors=1` in production both looks unprofessional and discloses filesystem
  layout/stack details; errors go to logs instead.
- `set_exception_handler(callable)` handles *uncaught* throwables — one place to log the
  fatal and render a generic response/exit code. `register_shutdown_function` +
  `error_get_last()` catches fatal errors the exception handler can't.
- Deprecation notices are future upgrade work made visible — treat them as CI failures
  (`failOnWarning`/`failOnRisky` in phpunit.xml, or convert via error handler) instead of
  ignoring until the major bump breaks.
- `#[\SensitiveParameter]` on secret-bearing parameters keeps them out of stack traces —
  stack traces of unexpected exceptions get logged and pasted into tickets.
