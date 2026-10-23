# Performance

Most PHP performance work is algorithmic, not exotic: the hot path does O(n²) array
work, or loads a million rows into memory. Measure before optimizing —
`hrtime()` for wall time, `memory_get_usage()`/`memory_get_peak_usage()` for memory —
and fix what measurement indicts.

## First: deployment basics

- **OPcache** compiles scripts once per server, keeping bytecode in shared memory. For
  web serving it should be enabled with sensible production settings
  (`opcache.validate_timestamps=0` in immutable deployments — revalidate in dev! —
  `opcache.memory_consumption` sized to the codebase, `opcache.max_accelerated_files`
  above your file count). Without OPcache every request re-parses every file.
- **JIT** (`opcache.jit=tracing`, `opcache.jit_buffer_size>0`) further compiles hot
  bytecode to machine code. It shines on CPU-bound loops (math, parsing) and is usually
  neutral for typical I/O-bound web requests — enable it when profiling shows CPU-bound
  work, not as a ritual. Compare with and without on realistic load.
- **Preloading** (`opcache.preload`) can eliminate autoload overhead in long-lived
  servers; treat as deployment tuning after the basics.
- CLI scripts: `opcache.enable_cli=1` helps only scripts that run long enough to
  amortize compilation; one-shot scripts don't benefit.

## The array costs that dominate ordinary code

```php
<?php
declare(strict_types=1);

$ids = range(1, 100_000);

$set = array_flip($ids);                      // O(1) lookups instead of O(n) scans
$ok = isset($set[57]);

$chunks = array_chunk($ids, 500);             // bounded batches beat one giant loop
```

- `in_array` / `array_search` on a list scan linearly. Inside a loop over n items,
  that's O(n²). Flip once, `isset` millions.
- `$list[] = $x` and `array_push` are both fine; `array_merge` in a loop is not — it
  copies the whole array each iteration. Accumulate and merge once, or use `...`
  spread: `$all = [...$a, ...$b];`.
- `count($items)` in a `for` condition re-evaluates every iteration; hoist it or use
  `foreach`, which is also the fastest general iteration.
- Nested lookups by key are hash lookups — build indexed maps once
  (`$byId = array_column($rows, null, 'id')`) instead of scanning for related rows.
- References (`foreach ($arr as &$v)`) are not a speed tool; they usually make things
  slower and definitely make code error-prone (stray `$v` after the loop). Copying
  arrays is copy-on-write — cheap until modified.

## Memory and streaming

- `fetchAll()` on a big result set, `file_get_contents` on a big file, and
  `json_decode` of a huge payload all materialize everything at once. Use PDO cursor
  iteration (see database reference), `fopen`+`fgets`/stream filters, and streaming
  JSON/XML parsers when data is big *or* unbounded (user uploads, imports).
- Generators (`yield`) let pipelines process lazily with constant memory:

```php
<?php
declare(strict_types=1);

function readLines(string $path): \Generator
{
    $handle = fopen($path, 'rb');
    try {
        while (($line = fgets($handle)) !== false) {
            yield rtrim($line, "\r\n");
        }
    } finally {
        fclose($handle);
    }
}

$count = 0;
foreach (readLines('/var/log/big.log') as $line) {
    $count += str_contains($line, 'ERROR') ? 1 : 0;
}
```

Generators are single-pass: iterating twice re-runs the source; wrap in
`iterator_to_array()` (memory!) or design for one pass. Trade-off is real: generator
pipelines are slower per element than tight array loops — choose them for memory
constraints, not micro-speed.

- Free big variables before long tails of work (`unset($bigPayload)`), and avoid
  keeping request-scoped caches in statics on long-running workers (Swoole/RoadRunner
  style) — memory grows and stale data leaks between requests.

## Strings and misc

- Concatenation in a loop (`$s .= $chunk`) is quadratic-ish for large strings in some
  engines' mental models — in PHP it's amortized but still heavier than collecting
  parts in an array and `implode('', $parts)` once. Do the latter in hot loops.
- `sprintf` is slower than concatenation; that only matters in tight loops —
  readability first elsewhere.
- Regex (`preg_*`) for parsing is fine until profiles show otherwise; literal-string
  functions (`str_contains`, `explode`) are faster and clearer when they suffice.
- `sleep`-based throttling and retries: use exponential backoff with jitter, and cap
  total time — in workers, unbounded retries are an outage amplifier.
- Caching (APCu, Redis, static in-request memoization of pure functions) is the fix
  for repeated expensive *pure* computation; never cache impure results (queries with
  different parameters) under one key.

## When it's actually slow

Profile before restructuring: OpenTelemetry or New Relic for request-level hot paths,
`spy`/telemetry for workers, `phpbench` for micro-benchmarks of candidate hotspots.
The counter-intuitive wins are almost always: fewer queries (batch/join), fewer
round-trips to external services, and smaller payloads — not micro-optimizing PHP.
