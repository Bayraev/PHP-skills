# Modern PHP: versions 8.1–8.5

Choose features by the project's minimum PHP version, not the newest release. This file
maps each version to its practically useful features (all verified against the php.net
migration guides) and gives upgrade guidance. Before writing code that uses a feature,
confirm the project's floor version via `composer.json`.

## Table of contents

- [Feature reference by version](#feature-reference-by-version)
  - [PHP 8.1](#php-81)
  - [PHP 8.2](#php-82)
  - [PHP 8.3](#php-83)
  - [PHP 8.4](#php-84)
  - [PHP 8.5](#php-85)
- [Choosing the target version](#choosing-the-target-version)
- [Feature gating and polyfills](#feature-gating-and-polyfills)
- [Upgrade checklist](#upgrade-checklist)

## Feature reference by version

### PHP 8.1

**Enums** — pure and backed, with methods. See `oop-design.md` for the full treatment.

```php
<?php
enum Status: string
{
    case Draft = 'draft';
    case Published = 'published';

    public function isPublic(): bool
    {
        return $this === self::Published;
    }
}
```

**Readonly properties** — initialize once from inside the class scope; the engine
guarantees immutability instead of discipline doing it.

```php
<?php
final class LineItem
{
    public function __construct(
        public readonly string $sku,
        public readonly int $quantity,
    ) {}
}
```

**First-class callable syntax** — `strlen(...)` replaces `Closure::fromCallable('strlen')`
and `[$obj, 'method']`. The `...` is part of the syntax, not an omission. Use it wherever
a callable is passed: `array_map(strtoupper(...), $words)`.

**Intersection types** — require *all* types; cannot be mixed with unions in the same
declaration until DNF types in 8.2.

```php
<?php
function persist(Countable & Iterator $collection): void {}
```

**`never` return type** — for functions that always throw or exit. It is the only type
that is implicitly covariant: a `never` function satisfies any expected return type,
letting the type checker narrow control flow after the call.

**`new` in initializers** — `new Foo()` allowed as parameter defaults, static variable
initializers, and attribute arguments. Enables the "default collaborator" pattern:
`function __construct(private Logger $logger = new NullLogger()) {}`.

**Array unpacking with string keys** — `['a' => 1, ...$overrides]` gives immutable-style
"overwrite" semantics that reads clearly.

**Fibers** — full-stack coroutines for library authors (Revolt, AmPHP, async adapters).
Application code rarely touches them directly.

### PHP 8.2

**Readonly classes** — marks every instance property readonly and forbids dynamic ones.
The cheapest way to make trustworthy DTOs and value objects:

```php
<?php
final readonly class Address
{
    public function __construct(
        public string $street,
        public string $city,
        public string $countryCode,
    ) {}
}
```

**DNF (disjunctive normal form) types** — unions of intersections, written
`(A & B) | C`. Useful for "iterable of X and Countable or array" style contracts.

**Standalone `null`, `false`, `true` types** — useful with unions: `false|Result`
for legacy "return false on failure" code, or `true` for always-succeeding checks.

**Constants in traits** — traits can now define constants, so shared behavior can carry
its configuration.

**`#[\SensitiveParameter]`** — redacts marked arguments in stack traces. Cheap hygiene
for anything handling credentials or tokens.

**Dynamic properties deprecated** — writing `$obj->undeclared = 1` emits a deprecation.
This is the main noise source when running older code on 8.2+; fix by declaring the
property (opt back in deliberately with `#[\AllowDynamicProperties]` only for genuine
dynamic containers like config bags).

### PHP 8.3

**Typed class constants** — extends type safety to constants on classes, interfaces,
traits, and enums.

```php
<?php
interface Config
{
    public const int CACHE_TTL = 3600;
}
```

**`#[\Override]` attribute** — the engine verifies the method actually overrides a parent
or implements an interface. Annotate overrides so renames upstream become compile errors
instead of silent dead code.

**`json_validate()`** — cheap syntactic validation of a JSON string without building the
decoded structure. Use it when you only need "is this valid JSON" (webhook signature
checks, config linting); when you will decode anyway, call `json_decode()` directly and
use `JSON_THROW_ON_ERROR` — validating first then decoding parses twice for nothing.

**Readonly amendments** — anonymous readonly classes allowed, and readonly properties may
be reinitialized while cloning, which enables clone-based "wither" methods on readonly
classes (see `oop-design.md`).

**Dynamic class-constant fetch** — `Foo::{$name}` replaces `constant("Foo::$name")`.

### PHP 8.4

**Property hooks** — per-property get/set logic; can make a property virtual (no backing
value). Replaces most getter/setter boilerplate while keeping the natural
`$person->name` syntax:

```php
<?php
final class Person
{
    public string $name {
        get => $this->name;
        set => trim($value);
    }

    public string $shoutName {
        get => strtoupper($this->name);
    }

    public function __construct(string $name)
    {
        $this->name = $name;
    }
}
```

A short `set => expr` form stores the expression's result automatically; a braced
`set { ... }` body assigns `$this->prop` itself. A getter-only property throws on write.

**Asymmetric visibility** — separate read and write visibility. Lets you expose public
reads while keeping mutation inside the class, without writing a getter:

```php
<?php
final class Order
{
    public private(set) \DateTimeImmutable $placedAt;

    public function __construct()
    {
        $this->placedAt = new \DateTimeImmutable();
    }
}
```

Rule: the set visibility may only be equal or narrower than the get visibility
(`public protected(set)`, `protected private(set)`, etc.).

**Lazy objects** — `ReflectionClass::newLazyGhost()` / `newLazyProxy()` defer
construction until first access. Primarily a building block for DI containers and ORM
proxies; reach for it when initializing an object is expensive and often unnecessary.

**`#[\Deprecated]`** — mark your own functions/constants deprecated with proper
deprecation diagnostics. Pair with a removal note in the docblock.

**`new` chaining without parentheses** — `new Router($config)->dispatch($request)`.

**`RoundingMode` enum** — `round($x, mode: RoundingMode::HalfAwayFromZero)` and friends,
clearer than `PHP_ROUND_HALF_*` constants, with four new modes.

**PDO driver subclasses** — `PDO::connect()` returns driver-specific instances
(`Pdo\MySQL` etc.). See `database-pdo.md`.

### PHP 8.5

**Pipe operator `|>`** — pass the left-hand value as the *first argument* to the
right-hand first-class callable; chains read top-to-bottom:

```php
<?php
$slug = '  Hello World '
    |> trim(...)
    |> strtolower(...);
```

The right side must be a bare `f(...)` — no extra arguments. When the value belongs in a
later position, wrap: `10 |> (fn (int $x): int => add($x, 5))(...)`.

**Closures and first-class callables in constant expressions** — allowed as attribute
arguments, property/parameter defaults, and (class) constants.

**`#[\NoDiscard]` and the `(void)` cast** — annotate functions whose return value matters;
ignoring it emits a warning. Cast to `(void)` when ignoring is intentional — the cast has
no runtime effect, it is documentation the engine can check.

**Attributes on constants**, `#[\Override]` on properties, static properties with
asymmetric visibility, constructor promotion of `final` properties, and casts in constant
expressions (`const T = (int) 0.3;`) round out the set.

**`clone($obj, withProperties: [...])`** — cloning with property overrides, the modern
wither primitive:

```php
<?php
class Money
{
    public function __construct(
        public readonly int $amount,
        public readonly string $currency,
    ) {}
}

class Bonus extends Money
{
    public function doubled(): static
    {
        return clone($this, withProperties: ['amount' => $this->amount * 2]);
    }
}
```

Reassigning a readonly property during clone-with is only allowed from a scope that has
set access (readonly behaves as implicitly `protected(set)`) — hence the subclass above.

**`FILTER_THROW_ON_FAILURE`** flag for the filter functions, and a new always-enabled
`uri` extension for RFC 3986/WHATWG URL work.

## Choosing the target version

- **Libraries:** support the oldest branch still in security support or later (8.3+ in
  late 2026), expressed as `"php": "^8.3"` in composer.json. Widening later is easy;
  breaking early adopters is not.
- **Applications/services:** 8.4+ is the sweet spot in late 2026 — full active support,
  and property hooks/asymmetric visibility remove real boilerplate. 8.5 if you control
  deployment.
- **Legacy platforms pinned to old PHP** (some commercial hosting): still write code that
  is forward-compatible — avoid re-introducing deprecated patterns (dynamic properties,
  `switch` fall-through, implicit `null` params) so the later upgrade is cheap.

## Feature gating and polyfills

When one codebase must run on multiple versions:

- Prefer polyfills (`symfony/polyfill-php83` etc.) to `function_exists` blocks; they are
  battle-tested and disappear from the diff on upgrade.
- For syntax-gated features, gate at the *file* level: keep 8.4+ syntax in classes loaded
  conditionally, or ship parallel implementations selected by a factory. `if (PHP_VERSION_ID >= 80400)`
  does not help if the file containing hooks fails to parse on 8.3 — parsing happens
  before execution.
- `PHP_VERSION_ID` constants: 80300 = 8.3.0, 80400 = 8.4.0. Use `>=`, never `===`.

## Upgrade checklist

1. Read the target version's migration guide on php.net (`migration84`,
   `migration85`, ...) — the "Backward Incompatible Changes" and "Deprecated Features"
   sections are the ones that bite.
2. Update `"php"` constraint and run `composer update --lock` / install on the new PHP;
   then `composer audit`.
3. Run the test suite on the new version *before* deploying — extensions and ini
   defaults change between versions.
4. Grep for known pain points of the target: dynamic properties (8.2 deprecation),
   `utf8_encode`/`utf8_decode` (deprecated 8.2), `${var}` string interpolation
   (removed 8.3), `E_ALL` error reporting to surface deprecations early.
5. Turn deprecations into CI failures (`error_reporting(E_ALL)` in phpunit.xml) so the
   *next* upgrade stays cheap.
