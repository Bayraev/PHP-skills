# PHP-skills

Agent skills for PHP development, grounded in the [official PHP manual](https://www.php.net/manual/en/).

## Skills

| Skill | Purpose |
|---|---|
| [`php-dev-guidelines`](php-dev-guidelines/) | Production-grade PHP 8.x standards: modern syntax (8.1–8.5), strict typing, OOP design, security, PDO, Composer/PSR, PHPUnit, performance, CLI scripts |

### php-dev-guidelines structure

- `SKILL.md` — hub: workflow (target version → strict_types → `php -l` → composer validate → tests), decision table routing to references
- `references/` — 11 topic files loaded on demand: modern-php, types-and-strictness, arrays-strings-functions, oop-design, error-handling, security, database-pdo, composer-psr, testing-phpunit, performance, cli-scripts

Facts were verified against php.net (manual, migration guides, supported-versions page) as of Oct 2026. All PHP samples pass `php -l` on PHP 8.5.

## Install

Copy or symlink the skill folder into your agent's skills directory, e.g.:

```bash
ln -s "$(pwd)/php-dev-guidelines" ~/.claude/skills/php-dev-guidelines
```

Built with [Anthropic's skill-creator](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator/skills/skill-creator); every skill passes its `quick_validate` check.
