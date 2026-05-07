# Sentry Error Tracking

Sentry integration is **optional**. If `SENTRY_DSN` is unset the app runs exactly as before — Sentry imports are skipped entirely.

## Environment variables

Set these in `/opt/corex-wms/.env` (loaded by systemd via `EnvironmentFile=`).

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `SENTRY_DSN` | Yes, to enable | (unset) | Project DSN from Sentry. When empty, Sentry is a no-op. |
| `APP_ENV` | No | `production` | Tag events with `production`, `staging`, `dev`, etc. |
| `SENTRY_TRACES_SAMPLE_RATE` | No | `0.05` | Fraction of requests to capture as performance traces (0.0–1.0). |
| `GIT_COMMIT` | No | `unknown` | Release tag for event grouping. Set in CI/CD (`git rev-parse HEAD`). |

Profiling is hard-disabled (`profiles_sample_rate=0.0`) and `send_default_pii=False` — see PII note below.

## Getting a DSN

1. Sign in to `sentry.io`.
2. Create a new project. Platform: **FastAPI** (Python).
3. Copy the DSN shown on the install page (looks like `https://<key>@o<org>.ingest.sentry.io/<project>`).
4. Paste into the `SENTRY_DSN` line in `/opt/corex-wms/.env` on the EC2 host.
5. `sudo systemctl restart corex-wms`.

## Testing the integration

After deploying:

1. Log in to the admin UI as a user with the `Admin` role.
2. Copy your JWT from local storage and hit the smoke-test endpoint:
   ```
   curl -H "Authorization: Bearer <JWT>" https://app.corexwms.com/api/health/sentry-test
   ```
   The server will respond with a 500 (that is the point).
3. Open the Sentry UI. Within ~30 seconds you should see an event with message
   `Sentry smoke test -- if this appears in Sentry, integration is working`.
4. If it appears, the pipeline (DSN, network egress, DSN auth, SDK init) is working.
5. If nothing shows up, check `journalctl -u corex-wms | grep -i sentry` for init errors.

## PII note

`send_default_pii=False` is set in `app/main.py`. Sentry will **not** automatically
attach request bodies, cookies, headers containing auth, or usernames. Stack
traces, file names, and the exception message are still sent. If anything in
your exception messages might leak customer data, scrub it at the `raise` site
before re-raising.

## What is captured

- Any unhandled exception that hits the FastAPI global exception handler in
  `app/main.py` (we call `sentry_sdk.capture_exception()` explicitly there
  because the handler swallows exceptions to return a friendly 500).
- A sampled fraction of normal request transactions (performance data), per
  `SENTRY_TRACES_SAMPLE_RATE`.
- SQLAlchemy queries on captured transactions (via `SqlalchemyIntegration`).

## Disabling

Comment out or remove `SENTRY_DSN` in `.env`, restart the service. No code
changes required.
