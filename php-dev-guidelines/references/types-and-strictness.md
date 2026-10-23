# Type system and strictness

PHP's type system is opt-in per file. Its value comes from making failures local and
obvious — which only happens when types are complete and strict.

## strict_types

`declare(strict_types=1);` must be the very first statement of the file (only comments
and the opening tag may precede it). It changes how *calls made from this file* coerce
argument and return types:

- Without it: `"5"` passes for `int $x` (numeric-string coercion), `5.9` passes for
  `int` (silently truncates to `5`), `5` passes for `string` (becomes `"5"`).
  Non-numeric strings (`"abc"`), `null`, and wrong resource/object types still throw
  `TypeError` even in weak mode — but the coercions that *do* happen are exactly the
  ones that turn wrong-but-numeric data into plausible-looking results.
- With it: values must already be the right type; wrong types throw `TypeError` at the
  call site.

Because the mode binds to the file where the *call* happens, put it in every file. The
one legitimate weak coercion — passing `int` where `float` is expected — still works
under strict types; that is the only widening allowed.

```php
<?php
declare(strict_types=1);

function takeInt(int $n): int
{
    return $n * 2;
}

takeInt('10');        // TypeError: must be of type int, string given — strict mode
takeInt(10);          // 20
```

## Type declarations in practice

- **Parameters and returns on everything public.** Untyped APIs push bug discovery from
  compile/test time to production.
- **Typed properties** (8.0+) — uninitialized typed properties are *not* `null`; reading
  before initialization throws `Error: must not be accessed before initialization`. That
  is a feature: it catches missing construction immediately. Nullable properties default
  to `null`.
- **`mixed`** means "any type, including null" — use it only at genuine boundaries
  (deserializers, generic containers) and narrow with `is_*` checks immediately after.
- **`void`** for "returns nothing, don't use the result"; **`never`** for "never returns"
  (throws/exits). `never` is useful in guards: after calling a `never`-returning
  `fail()`, static analyzers know the code below is unreachable.
- **Nullable spelling:** `?Type` and `Type|null` are identical; pick one per codebase.
  Avoid `?Type` *parameters* when a sensible non-null default exists — nullable
  parameters create two code paths where one would do.

## Composite types

```php
<?php
declare(strict_types=1);

interface HasId {}
interface HasName {}

// Union (8.0): one of these types
function render(int|string $id): string
{
    return "id: " . $id;
}

// Intersection (8.1): all of these types at once
function label(HasId & HasName $entity): string
{
    return $entity->getName() . ' (#' . $entity->getId() . ')';
}

// DNF (8.2): unions of intersections, intersections must be parenthesized
function countAll((Countable & Iterator) | array $items): int
{
    return is_array($items) ? count($items) : iterator_count($items);
}

// Standalone null/false/true (8.2) — good for incremental typing of legacy returns
function findUser(int $id): User|false
{
    return false; // legacy sentinel; prefer User|null or exceptions in new code
}
```

Intersection types are invariant on parameters in practice — pass exactly the declared
intersection or a subclass satisfying all members.

Variance worth remembering: return types may narrow (covariant), parameter types may
widen (contravariant). Property types are invariant — a child class cannot change a
property's type at all.

## Coercion traps to know cold

These bite even experienced developers; each is a real bug class:

- **String-to-number comparison changed in 8.0:** `0 == 'foo'` was `true` in PHP 7
  (non-numeric string cast to 0) and is `false` in 8+ (number cast to string when the
  other side is non-numeric). Legacy code relying on the old behavior silently changes
  meaning after upgrade. Always compare against `===` unless you *want* numeric
  coercion.
- **Numeric strings:** `'1e3' == '1000'` is true — both are numeric strings, compared
  numerically. When comparing identifiers, cast: `(int)$id === $id2` or compare as
  strings deliberately.
- **`array_search`/`in_array` default to loose comparison.** `in_array('abc', [0])` was
  true pre-8.0; the 8.0 fix covers the worst cases, but pass `strict: true` whenever the
  haystack contains non-scalars or the needle could be a numeric string.
- **Float equality:** never `===` two computed floats. Compare with an epsilon:
  `abs($a - $b) < PHP_FLOAT_EPSILON * max(abs($a), abs($b), 1)`, or work in integers
  (cents, microseconds).
- **Integer division:** `intdiv(7, 2)` is `3`; `7 % 2` is `1`; `7 / 2` is `3.5` — and
  `intdiv(PHP_INT_MIN, -1)` throws `ArithmeticError`. `%` converts operands to int
  first, so `7.5 % 2` is `1`, a classic source of off-by-halves.
- **String increment:** `++$s` on `'a9'` gives `'b0'` (Perl-style), and on 'Z' gives
  'AA' — occasionally useful, more often surprising. Avoid.

## Checking and narrowing types

Prefer `is_*` functions over `gettype()` comparisons; they read better and the engine
specializes them:

```php
<?php
declare(strict_types=1);

function size(mixed $value): int
{
    if (is_string($value)) {
        return strlen($value);   // static analyzers narrow $value to string here
    }
    if (is_array($value)) {
        return count($value);
    }
    throw new \InvalidArgumentException('Unsupported value');
}
```

`is_numeric` accepts numeric strings, floats, ints — use it only when all three are truly
fine. `is_iterable` covers arrays and `Traversable`. `is_callable` and `is_a($obj,
Class::class, allow_string: true)` for object/class checks.

`assert()` is a debugging tool governed by `zend.assertions` in php.ini — production
usually ships with it off. Do not use assertions for input validation; use real checks
that always run.
