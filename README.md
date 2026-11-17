# WordPress SQLi Lab (local only)

A deliberately vulnerable WordPress site for learning SQL injection (SQLi)
safely, then fixing it with `$wpdb->prepare()`.

> **Current state: VULNERABLE.** Phase 1 is live — the plugin concatenates raw
> user input into a SQL query on purpose. Phase 2 (the fix) is planned and has
> **not** been applied. Do not copy this code into a real project.

## What this lab is

- WordPress 7.1.4 running on the bundled PHP 8.3 CLI with a SQLite drop-in.
- One page, `?pagename=sqli-lab`, with a `[sqli_lab_search]` shortcode that looks
  up posts by exact title.
- The generated SQL is rendered on the page so you can see exactly what ran,
  along with any DB error and the returned rows.
- A secret post (`s3cr3t-note`) holds the flag, plus a WordPress admin user whose
  password hash can be dumped with a UNION payload.
- No Docker, no database server, no web server, and no restart policy. The PHP
  built-in server binds only to `127.0.0.1:8080`.

**Use it locally only.** Do not expose it to a network or deploy it. The SQL
concatenation in the plugin is an intentional violation of WordPress's "prepare
your queries" security rule — it is the lab target.

Part of the [Web Security Labs](../README.md) map. All six labs are currently
`VULNERABLE / UNFIXED`.

## Requirements

- Linux with `bash`
- The bundled PHP 8.3 runtime and SQLite extensions (in `runtime/`)
- `curl` (only for `exploit.sh`)

No system PHP, Docker, or database server is required. `start.sh` regenerates
`runtime/php.ini` and sets `LD_LIBRARY_PATH` / `PHP_INI_SCAN_DIR` for the bundled
runtime on every launch.

## Quick start

From this directory:

```bash
./start.sh    # starts the lab at http://127.0.0.1:8080/?pagename=sqli-lab
./stop.sh     # stops the lab
```

`start.sh` writes the server PID to `.server.pid`; `stop.sh` reads it and stops
the process. Running `start.sh` while the lab is already up is a no-op. Pressing
`Ctrl-C` in the `start.sh` terminal also stops the server.

The lab is bound to loopback only (`127.0.0.1`), so it is not reachable from
other machines. Nothing restarts automatically.

## URLs and credentials

| Thing | Value |
|---|---|
| Lab page | `http://127.0.0.1:8080/?pagename=sqli-lab` |
| Admin login | `http://127.0.0.1:8080/wp-login.php` |
| Admin user | `admin` |
| Admin password | `LabAdmin!234` |
| Secret post | `s3cr3t-note` → `FLAG{wordpress-sqli-lab-lab-only}` |

## Working through the lab

1. Start the lab and open the lab page.
2. Search for `s3cr3t-note` and observe the normal result and the raw SQL.
3. Follow the step-by-step walkthrough in [`ATTACK.md`](ATTACK.md) for the
   tautology, column-count, and UNION-based payloads.
4. Send payloads with the browser form or the convenience wrapper:

   ```bash
   ./exploit.sh "s3cr3t-note"                                                  # normal
   ./exploit.sh "' OR '1'='1"                                                  # 6 rows
   ./exploit.sh "' UNION SELECT ID, user_login, user_pass FROM wp_users -- "   # hashes
   ```

   `exploit.sh` URL-encodes the payload, filters the response to the useful
   lines, and strips HTML tags.

### Phases

- **Phase 1 (current):** find and exploit the injection — see
  [`ATTACK.md`](ATTACK.md).
- **Phase 2 (planned):** replace the string concatenation in
  `vuln-sqli-demo.php` with `$wpdb->prepare( "SELECT ... WHERE post_title = %s", $q )`,
  then re-run the same
  payloads to confirm the flag and password hashes are no longer returned. A
  preview of the fix, and the equivalent Laravel mistake, are at the end of
  [`ATTACK.md`](ATTACK.md).

## Project layout

| Path | Purpose |
|---|---|
| `start.sh` / `stop.sh` | Start/stop the PHP built-in server (PID in `.server.pid`) |
| `exploit.sh` | Send a URL-encoded payload to the local lab |
| `ATTACK.md` | Phase 1 walkthrough and Phase 2 fix preview |
| `runtime/` | Bundled PHP 8.3 (`php-root/`), `wp-cli.phar`, generated `php.ini` |
| `wp-core/` | WordPress 7.1.4 document root |
| `wp-core/wp-content/db.php` | SQLite drop-in; loads the implementation from the plugin below |
| `wp-core/wp-content/plugins/sqlite-database-integration/` | SQLite database driver |
| `wp-core/wp-content/database/.ht.<hash>/.ht.sqlite` | Seeded SQLite database (hidden) |
| `wp-core/wp-content/plugins/vuln-sqli-demo/vuln-sqli-demo.php` | Vulnerable raw SQL (lab target) |
| `docker-compose.yml` | Optional MariaDB + WordPress alternative — **not used** by `start.sh`; every service has `restart: "no"` |

## Troubleshooting

- **`Lab already running (PID ...)`** — a previous server is still alive. Run
  `./stop.sh` first, or kill the PID recorded in `.server.pid`.
- **Port 8080 already in use** — another process owns the port. Stop it (the
  sibling labs bind 8081–8086) and retry.
- **Blank page or database errors** — make sure `runtime/` is intact;
  `start.sh` regenerates `runtime/php.ini` on each launch. Check the `start.sh`
  terminal output for PHP errors.

## Ground rules

- Bound to `127.0.0.1` — do not expose it.
- Do not reuse the credentials or code outside this lab.
- Do not run automated scanners against anything but this local instance.
