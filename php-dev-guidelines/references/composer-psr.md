# Composer, autoloading, and PSR standards

Composer is the dependency and autoloading manager; PSR standards are the
interoperability layer. Get both right and any PHP project (and any developer/agent)
can navigate the codebase.

## composer.json essentials

```json
{
    "name": "acme/shop-core",
    "description": "Domain core for the shop service",
    "type": "library",
    "license": "proprietary",
    "require": {
        "php": "^8.4",
        "ext-pdo": "*",
        "monolog/monolog": "^3.7"
    },
    "require-dev": {
        "phpunit/phpunit": "^12.0"
    },
    "autoload": {
        "psr-4": {"Acme\\Shop\\": "src/"}
    },
    "autoload-dev": {
        "psr-4": {"Acme\\Shop\\Tests\\": "tests/"}
    },
    "config": {
        "sort-packages": true,
        "optimize-autoloader": true
    },
    "scripts": {
        "test": "phpunit",
        "lint": "find src -name '*.php' -exec php -l {} +",
        "ci": ["@lint", "@test"]
    },
    "scripts-descriptions": {
        "ci": "Runs all checks used in CI"
    }
}
```

Points that matter:

- **Commit `composer.lock`** for applications (exact reproducible installs; `composer
  install` respects it). Libraries omit the lock and constrain loosely.
- `composer validate` checks schema and common mistakes (run it in CI). `composer
  audit` checks installed packages against the security advisory database.
- Declare PHP version *and* required extensions (`ext-pdo`) in `require` — deployment
  environments and `composer check-platform-reqs` read these.
- `scripts` give the project one vocabulary (`composer test`) — prefer them over
  README-only commands; `@lint`-style references compose.

## Version constraints

| Constraint | Means | Example |
|---|---|---|
| `^8.4` | `>=8.4.0 <9.0.0` — minor+patch allowed, no major | the usual choice |
| `^3.7` | `>=3.7.0 <4.0.0` | same, per-package |
| `~3.7.2` | `>=3.7.2 <3.8.0` — tilde stops at the next significant level given | rare vs caret |
| `1.2.3` | exactly 1.2.3 | pin only with reason |
| `>=1.2 <2.0-dev` | explicit range | rare, legacy |
| `dev-main` | branch | local dev only |

Prefer `^` on stable releases. For the `php` platform: `^8.4` allows 8.4 and 8.5 —
matching reality where feature use is gated by the constraint. Use
`"config": {"platform": {"php": "8.4.99"}}` to pin dependency resolution to a floor
version regardless of the local PHP.

## Autoloading (PSR-4)

PSR-4 maps a namespace prefix to a directory; class `Acme\Shop\Order\LineItem` lives at
`src/Order/LineItem.php` given the mapping above. Rules that keep the autoloader happy:

- One class per file, file path mirrors the namespace after the prefix, class name =
  file basename. Case-sensitive even on case-insensitive filesystems (CI on Linux will
  catch what macOS hid).
- After adding classes or changing mappings: `composer dump-autoload`. Production
  deployments use `--optimize` (or `--classmap-authoritative` for immutable builds) to
  convert runtime path lookups into a classmap.
- `autoload.files` (e.g. a `functions.php` with helper functions — functions can't be
  autoloaded) and `autoload.classmap` (generated paths, useful for non-PSR-4 legacy)
  are the escape hatches; prefer PSR-4 for anything new.
- Start every file with `<?php` and `declare(strict_types=1);`; do NOT close with `?>`
  in PHP-only files (any trailing whitespace after it becomes output — headers/JSON
  break mysteriously).

## PSR-1 / PSR-12 style in brief

The baseline most tools (PHP-CS-Fixer, PHPStan's taste, reviewers) assume:

- Files use `<?php` only, UTF-8, no BOM; no side effects in files that declare symbols
  (a class file shouldn't also send headers or write files — PSR-1).
- Class names `StudlyCaps` (map to filename), methods `camelCase`, constants
  `UPPER_SNAKE` (enum cases may be `StudlyCaps`), properties whichever of
  `$StudlyCaps`/`$camelCase`/`$snake_case` the project picked — consistently.
- 4 spaces, no tabs; soft line limit 120; opening braces on the same line for control
  structures, next line for classes/methods; `elseif` not `else if`; `switch`/`match`
  indented one level per structure.
- `use` statements at top, one per line, alphabetical; import classes instead of
  fully-qualified names inline (`use Acme\Shop\Order;` then `new Order()`).
- Declare strict types as shown, `declare(strict_types=1);` with a space after the
  semicolon... exact `declare` formatting varies by tool config — follow the repo's.
- Type order in signatures: visibility, then type, then name; nullable as `?Type` or
  `Type|null` consistently; return type on its own line only when the signature is long.

PER (Coding Style) supersedes/extends PSR-12 as the living standard; tool configs
(php-cs-fixer `.php-cs-fixer.dist.php`) are the operative authority — match the repo's
config, and never mix styles in one diff.

## Working on a Composer project

- Install: `composer install` (uses lock) — then `composer dump-autoload -o` in CI
  images only; dev needs the fresh lookup.
- Add a dependency: `composer require vendor/pkg` (picks a constraint for you); dev
  tooling with `--dev`.
- Update deliberately: `composer update vendor/pkg` for one package, full `composer
  update` with tests following; inspect `composer.lock` diffs in review like source.
- `composer outdated --direct` shows available majors of direct deps; `composer
  why vendor/pkg` explains who pulled a transitive package in.
