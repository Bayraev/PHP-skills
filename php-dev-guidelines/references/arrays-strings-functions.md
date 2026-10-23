# Arrays, strings, and function idioms

Arrays are PHP's workhorse data structure (ordered hash map). Most performance and
correctness bugs in ordinary PHP come from array and string handling, so the idioms
matter.

## Array idioms

**Construction and "with" semantics.** Spread with string keys (8.1+) gives clean
immutable-style overrides; later keys win:

```php
<?php
declare(strict_types=1);

$defaults = ['host' => 'localhost', 'port' => 3306, 'timeout' => 5];
$config   = [...$defaults, 'port' => 3307, 'user' => 'app'];
// ['host' => 'localhost', 'port' => 3307, 'timeout' => 5, 'user' => 'app']
```

**Destructuring** beats repeated index access, and works in `foreach`:

```php
<?php
declare(strict_types=1);

foreach ($users as [$id, $name]) {
    printf("%d: %s\n", $id, $name);
}
foreach ($points as ['x' => $x, 'y' => $y]) {
    $area[] = $x * $y;
}
```

**map/filter/reduce** over hand-rolled loops — they declare intent:

```php
<?php
declare(strict_types=1);

$emails = array_map(strtolower(...), $rawEmails);          // first-class callable (8.1)
$adults = array_filter($users, fn (User $u): bool => $u->age >= 18);
$total  = array_reduce($items, fn (int $sum, LineItem $i): int => $sum + $i->qty, 0);
```

Watch the edges: `array_filter` preserves keys — reindex with `array_values()` when the
result is used as a list; `array_map` with multiple arrays iterates to the longest one,
passing `null` for missing elements — avoid unequal lengths rather than relying on that.
A `fn` arrow function captures outer variables by value automatically and cannot have a
body longer than one expression — use a full `function` when logic grows.

**Sorting.** `sort`/`rsort`/`usort` reindex; `asort`/`uasort`/`ksort`/`uksort` preserve
keys. All sorts are stable since 8.0, so equal elements keep their order — you can stop
adding tiebreakers for determinism alone. Custom comparator:

```php
<?php
declare(strict_types=1);

usort($files, fn (SplFileInfo $a, SplFileInfo $b): int => $a->getMTime() <=> $b->getMTime());
```

Return `-1|0|1` (the spaceship `<=>` does exactly that). Returning `true`/`false` from a
comparator is invalid since 8.0 and throws.

**Membership tests.** `in_array()` on a list is a linear scan; `isset($set[$key])` on an
associative array is O(1). For repeated lookups, build a set once:
`$set = array_flip($ids);` then `isset($set[$needle])`. This is the most common
accidental O(n²) in PHP codebases.

**List and map helpers worth knowing:** `array_key_exists` (true even for null values —
`isset` is false for them), `array_column`, `array_combine`, `array_slice`, `array_splice`
(in-place), `array_unique` (loose comparison — sort flags available), `array_chunk`,
`array_sum`/`array_product`, `+` array union (left wins, additions only) vs
`array_merge` (reindexes numeric keys, right wins for strings).

## String handling

**Bytes vs characters.** PHP strings are byte arrays; `strlen` counts bytes. Anything
user-facing in a multibyte encoding (UTF-8 content) needs the `mb_*` equivalents:
`mb_strlen`, `mb_substr`, `mb_strtoupper`, `mb_strpos`. Mixing families corrupts
multibyte text silently — decide per project and stay consistent. Also common:
`htmlspecialchars` for output (see security reference), `sprintf`/`vsprintf` for
formatting, `number_format` for display numbers.

**Modern search functions (8.0+) replace `strpos !== false` gymnastics:**

```php
<?php
declare(strict_types=1);

if (str_contains($haystack, 'needle')) {}
if (str_starts_with($filename, 'upload_')) {}
if (str_ends_with($email, '@example.com')) {}
```

**Interpolation and escaping:** double-quoted `"{$user->name}"` interpolates; single
quotes never do. `"${var}"` interpolation was removed in 8.3 — use `"{$var}"`. For
complex expressions, `sprintf('%s (%d)', $name, $count)` beats nested concatenation for
readability.

**Splitting:** `explode(',', $csv)` — pass the `$limit` argument for user input so a
malicious megabyte of separators can't balloon memory: `explode(',', $line, 5)`.
`preg_split` for pattern splits.

**str_pad/trim family:** `trim($input, " \t\n\r")` — note `trim`'s second argument is a
character list, not a substring. To strip a known *substring*, use a regex or
`substr` checks.

## match, and flow as expressions

`match` (8.0+) is an expression with strict comparison and no fall-through; it throws
`UnhandledMatchError` when nothing matches — add a default arm unless unhandled truly
means bug:

```php
<?php
declare(strict_types=1);

$verb = match ($method) {
    'GET', 'HEAD' => 'read',
    'POST' => 'create',
    default => throw new \InvalidArgumentException("Unsupported method {$method}"),
};
```

Each arm is a single expression; multiple conditions per arm separated by commas. For
block bodies, use classic `switch` or call a method per arm. `throw` being an expression
(8.0) makes it usable in arrow functions and defaults: `$user = $users[0] ??
throw new \RuntimeException('No users');`.

## JSON

Decode and encode with exceptions — the `null`-return protocol of `json_decode` is
indistinguishable from decoding the literal `null`:

```php
<?php
declare(strict_types=1);

try {
    $payload = json_decode($body, associative: true, flags: JSON_THROW_ON_ERROR);
} catch (\JsonException $e) {
    throw new \InvalidArgumentException('Malformed JSON body', previous: $e);
}
```

`json_validate($body)` (8.3+) answers only "is this valid JSON" and is cheaper than
decoding; use it when you won't decode (validating stored blobs, gatekeeping) and skip
it when you will — validating then decoding parses twice. On 8.2, polyfill with
`json_decode(..., JSON_THROW_ON_ERROR)` in a try/catch. `json_decode` flags worth
knowing: `JSON_BIGINT_AS_STRING` (large ints as strings instead of float loss),
`JSON_INVALID_UTF8_IGNORE`.

## Callable handling

Pass callables as first-class callables (`strtoupper(...)`, `$this->handler(...)`) rather
than strings or `[$obj, 'method']` arrays — the syntax validates at parse time and works
with static analysis. `is_callable($cb)` checks callability; `Closure::fromCallable`
is now rarely needed. To invoke a variable callable: `$cb($arg)` directly, or
`$result = $callable(...$args)` with spread.

## Generators bridge

`array_*` functions do not accept generators — `iterator_to_array($gen)` converts when
needed, but if you find yourself converting, consider keeping the pipeline lazy
(see performance reference for when generators pay off).
