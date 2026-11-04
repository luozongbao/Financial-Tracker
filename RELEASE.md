# Release Notes: v1.0

## Summary

First release of the Financial Tracker. It records income and spending, and it searches and sums them over a day, week, month or year. It also reports the average per day, week, month or year.

The release covers the twenty original issues (001–020), the 2026-10-28 security audit (021–025) and the requirement gap found against the brief (026).

## Features

- Income and spending transactions with categories, filters and presets.
- Dashboard with income, spending and net tiles, and recent activity.
- Reports by period (day, week, month, year) and by custom range, with totals, averages and shares.
- Account registration, sign-in, password reset and e-mail verification. Registration can be closed with `ALLOW_REGISTRATION`.
- Profile and password settings, and an installation card that shows the mail queue state.
- Light and dark themes, and a web installer that writes `config.php`.

## Security fixes in this release

| Issue | Finding | Fix |
|---|---|---|
| 021 | The sign-in lockout could be used as a denial of service. | Decaying backoff with per-identifier and per-source counters. The stricter counter wins. The source key uses `PROXY_REMOTE_ADDR`, because LiteSpeed rewrites `REMOTE_ADDR` from `X-Forwarded-For`. |
| 022 | Reset and registration were unthrottled, the reset screen revealed which addresses exist, and the log held personal data. | Byte-identical reset responses. Mail is queued and sent by a worker. The SMTP timeouts are bounded. Reset and registration are throttled. The log carries no submitted values. |
| 023 | The web root served `docker-compose.yml`, which holds the default database credentials. | The deny block covers those files and everything under `docs/` except the mockup. The WebAdmin is bound to 127.0.0.1. Floating image tags are pinned by digest. |
| 024 | No CSP, HSTS or Permissions-Policy, and the installer sent no headers. | One `sendSecurityHeaders()` call used by both entry points. |
| 025 | The installer kept the database and SMTP passwords in a clear-text session file in the shared `/tmp`. | No secret is stored in the session. The session ID is rotated, strict mode is on, and the store is `storage/sessions/` (mode `0700`). |
| 026 | The average per period was not implemented, although the brief requires it. | Averages per day, week, month and year, computed in SQL over the calendar. |

The SQL injection, IDOR, XSS, CSRF, open redirect, session and path traversal checks came back clean in the audit.

## Requirements

- Docker and Docker Compose, for the development stack.
- PHP 8.5.12 or later, if you run outside the container. Earlier 8.5 builds are vulnerable.
- MariaDB 10.11.
- An SMTP server, for registration and password reset.

## Upgrade and install notes

- A fresh install loads `database/schema.sql`, which the installer does automatically.
- A database created before this release must apply the migrations in `database/migrations/` by hand, in date order. See [database/migrations/README.md](database/migrations/README.md).
- Install with the wizard at `/install.php`, then delete `install.php`.
- Schedule `database/outbox_worker.php` from cron. Without it, no mail is sent.
- Run `docker compose restart app` after any change to `.htaccess`.
- Keep `config.php` out of version control. It holds the database and SMTP passwords.

## Known limitations

- Single user per installation. There is no sharing and no multi-tenant separation.
- No import or export of transactions.
- Amounts are stored as `DECIMAL(14,2)` with no currency field.
- The mockup viewer is a separate profile, for design review only. It is not part of the application.
- The development stack exposes phpMyAdmin on port 8090. Do not expose it to the internet.
- The application has no browser-level test suite. The live tier covers the running application over HTTP, and the CSS cascade is checked by a resolver written for this project.
- Behind a reverse proxy, the sign-in source bucket is shared by every visitor that uses the proxy. This is documented behaviour.

## Verification

Results from the validation run on the current tree:

- Source tier: `run-tests.sh` reports 15 PASS, 0 FAIL.
- Live tier: `database/live/run-live-tests.sh` reports 2 PASS, 0 FAIL.

Re-run both before tagging the release.

## Known open items

None of the twenty-six issues is open. See [docs/issues/README.md](docs/issues/README.md).
