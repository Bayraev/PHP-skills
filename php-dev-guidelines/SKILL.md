---
name: php-dev-guidelines
description: Production-grade PHP 8.x standards for writing, reviewing, modernizing, and debugging PHP code. Use this skill for ANY task that touches .php files, composer.json, phpunit.xml, or PHP tooling — writing or editing PHP, fixing PHP bugs or errors, code review, upgrading or migrating PHP versions (7.x → 8.x, 8.3 → 8.4), using PHP 8.1–8.5 features (enums, readonly, property hooks, asymmetric visibility, pipe operator), PDO/database code, PHP security (SQL injection, XSS, password hashing, file uploads), Composer/PSR-4/PSR-12, PHPUnit testing, OPcache/performance, or CLI scripts — even for "just a small change" to an existing PHP file, and even when the user only mentions PHP, Laravel, Symfony, WordPress plugin code, or a PHP error message without asking for guidelines.
---

# PHP Development Guidelines

Standards and workflows for production PHP 8.x code: modern language use, strict typing,
OOP design, security, database access, Composer, testing, performance, and CLI scripts.
Grounded in the official PHP manual (php.net); version notes below were verified against
the manual's migration and supported-versions pages as of Oct 2026.

## Core workflow

Apply this loop whenever you write or modify PHP:

1. **Determine the target PHP version before writing code.**
   Check `composer.json` (`"require": {"php": "..."}`) and `php -v`. Feature availability
   differs per release (see the version snapshot below); code that is elegant on 8.5 is a
   parse error on 8.3. When the target is ambiguous, prefer the newest version the project
   already requires and note anything that needs a lower-version fallback.
2. **Start every file with `<?php` and `declare(strict_types=1);`** as the first statement
   (nothing may precede it, not even a docblock). Rationale: in weak mode PHP silently
   coerces `"5"` to the int `5` and truncates `5.9` to `5` at call boundaries —
   numeric-looking data slips through as the wrong type, and the bug surfaces far from
   its cause. With strict types, wrong types fail loudly at the exact call site.
   Note strictness is enforced per-file on the *calling* side, so it must be in every file,
   not just the entry point.
3. **Lint early, lint often.** After editing, run `php -l <file>` (or loop over changed
   files). It catches parse errors in milliseconds without executing anything — even code
   using 8.4/8.5 syntax you may not have memorized.
4. **Validate Composer state when touching dependency or autoload concerns:**
   `composer validate` and `composer dump-autoload` after autoload/classmap changes;
   `composer audit` for known-vulnerable dependencies.
5. **Run the test suite** (`vendor/bin/phpunit`) after any non-trivial change. If a project
   has no tests, at minimum exercise the changed code path with `php -r` or a throwaway
   script rather than assuming it works.
6. **Match the existing style** of the file (PSR-12 is the baseline; see
   `references/composer-psr.md`). Consistency beats personal preference.

## Quick decision table — which reference to read

| Situation | Read |
|---|---|
| Deciding which language features are safe to use; upgrading between PHP versions | `references/modern-php.md` |
| Type declarations, `strict_types`, union/intersection/DNF types, coercion traps | `references/types-and-strictness.md` |
| Array manipulation, string handling, function/parameter idioms, JSON | `references/arrays-strings-functions.md` |
| Designing classes, interfaces, traits, enums, DTOs, property hooks, static-analysis-friendly code | `references/oop-design.md` |
| Exceptions vs errors, what to throw/catch, error reporting configuration | `references/error-handling.md` |
| Any code touching user input, SQL, output, passwords, uploads, shells, or secrets | `references/security.md` |
| Database access with PDO: connections, prepared statements, transactions | `references/database-pdo.md` |
| composer.json, autoloading, version constraints, PSR-12 style, scripts | `references/composer-psr.md` |
| Writing or reviewing PHPUnit tests, mocks, data providers | `references/testing-phpunit.md` |
| Profiling, OPcache/JIT, loops, memory, generator-based streaming | `references/performance.md` |
| Command-line scripts: argv, exit codes, signals, long-running workers | `references/cli-scripts.md` |

Read only the references the task needs — they are self-contained. When a task spans
several (e.g., "add an endpoint that saves to the database"), read security and database
at minimum, because that is where irreversible damage lives.

## Version snapshot (verified Oct 2026)

| Branch | Status on 2026-10-23 | Active until | Security until |
|---|---|---|---|
| 8.5 | Current stable (released 20 Nov 2025) | 2027-12-31 | 2029-12-31 |
| 8.4 | Active support | 2026-12-31 | 2028-12-31 |
| 8.3 | Security fixes only | (ended 2025-12-31) | 2027-12-31 |
| 8.2 | Security fixes only, EOL 2026-12-31 | (ended 2024-12-31) | 2026-12-31 |

Implications: 8.2 is weeks from end-of-life — treat any 8.2-only deployment as an
upgrade-in-progress. New projects should target 8.4+ (8.3+ minimum for library packages
that must support the security-only branch). Re-verify at
https://www.php.net/supported-versions.php because these dates move.

Minimum-version cheat sheet for commonly wanted features:

- 8.1: enums, readonly properties, first-class callable syntax `f(...)`, intersection
  types, `never`, `new` in initializers, array spread with string keys
- 8.2: readonly classes, DNF types, standalone `null`/`false`/`true` types, constants in
  traits, `#[\SensitiveParameter]`
- 8.3: typed class constants, `#[\Override]`, `json_validate()`, readonly re-clone
  amendments, dynamic class-constant fetch `C::{$name}`
- 8.4: property hooks, asymmetric visibility (`public private(set)`), lazy objects,
  `#[\Deprecated]`, `new` chaining without parentheses, `RoundingMode` enum
- 8.5: pipe operator `|>`, closures/first-class callables in constant expressions,
  `#[\NoDiscard]` + `(void)` cast, attributes on constants, `clone($obj, withProperties: ...)`

## Non-negotiable rules with rationale

These few rules exist because violating them produces bugs or vulnerabilities that are
invisible in review:

- **Prepared statements for all SQL involving variables.** String interpolation into SQL
  is the single most common PHP vulnerability. There is no "safe" manual escaping.
- **Escape on output, not on input:** `htmlspecialchars($value, ENT_QUOTES, 'UTF-8')`
  when placing untrusted data in HTML. Encoding depends on destination context — do it at
  the boundary where the context is known.
- **`password_hash()` / `password_verify()`** for passwords. Anything hand-rolled
  (md5, sha1, "salt" columns) is broken by design.
- **No secrets in code or VCS.** Read from environment variables; `.env` files stay out of
  git via `.gitignore`. Assume anything committed is public eventually.
- **Fully typed public APIs.** Every parameter, return type, and property gets a real type
  so the runtime and static analyzers can catch what tests miss.
- **One behavior per test, no assertions on mock call counts you don't care about** —
  tests that over-specify break on refactoring and teach nothing.

## Style rules (short, with reasons)

- Constructor property promotion over hand-written getters/setters boilerplate; it keeps
  construction logic in one place and pairs naturally with `readonly`.
- `match` over `switch` for value selection: no fall-through, strict `===` comparison,
  expression form, and the compiler enforces exhaustiveness via `UnhandledMatchError`.
- Named arguments when positional meaning is unclear — `array_slice($a, offset: 2,
  length: 5)` documents intent; on obvious calls they are noise, so use judgment.
- `final` on classes by default; inheritance is a design decision the child makes, and
  `final` pushes toward composition, which is easier to test and evolve. Remove it
  deliberately when extension is part of the API.
- Prefer pure functions/services (deterministic input → output) over globals and
  `static` state; pure code is trivially testable and immune to ordering bugs.
- Throw exceptions for failures, return values for results — avoid `false`/`null` sentinel
  returns that force `===` checks at every call site and silently propagate when missed.

## What this skill does not cover

Framework-specific conventions (Laravel, Symfony, WordPress internals) — apply these
guidelines underneath framework conventions. When framework docs and general PHP best
practice conflict on framework territory, the framework wins.
