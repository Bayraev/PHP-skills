# Testing with PHPUnit

Tests are the safety net that makes refactoring and upgrades cheap. PHPUnit 10+
attributes are the current style (PHPUnit 12 supports PHP 8.3+); older annotation-based
suites (`@dataProvider` docblocks) still appear in maintained codebases — match the
existing style when extending.

## Layout and configuration

```
src/                    # production code (PSR-4: Acme\Shop\ => src/)
tests/
    Unit/               # fast, in-memory, no I/O
    Integration/        # database, filesystem, real HTTP
    phpunit.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         bootstrap="vendor/autoload.php"
         colors="true"
         failOnWarning="true"
         failOnRisky="true">
    <testsuites>
        <testsuite name="unit">
            <directory>tests/Unit</directory>
        </testsuite>
    </testsuites>
    <source>
        <include>
            <directory>src</directory>
        </include>
    </source>
</phpunit>
```

`failOnWarning`/`failOnRisky` turn deprecations and sloppy tests into CI failures —
that is what keeps the next PHP upgrade cheap. Run with `vendor/bin/phpunit`, filter
with `--filter testMethodName`, one suite with `--testsuite unit`.

## Test anatomy

One behavior per test; the method name states the behavior under test, not the method
name being called:

```php
<?php
declare(strict_types=1);

namespace Acme\Shop\Tests\Unit;

use Acme\Shop\Money;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\TestCase;

final class MoneyTest extends TestCase
{
    #[Test]
    public function rejects_negative_amounts(): void
    {
        $this->expectException(\InvalidArgumentException::class);

        new Money(-5, 'EUR');
    }

    #[Test]
    public function adds_same_currency_amounts(): void
    {
        $sum = (new Money(150, 'EUR'))->add(new Money(50, 'EUR'));

        $this->assertSame(200, $sum->amount);
    }

    public static function provides_currencies(): \Generator
    {
        yield 'euro' => ['EUR'];
        yield 'dollar' => ['USD'];
    }

    #[DataProvider('provides_currencies')]
    #[Test]
    public function keeps_currency_through_operations(string $currency): void
    {
        $money = new Money(100, $currency);

        $this->assertSame($currency, $money->currency);
    }
}
```

- Providers are `public static` returning an iterable; keys become case names in
  output — named yields make failures readable. Prefer providers over loops *inside*
  a test so each case reports independently.
- Assert with the most specific method: `assertSame` (identity) for scalars and
  enums — `assertEquals`'s loose comparison hides type bugs. `assertSame` on enums
  checks the singleton. For objects/arrays: `assertEquals` structural checks or
  `assertSame` for identity, chosen deliberately.
- `expectException()` must be called *before* the throwing code; it asserts the class —
  add `expectExceptionMessage` for specificity. To assert on post-exception state, wrap
  in try/catch with a `$this->fail()` fallback.
- Don't test private methods/protected internals. Test the observable behavior that
  depends on them; when you feel the need to test a private, the class usually has a
  second concept begging to be extracted.

## Test doubles

```php
<?php
declare(strict_types=1);

$repo = $this->createMock(OrderRepository::class);
$repo->method('find')->willReturn(new Order(id: 7, status: 'shipped'));

$notifier = $this->createMock(Notifier::class);
$notifier->expects($this->once())
    ->method('sendShipped')
    ->with($this->callback(fn (Order $o): bool => $o->isShipped()));
```

- Mock across boundaries you own the interface of (repositories, clocks, transport).
  For third-party clients, wrap them in your own thin interface and mock the wrapper —
  you cannot change the vendor API when tests need seams.
- `expects($this->once())` is a specification of an interaction — use it only when the
  interaction *is* the behavior (email sent exactly once). Over-specifying call counts
  couples tests to implementation and they start failing on harmless refactors.
- Stub values with `willReturn`, dynamic with `willReturnCallback`, sequences with
  `willReturnOnConsecutiveCalls`, exceptions with `willThrowException`.
- Fake implementations (in-memory repository) beat heavy mocks for stateful boundaries —
  faster and closer to real behavior.
- Time: inject a clock (`function now(): \DateTimeImmutable`) so tests control time
  instead of sleeping or `strtotime` hacks.

## Integration tests

- Separate testsuite (they're slower); skip cleanly when infrastructure is absent:
  `markTestSkipped('DB not configured')` in `setUp`.
- Database: transaction-per-test rolled back in `tearDown`, or truncate between tests
  (see database reference). Assert via fresh queries, not the code under test.
- Each test leaves the world as it found it; tests that depend on execution order are
  bugs that surface only in CI.
