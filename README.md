# Financial Tracker

A personal finance tracker. Record income and spending, then search and sum (and average) them over a day, week, month or year.

The product brief is in [docs/About.md](docs/About.md). The full issue index is in [docs/issues/README.md](docs/issues/README.md).

## Stack

| Layer | Choice |
|---|---|
| Web server | OpenLiteSpeed with LSPHP 8.5.12 (`litespeedtech/openlitespeed:1.9.4-lsphp85`) |
| Language | PHP 8.5.12 or later (the patched release; see the note below) |
| Database | MariaDB 10.11, `utf8mb4` |
| Mail | PHPMailer over SMTP, through a database queue and a CLI worker |
| Dev environment | Docker Compose (development only) |

There is no framework, no ORM, no bundler and no build step. Third-party PHP libraries are vendored at build time with Composer.

> **PHP 8.5.12 is the floor.** Releases 8.5.0 to 8.5.11 are vulnerable to PHP2Shell (CVE-2026-109618, CVE-2026-109619). Do not lower the floor.

## Quick start (Docker)

```bash
cp .env.example .env                      # set the passwords first
docker compose up -d                      # app, db, phpmyadmin
```

Install the PHP dependencies once, from the project root. Composer is not in the runtime image, so it is mounted from `/tmp`:

```bash
curl -sSL -o /tmp/composer.phar https://getcomposer.org/composer-stable.phar
docker run --rm -v "$PWD":/app -v /tmp:/w -w /app \
  --entrypoint php litespeedtech/openlitespeed:1.9.4-lsphp85 \
  /w/composer.phar install --no-dev --optimize-autoloader --no-interaction
```

Then open the installer at `http://localhost:8088/install.php` and follow the three steps (database, mail, access). The installer writes `config.php`. **Delete `install.php` after a successful install.** Once `config.php` exists, the installer refuses to run.

## Ports

| Service | URL | Notes |
|---|---|---|
| Application | `http://localhost:8088` | `APP_PORT` |
| phpMyAdmin | `http://localhost:8090` | `PHPMYADMIN_PORT`; development only |
| Mockup viewer | `http://localhost:8089` | `MOCKUP_PORT`; start with `docker compose up -d mockup` |
| LiteSpeed WebAdmin | `http://127.0.0.1:7080` | Bound to loopback only; use an SSH tunnel for remote access |

A plain `docker compose up -d` does not start the mockup viewer.

## Configuration

`config.php` is the only configuration source. It is written by the installer and is git-ignored. [config.template.php](config.template.php) is the tracked twin and declares the same constant set.

| Constant | Purpose |
|---|---|
| `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASS` | Database connection |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS` | Outgoing mail |
| `MAIL_FROM`, `MAIL_FROM_NAME` | Sender identity |
| `SESSION_SAVE_PATH` | Private session store (`storage/sessions/`, mode `0700`) |
| `ALLOW_REGISTRATION` | Boolean. `true` by default. `false` closes registration on both the front end and the back end |
| `REQUEST_THROTTLE_ENABLED` | Boolean kill switch for the request limiters |
| `RESET_ADDRESS_LIMIT`, `RESET_SOURCE_LIMIT`, `REGISTER_SOURCE_LIMIT` | Request limits for the reset and registration screens |
| `WEEK_START` | First day of the week for reports (Monday by default) |
| `TIMEZONE` | Application timezone |

Rules: never use `getenv()`, `$_ENV` or `.env` in application code. Booleans are real `true` / `false`, never strings.

## Running the mail worker

The request path only queues mail. The `mail_outbox` table is drained by `database/outbox_worker.php`. **An installation that never runs the worker never sends mail.** Schedule it from cron on the host:

```bash
*/2 * * * * docker compose -f /path/to/financial-tracker/docker-compose.yml exec -T app php /var/www/html/database/outbox_worker.php --quiet
```

A manual run:

```bash
docker compose exec -T app php /var/www/html/database/outbox_worker.php --limit=20
```

The queue depth is shown in the installation card on the settings page.

## Tests

The tests run inside the container. `php` is not on the host PATH.

```bash
# Lint a file
docker compose exec -T app php -l /var/www/html/includes/functions.php

# Source tier: the whole suite (15 test files)
.github/skills/dev-stack-ops/scripts/run-tests.sh

# Live tier: sweeps the running application over HTTP (needs the stack and config.php)
bash database/live/run-live-tests.sh
```

A test passes only on exit code 0 **and** a completion marker. The live tier is not part of `run-tests.sh`, because it writes to the dev database and cleans up after itself.

## Operations

```bash
docker compose ps
docker compose logs -f app
docker compose restart app          # required after editing .htaccess
docker compose down                 # keeps the database volume
```

Never run `docker compose down -v` without a database dump first. It deletes the MariaDB volume.

## Security

The web root is the repository root. The `.htaccess` deny block is a list of exceptions, so a new file in the tree is published unless a rule is added in the same change. See [docs/issues/issue-003.md](docs/issues/issue-003.md) and [docs/issues/issue-023.md](docs/issues/issue-023.md).

Security measures in place: prepared statements, output escaping, CSRF on every write, ownership filtering in SQL, a sign-in throttle per identifier and per source, a mail queue that does not block requests, a sanitised failure log, a strict Content Security Policy, and a session store outside the shared `/tmp`. The details are in [docs/issues/issue-021.md](docs/issues/issue-021.md) to [docs/issues/issue-026.md](docs/issues/issue-026.md).

## Documentation

- [RELEASE.md](RELEASE.md): release notes for v1.0
- [docs/About.md](docs/About.md): the product brief
- [docs/issues/README.md](docs/issues/README.md): the index of all issues
- [docs/mockup/README.md](docs/mockup/README.md): the design mockups
- [AGENTS.md](AGENTS.md): the guide for contributors and agents

