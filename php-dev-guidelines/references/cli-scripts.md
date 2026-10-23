# CLI scripts and shells

PHP CLI scripts are first-class: workers, cron jobs, import/export tools, dev
utilities. They differ from web PHP in lifecycle (no request timeout by default, no
SAPI superglobals like `$_SESSION`), which is where the gotchas live.

## Executable scripts

```php
#!/usr/bin/env php
<?php
declare(strict_types=1);

if (PHP_SAPI !== 'cli') {
    exit("CLI only\n");
}

require __DIR__ . '/vendor/autoload.php';
```

- Shebang line first, `chmod +x`, and the file runs like any shell command.
  The shebang must be line 1; PHP ignores it inside the `<?php` context correctly when
  it precedes the opening tag exactly like this.
- Guard `PHP_SAPI !== 'cli'` for scripts that must never be reached via a webserver
  (a script dropped in a web root would otherwise just execute on request).
- Autoload via composer when the script is part of a project; standalone tools may
  bundle a classmap.

## Arguments and options

```php
<?php
declare(strict_types=1);

$options = getopt('v', ['env:', 'dry-run'], $optind);
$verbose = isset($options['v']);
$env = $options['env'] ?? 'development';
$dryRun = isset($options['dry-run']);
$args = array_slice($argv, $optind);   // positionals after options
```

- `$argv[0]` is the script name; real arguments start at index 1. `$argc` is the count.
  Register_argc_argv is on for CLI.
- `getopt` handles `-v` flags and `--env=prod` / `--env prod` long options (`:` marks a
  required value). For anything multi-command, use a real console library
  (symfony/console) — argument parsing, help text, and completion come with it.
- Validate inputs early and print usage to `STDERR`:

```php
<?php
declare(strict_types=1);

if (count($argv) < 2) {
    fwrite(STDERR, "Usage: {$argv[0]} <input.csv>\n");
    exit(1);
}
```

`STDIN`, `STDOUT`, `STDERR` constants exist only in CLI SAPI — as do `interactive_*`
functions. Reading piped input: `stream_get_contents(STDIN)`, or line-by-line with
`fgets(STDIN)` for streaming filters. Check `posix_isatty(STDIN)` (requires
ext-posix) if behavior should differ between piped and terminal input.

## Exit codes and output discipline

- `exit(0)` on success; non-zero on failure — shells, cron, and CI keys off this.
  1 for generic failure, 2 for usage errors (getopt convention).
- Exit codes are a byte: anything ≥ 256 wraps (mod 256); pass small stable codes and
  put detail in the output.
- Progress and diagnostics go to `STDERR`, data to `STDOUT`, so pipes compose:
  `import.php < in.csv > out.json 2> import.log` works as users expect.
- Never echo into pipelines that emit structured data — a stray "Done!" corrupts the
  JSON downstream.

## Long-running scripts

- No request timeout applies (`max_execution_time` is 0 for CLI by default) — enforce
  your own budget: `set_time_limit(300)` around risky phases, deadline checks in
  loops (`hrtime(true)` against a computed cutoff).
- Memory only grows if you let it: unset accumulators per item, monitor
  `memory_get_usage(true)` and bail/restart (supervisor pattern) past a threshold.
  Fork/worker managers restart children on leaks.
- Watch for duplicate script instances on the same job: a lock
  (`fopen($lockFile, 'c')` + `flock($lockFile, LOCK_EX | LOCK_NB)`) is the standard
  cheap guard for cron tasks.
- Retry external failures with capped exponential backoff + jitter; one long `sleep`
  and infinite retries both cause incidents.

## Process control (pcntl) and signals

`ext-pcntl` (Unix CLI only — not on Windows, not in web SAPI) enables forking and
signal handling:

```php
<?php
declare(strict_types=1);

if (!function_exists('pcntl_signal')) {
    fwrite(STDERR, "pcntl not available\n");
    exit(1);
}

$running = true;
pcntl_async_signals(true);   // dispatch callbacks immediately, no ticks needed
pcntl_signal(SIGTERM, function () use (&$running): void {
    $running = false;        // graceful stop, not die() mid-write
});

while ($running) {
    $job = $queue->reserve(seconds: 5);
    if ($job !== null) {
        $job->process();
    }
}
```

- Handle SIGTERM/SIGINT with a flag and finish the current item — containers and
  supervisors send SIGTERM before SIGKILL; dying mid-item corrupts state.
- `pcntl_fork` children inherit the parent's DB connections — reconnect
  (or establish connections *after* forking) or you'll share socket state mysteriously.
- `proc_open` (see security reference) is the portable way to run and manage child
  processes; `exec()` replaces the current process — usually not what you meant.

## Cron and environment

- Cron's environment is minimal: PATH and env vars you take for granted interactively
  may be absent. Use absolute paths (binary + files) and load `.env` explicitly.
- Errors from cron should page someone: `log_errors` with `error_log` pointing at a
  monitored destination, or a wrapper that alerts on non-zero exit.
- Lock or make idempotent anything cron can overlap (see flock above).
