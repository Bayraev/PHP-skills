# Security

PHP security is mostly about trusting nothing that crosses a boundary — request data,
file uploads, database results, environment. The defense-in-depth principle from the
manual's Security guide applies: layer controls so a single miss isn't a breach.

## Table of contents

- [SQL injection](#sql-injection)
- [XSS: escaping output](#xss-escaping-output)
- [Passwords and password hashing](#passwords-and-password-hashing)
- [File uploads](#file-uploads)
- [Command injection](#command-injection)
- [Filesystem paths and null bytes](#filesystem-paths-and-null-bytes)
- [Randomness](#randomness)
- [Sessions](#sessions)
- [Secrets handling](#secrets-handling)

## SQL injection

The only reliable control is prepared statements — SQL is sent to the server separately
from data, so data can never be parsed as SQL. See `database-pdo.md` for mechanics.

What still goes wrong on top of prepared statements:

- **Identifiers can't be parameterized.** Table/column/order-by names interpolated into
  SQL ("ORDER BY {$_GET['sort']}") remain injectable. Whitelist them:
  `in_array($sort, ['name', 'created_at'], true) ? $sort : 'name'`.
- **Multi-value IN clauses**: build one `?` per element — `WHERE id IN (` .
  `implode(',', array_fill(0, count($ids), '?')) . `)` — and bind each, rather than
  joining values into the string.
- **LIKE wildcards**: user input containing `%`/`_` isn't SQL injection, but it is a
  query-injection; escape with `addcslashes($term, '%_\\')` if wildcards weren't intended.

## XSS: escaping output

Escape at the point of output, for the destination context, not on input (stored data
may be validly rich in one context and hostile in another):

```php
<?php
declare(strict_types=1);

$template = '<p title="%s">%s</p>';
printf(
    $template,
    htmlspecialchars($user->title, ENT_QUOTES, 'UTF-8'),
    htmlspecialchars($user->bio, ENT_QUOTES, 'UTF-8'),
);
```

- `ENT_QUOTES` escapes both `"` and `'`; the `'` case is what attribute-injection needs.
- Always pass `'UTF-8'` — the default charset depends on ini settings.
- Contexts differ: HTML body, attributes, JavaScript (avoid embedding user data in JS at
  all; pass via `data-` attributes), URLs (`urlencode` for query parts), CSS (don't).
  A template system with auto-escaping (Twig) handles the common case; escaping manually
  is for the seams templates don't cover.
- `json_encode` of data into a `<script>` block still needs care: `</script>` inside a
  string terminates the tag. Use `JSON_HEX_TAG` or put the payload in a `data-`
  attribute.

## Passwords and password hashing

Use the password extension — `password_hash()` for storage, `password_verify()` for
checking, `password_needs_rehash()` for upgrading parameters at login time:

```php
<?php
declare(strict_types=1);

final class AuthService
{
    public function register(string $password): string
    {
        // PASSWORD_DEFAULT currently selects bcrypt; the constant is designed so PHP can
        // upgrade it later, which is why verify/needs_rehash work across algorithm changes.
        return password_hash($password, PASSWORD_DEFAULT);
    }

    public function verify(string $password, string $storedHash): bool
    {
        return password_verify($password, $storedHash);
    }

    public function upgradeHash(string $password, string $storedHash): ?string
    {
        if (password_needs_rehash($storedHash, PASSWORD_DEFAULT)) {
            return password_hash($password, PASSWORD_DEFAULT);
        }

        return null; // still current; keep the stored hash
    }
}
```

Facts that shape correct use (from the manual):

- The salt is generated automatically — never supply one; explicit salts are deprecated
  and ignored since 8.0. Everything needed to verify is *inside* the hash string.
- bcrypt (`PASSWORD_DEFAULT`/`PASSWORD_BCRYPT`) truncates input at 72 bytes and rejects
  NUL bytes. For bcrypt, pre-hash long passwords (e.g. `hash_hmac('sha256', $pw, $pepper)`,
  hex-encoded) — this also adds a pepper. For longer passwords generally, prefer
  `PASSWORD_ARGON2ID` when compiled in (options: `memory_cost` KiB, `time_cost` passes,
  `threads`).
- bcrypt `cost` defaults to 12 since 8.4 (was 10); tune so one hash takes ~250–350 ms on
  your hardware.
- Store the result in a column that tolerates growth (255 chars is the standard
  recommendation) — `PASSWORD_DEFAULT` output format can change across PHP versions.
- Don't limit password length below 72 bytes for bcrypt paths without pre-hashing;
  don't force silly composition rules — length wins.

Never: md5/sha1(+salt) for passwords, reversible encryption for passwords, or logging
passwords anywhere. Mark password parameters `#[\SensitiveParameter]` so a stray
exception trace cannot leak them.

## File uploads

Every field of `$_FILES` except `size` and `tmp_name` is attacker-controlled, including
`name` (the client filename) and `type` (the declared MIME). Also, PHP itself trusts
the multipart boundary only for POST — 8.4's `request_parse_body()` exists for other
methods.

```php
<?php
declare(strict_types=1);

final class UploadGuard
{
    private const ALLOWED_EXTENSIONS = ['pdf', 'png', 'jpg'];

    public function store(array $file): string
    {
        if (($file['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_OK) {
            throw new \RuntimeException('Upload failed');
        }
        if (!is_uploaded_file($file['tmp_name'])) {
            throw new RuntimeException('Not an uploaded file');
        }

        $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
        if (!in_array($ext, self::ALLOWED_EXTENSIONS, true)) {
            throw new RuntimeException("Extension {$ext} not allowed");
        }

        // random server-side name; never reuse the client filename (it can contain
        // traversal sequences, control chars, or collide)
        $target = bin2hex(random_bytes(16)) . '.' . $ext;

        return $target; // caller: move_uploaded_file($file['tmp_name'], $dir . '/' . $target)
    }
}
```

- Check `error` first — `UPLOAD_ERR_OK` means success; other codes mean size exceeded
  (`UPLOAD_ERR_INI_SIZE`/`FORM_SIZE`), partial upload, or missing file.
- `is_uploaded_file()` / `move_uploaded_file()` are the only safe ways to touch the
  temp file; they verify the file actually came from a POST upload.
- Store outside the webroot (or in object storage) with generated names, so a `.php`
  disguise (`shell.php.jpg` tricks) never becomes executable code and predictable names
  can't be enumerated.
- Validate content when it matters (finfo MIME, image dimensions, virus scan) — extension
  checks are routing, not validation.
- Enforce sizes in PHP (`$file['size']`, post-decode) in addition to ini limits.

## Command injection

Any shell invocation containing untrusted data is a hole — quotes inside data defeat
manual quoting. In order of preference:

1. Avoid the shell: use native PHP APIs (`mkdir`, `copy`, `unlink`, ZipArchive) instead
   of `exec('mkdir ...')`.
2. `proc_open()` with an argument array (bypasses shell parsing entirely) and explicit
   pipes; capture exit status from `proc_close`.
3. If a shell string is unavoidable: `escapeshellarg($userInput)` every fragment of
   external data (`escapeshellcmd` is for whole commands and is easy to misuse — prefer
   arg-level), and allow-list any structure.

```php
<?php
declare(strict_types=1);

$process = proc_open(
    ['git', 'log', '--oneline', '-n', '10'],
    [1 => ['pipe', 'w'], 2 => ['pipe', 'w']],
    $pipes,
);
if (!is_resource($process)) {
    throw new \RuntimeException('Cannot spawn git');
}
$output = stream_get_contents($pipes[1]);
fclose($pipes[1]);
fclose($pipes[2]);
$exitCode = proc_close($process);
```

Related: `include`/`require` of a path derived from user input is remote-code-execution
by design — never do it; `eval` likewise.

## Filesystem paths and null bytes

- Treat any user-influenced path with suspicion: prefix-fixed does not mean contained.
  `../` sequences survive naive string checks; verify with
  `strpos(realpath($path), $baseDir) === 0` style checks (realpath resolves traversal),
  or use allow-listed filenames.
- PHP functions reject paths containing NUL bytes (`\0`) since 5.3.4, but your own
  string code may not — validate binary-safely when data can be binary.
- `allow_url_fopen` makes file functions accept URLs — if you never intend that, its ini
  default scrutiny belongs in your deployment review; `file_get_contents` on a
  user-supplied "path" can become SSRF.

## Randomness

`random_bytes()` / `random_int()` use a CSPRNG — use them for anything security
related: tokens, filenames, nonces, idempotency keys. `rand()`, `mt_rand()`,
`uniqid()`, `str_shuffle()` are predictable and have no place in security contexts.
For public identifiers, prefer `bin2hex(random_bytes(16))` over `uniqid`-anything.
Comparing secrets: `hash_equals($known, $given)` is timing-safe; `===` is not.

## Sessions

- Call `session_regenerate_id(true)` on privilege change (login, re-auth) — a session
  that kept its ID across login enables session fixation.
- Set cookie flags: `session_start(['cookie_httponly' => true, 'cookie_secure' => true,
  'cookie_samesite' => 'Lax'])` (or ini), and enable `session.use_strict_mode=1` so
  attacker-chosen session IDs are rejected.
- Store a minimal session; keep the juicy data in the database keyed by session id.
- Rotate/expire: idle timeouts server-side, not just cookie expiry.

## Secrets handling

- Read credentials from environment variables (or a secrets manager); never hardcode,
  never commit `.env` (gitignore it, commit `.env.example` with placeholders).
- Redact in logs and traces: `#[\SensitiveParameter]`, and strip Authorization headers
  from request logging.
- Webhooks/APIs sign payloads — compare signatures with `hash_equals`, and use
  `hash_hmac('sha256', $body, $secret)` (raw bytes, then `hash_equals`) rather than
  simple hashes.
- If a secret lands in VCS history, rotate it — history is forever.
