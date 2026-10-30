# FilsPay

Card payments for Bahraini merchants, in **Bahraini Dinar, counted in fils**
(1 BD = 1000 fils). Merchants take payments through the API or a shareable
payment link, refund them, and are paid out every night. Finance and support
staff run the platform from an admin console.

This fork is team **Robins-AUBH**'s entry to *The Handover* hackathon. We were
handed the codebase as it was and asked to audit it, fix what was wrong with
the smallest change at the cause, finish refunds, host it, and show it holds up
under load.

| | |
|---|---|
| Merchant dashboard | https://168.144.75.125/dashboard |
| Checkout | `https://168.144.75.125/pay/<token>` (one per payment link) |
| Admin console | https://168.144.75.125:8443 |
| Public API | https://168.144.75.125/v1 · health: https://168.144.75.125/healthz |
| Team | Sama Elsawaf, Amna (`amna-T` on GitHub) |

The server uses a self-signed certificate (there is no domain name), so the
browser will ask you to accept it once.

---

## What we did

- **Audited the whole codebase against the ADRs.** 104 graded findings
  (19 critical, 47 high, 38 medium), each with where, why, what it breaks and
  the fix: [`FINDINGS.md`](FINDINGS.md).
- **Fixed them with a failing test first.** Every fix comes with a new test file
  (`*.new.test.ts`, `*_test.go`) that fails on the old code. Existing tests were
  changed only with approval, and each change is recorded in FINDINGS.md.
- **Highlights:**
  - login worked without the OTP code;
  - OTPs and payment-link tokens came from `Math.random()`;
  - CSV imports were 10× too small (cents instead of fils);
  - a payment link could be charged twice;
  - test merchants were settled for real;
  - the daily close ran at 03:00 instead of Bahrain midnight;
  - one request could freeze any merchant's checkout;
  - fuzzing all 79 API routes found 44 inputs that returned 500, now 0.
- **Finished refunds** (FP-212 / FP-300): full and partial, with no refunding
  more than was captured and no double refund under concurrency. Refunds go to
  the merchant's webhooks and reports, and are deducted from payouts.
- **Hosted it** with Docker Compose behind Caddy (HTTPS, security headers), with
  resource limits, health checks, restart policies and a one-command redeploy.
- **Load-tested it** on the hosted server; results are under Capacity below.
- **Redesigned the UI** ("Pearl"): merchant dashboard, checkout and admin
  console. See [`CHANGELOG.md`](CHANGELOG.md).
- **Rewrote [`CLAUDE.md`](CLAUDE.md)**, the AI assistant's instructions. The
  old version told it money was in cents and that `Math.random` was fine for
  tokens, which would have led it into the bugs above.

## Architecture

```
                         ┌──────────────────────── Docker Compose ────────────────────────┐
 browser ──HTTPS:443───▶ │ proxy (Caddy)  ── /v1, /healthz ──▶ api (Fastify, :3000)       │
 browser ──HTTPS:8443──▶ │  serves web + admin builds          │   0.5 vCPU / 512 MB       │
                         │                                      ▼                          │
                         │ settlement (Go daemon) ──────▶ postgres 16 ◀── reporting        │
                         │  nightly close at 00:00 Bahrain       ▲      (outbox consumer)  │
                         └───────────────────────────────────────┴─────────────────────────┘
```

| Path | What it is |
|---|---|
| `apps/api` | TypeScript / Fastify API: auth, payments, payment links, refunds, webhooks, admin |
| `apps/settlement` | Go daemon: nightly batches, fees, payouts, splits, ledger rebuilds |
| `apps/reporting` | TypeScript outbox consumer that builds the reporting tables |
| `apps/web` | React app: merchant dashboard (`/dashboard`) and public checkout (`/pay/:token`) |
| `apps/admin` | React app: operations console |
| `packages/money` | Integer-fils money helpers (frozen: audited Q2 2026) |
| `packages/events` | Shared event types |
| `db/migrations` | SQL migrations, applied in order by `npm run migrate` |
| `scripts` | Operational scripts (`expire_holds`, `set_passwords`, `reconcile_march`) |
| `ops` | Compose file, Dockerfiles, Caddyfile, `deploy.sh`, load tests |
| `docs` | Architecture, ADRs, runbooks, admin guide, AI setup |

**Rules the code follows** (from [`docs/adr/`](docs/adr), which win over
everything else):

- Money is integer fils; text amounts go through `parseBhdToFils`; rates are
  basis points; no floats on any money path (ADR-0001).
- Scheme exports are truncated to 2 decimals and the residue is journaled
  (ADR-0002).
- A business day is midnight to midnight Asia/Bahrain, UTC+3 with no DST
  (ADR-0003).

## Running it

### With Docker (the whole stack)

```bash
cp ops/.env.docker.example ops/.env.docker      # then set ACQUIRER_CALLBACK_SIGNING_SECRET
export POSTGRES_PASSWORD=$(openssl rand -hex 24)
export SETTLEMENT_INTERNAL_TOKEN=$(openssl rand -hex 24)
export WEBHOOK_SECRET_ENCRYPTION_KEY=$(openssl rand -base64 32)
export SITE_ADDRESS=localhost

docker compose -f ops/docker-compose.yml up -d --build postgres
docker compose -f ops/docker-compose.yml run --rm migrate
docker compose -f ops/docker-compose.yml run --rm seed   # local only: wipes data
docker compose -f ops/docker-compose.yml up -d --build proxy api settlement reporting
curl -k https://localhost/healthz
```

Then open https://localhost/dashboard and https://localhost:8443. The full
guide, including demo logins and replacing the seeded passwords, is in
[`ops/README.md`](ops/README.md).

### Redeploying the hosted server

```bash
ops/deploy.sh
```

This pulls `main`, builds, migrates, restarts the services and checks
`/healthz` through the proxy. It refuses to run between 00:00 and 01:00
Bahrain time, while the nightly settlement runs, unless `FORCE=1`. Settlement
also gets 120 s to finish a batch before it is stopped.

### Tests

```bash
export DATABASE_URL=postgres://...               # a Postgres 16 database
npm install
npm run migrate                                  # every migration applies on an empty database
npm run typecheck
npm test                                         # TypeScript: all packages and apps
cd apps/settlement && go test -p 1 ./...         # Go
```

Run the TypeScript and Go suites one after the other, and Go with `-p 1`: they
share one database.

## Capacity

Measured with k6 against the API limited to 0.5 vCPU / 512 MB
([`ops/loadtest/README.md`](ops/loadtest/README.md)):

| Test | Result |
|---|---|
| Fixed scenario (`scenario.js`): 9 in 10 users are shoppers paying every 30 s, 1 in 10 are merchants refreshing their transactions every 10 s | **300 simulated users** pass (checkout p95 872 ms, 0% turned away). 325 users just miss the 1 s p95 |
| Throughput (`checkout.js`) on the hosted box, through HTTPS | **14 checkouts/s** sustained, p95 under 1 s; ceiling about 15/s |

- **When overloaded,** the API answers at once with `503 overloaded` and
  `Retry-After: 1` instead of queueing. Latency stays bounded and nothing
  crashes or restarts.
- **Recovery:** it recovers as soon as load drops.
- **Retries are safe:** every payment request takes an `Idempotency-Key`, so a
  retry can't charge twice.

## Security notes

- Secrets come from `ops/.env` and `ops/.env.docker`, which are gitignored;
  only `.env.docker.example` is committed.
- The acquirer callback signing secret was committed in the original history.
  It is untracked now, but **must be rotated** with the acquirer.
- Acquirer callbacks must be signed (over the raw body), fresh, and unique by
  `callback_id`.
- Webhook URLs can't point at internal hosts.
- Handlers re-check authorisation and merchant scoping, not just the middleware.
- `npm run seed` wipes data and refuses `NODE_ENV=production`.

## Using an AI assistant on this repo

We worked with Claude Code. Its setup is committed so it can be reused:

| Piece | File |
|---|---|
| Agent instructions | [`CLAUDE.md`](CLAUDE.md), [`AGENTS.md`](AGENTS.md) |
| Permission rules, custom commands, MCP config | [`docs/ai-setup/`](docs/ai-setup) (install steps in its README) |
| Helper scripts | `ops/deploy.sh`, `scripts/set_passwords.ts`, `ops/loadtest/` |

## Open items

| Item | Status |
|---|---|
| Rotate the acquirer callback secret from git history | Needs the acquirer |
| Manual credits and journaled adjustments never reach a payout (R4-4) | Waiting on a finance decision |
| Acquirer retry idempotency key (R4-6) | Needs the real acquirer's retry contract |
| Checkout ceiling: a database connection is held during the acquirer call | Release it during the call to go past ~15 checkouts/s |

## Further reading

| Document | What's in it |
|---|---|
| [`FINDINGS.md`](FINDINGS.md) | Every finding, its fix and its test |
| [`CHANGELOG.md`](CHANGELOG.md) | What changed, by area |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | How the system fits together |
| [`docs/adr/`](docs/adr) | The decisions the code must follow |
| [`docs/RUNBOOK.md`](docs/RUNBOOK.md), [`docs/ADMIN.md`](docs/ADMIN.md) | Operating it |
| [`docs/ONBOARDING.md`](docs/ONBOARDING.md) | Getting started as a developer |
| [`ops/README.md`](ops/README.md) | Hosting, secrets, redeploy |
