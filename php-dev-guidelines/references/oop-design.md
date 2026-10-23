# OOP design in modern PHP

Classes are the unit of design. The goals: invariants enforced by the engine, APIs that
static analysis can check, and hierarchies shallow enough to reason about.

## Classes and immutability

Default to `final` classes with promoted, typed, `readonly` properties. This one shape —
a value object / DTO — covers most classes in an application, and the engine enforces
its invariants instead of code review:

```php
<?php
declare(strict_types=1);

final readonly class Money
{
    public function __construct(
        public int $amount,       // minor units: cents
        public string $currency,  // ISO 4217
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative');
        }
    }

    public function add(self $other): self
    {
        if ($other->currency !== $this->currency) {
            throw new \InvalidArgumentException('Currency mismatch');
        }

        return new self($this->amount + $other->amount, $this->currency);
    }
}
```

Notes on the shape:
- `readonly` classes (8.2+) make every promoted property readonly; combine with
  `final` so no subclass can weaken the guarantee.
- Store compound values in their domain representation (cents + ISO code), not strings
  — parsing belongs at the boundary.
- `self` parameter/return types document "expects this exact value type."
- "Withers" (methods returning modified copies) on 8.5 use
  `clone($this, withProperties: [...])`; on 8.3–8.4, readonly reinit during `__clone` or
  plain `new self(...)` construction.

Constructor property promotion rules: promoted parameters may declare visibility
(`public`/`protected`/`private`), `readonly` (8.1+), and hooks (8.4+). Promoted
parameters cannot have defaults that are non-constant... `new` in initializers (8.1+)
*is* allowed, enabling default collaborators.

## Property hooks vs getters (8.4+)

Hooks keep the property syntax while adding behavior — callers see `$order->total`, not
`$order->getTotal()`, and interfaces can now *require* properties:

```php
<?php
declare(strict_types=1);

interface HasLabel
{
    public string $label { get; }   // 8.4: interface can declare a property with get
}
```

Guidance: use hooks for access-shaping (trim, normalize, derive); use methods for
computation with parameters or side effects (queries, I/O, expensive builds). A virtual
property (getter-only, no backing value) throws on write, making it a cheap computed
field. Asymmetric visibility (`public private(set)`) covers the "public read, internal
write" case that used to force a getter — prefer it over hooks when the only goal is
write protection.

## Interfaces and abstract classes

- **Interfaces define capabilities** (`RepairsClock`, `StoresEvents`) — keep them small,
  one verb; classes compose many. Program call-sites against the interface so test
  doubles and alternate implementations slot in.
- **Abstract classes share implementation** along one axis. Use for template methods
  where the base fixes the algorithm and hooks fill steps. Unlike interfaces, a class
  extends only one — reserve that for genuine "is a kind of, with shared state."
- Add `#[\Override]` (8.3+) on implementing/overriding methods. When a parent signature
  changes, implementations that no longer match become engine errors instead of silent
  orphans.
- Constructor signature contracts: interfaces cannot force one, so expose a named static
  factory in the interface (`fromArray(array $row): static`) when implementations must be
  constructible from shared inputs.

## Traits

Traits paste code into the using class; use for horizontal reuse across unrelated
classes (auditing, timestamp behavior), not as a substitute for composition — a trait
cannot be mocked or swapped independently. Conflict resolution with `insteadof`/`as`
when two traits collide; `as` can also rename for clarity or change visibility
(`use Timer { tick as protected; }`). Traits support methods, properties, constants
(8.2+), abstract method requirements — declaring an abstract method in a trait forces
the using class to implement it, useful for "needs a `key()` to be usable."

## Enums (8.1+)

Enums replace string/int flag constants with a closed set of instances, exhaustively
checkable with `match`:

```php
<?php
declare(strict_types=1);

enum OrderStatus: string
{
    case Draft = 'draft';
    case Paid = 'paid';
    case Shipped = 'shipped';

    public function label(): string
    {
        return match ($this) {
            self::Draft => 'Awaiting payment',
            self::Paid, self::Shipped => 'In fulfilment',
        };
    }

    public static function fromName(string $name): self
    {
        return match (true) {
            str_starts_with(strtolower($name), 'd') => self::Draft,
            default => throw new \UnexpectedValueException("Unknown status: {$name}"),
        };
    }
}

$status = OrderStatus::from('paid');      // throws ValueError on unknown value
$maybe  = OrderStatus::tryFrom('zombie'); // null instead of throwing
$all    = OrderStatus::cases();           // for exhaustive UI lists and tests
```

Design points:
- Backing type is `int` or `string` only. `from`/`tryFrom` map scalar↔case; never accept
  the scalar into your domain — convert at the boundary.
- Enums may implement interfaces, use traits, have static methods, constants, and
  constructor-less private state via methods; they cannot declare properties (instance
  or static) and cannot be instantiated with `new`.
- Each case is a singleton: compare with `===`, share them freely.
- Enums are serialized to their backing value by `json_encode` (backed) — define
  `from`-based hydration on the way in.
- When a category needs data, switch from const-like enums to "smart enum" methods
  returning `match`ed values rather than adding per-case classes, until behavior truly
  diverges — then one class per case (a small hierarchy) is cleaner.

## Static-analysis-friendly code

PHPStan/Psalm read the same type system as the engine, so anything you write for the
engine helps them, plus:

- Fully type every property, parameter, return — including `private` members, where the
  engine never checks but analysis does.
- `assert()` or `is_*`-narrow at boundaries when input is `mixed`; analyzers understand
  the narrowing and stop complaining downstream.
- Prefer `instanceof` checks over `get_class()` comparisons — subclass-friendly.
- `@phpstan-`/`@psalm-` annotations only where the native syntax cannot express the
  shape (generics on arrays/collections, conditional returns); keep them close to the
  signature and sparse.
- Nullability is a type-state decision: if a property is temporarily null only between
  construction and a setup call, that's two-phase initialization — usually better
  modeled by constructing complete objects later (factory) or making setup part of the
  constructor.

## Object lifecycle details that matter

- `__construct` may throw; partially constructed objects are garbage-collected safely.
- `clone` is shallow by default; deep-copy semantics belong in `__clone`. In 8.5,
  `clone($obj, withProperties: [...])` covers the common "copy with changes" without
  `__clone` at all, and can reassign readonly props from scopes with set access.
- `__debugInfo()` customizes `var_dump`; `__toString()` must return a string or throw —
  no side effects, it runs in string context implicitly.
- Serialization: implement `__serialize()`/`__unserialize()` (arrays, not byte strings)
  rather than `__sleep`/`__wakeup`; enums serialize as their name.
