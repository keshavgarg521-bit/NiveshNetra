# NiveshNetra
AI-powered FinTech risk analyzer for transaction monitoring, anomaly detection, fraud risk scoring, and explainable financial insights.
# PayLedger: Auditable Payment Infrastructure with an Explainable Risk Layer

---

## Why this project exists

Most student fintech projects are CRUD apps with a `transactions` table. Real payment systems are built differently: money is recorded as balanced ledger entries, retries must never move money twice, and every number must be reconcilable against an outside source.

This project is a **reference implementation** showing how to build a payment backend where:

- **Correctness is provable**, not just claimed (tests and database constraints enforce it)
- **AI is safely contained**: a risk model can flag or hold a transfer, but it can never write to the ledger

This is **not** a production payment system and does not handle real money or card data. It uses a simulated payment provider.

---

## Features

### Phase 1: Payment core (baseline)

- [ ] Double-entry ledger (append-only, integer minor units, debits = credits)
- [ ] Wallet accounts and system accounts (platform cash, fee revenue, suspense)
- [ ] Transfers with row-level locking (no overspending under concurrency)
- [ ] Idempotent APIs via the `Idempotency-Key` header
- [ ] Transfer state machine with an allowed-transitions table
- [ ] Mock payment provider (deposits, withdrawals, delayed and failed responses)
- [ ] Inbound webhooks (signature check, dedupe, background processing)
- [ ] Refunds (partial refunds, capped at the original amount)
- [ ] Daily reconciliation against provider settlement files
- [ ] Basic audit log
- [ ] Simple dashboard (wallet, transfer history, ledger view, recon report)

### Phase 2: Differentiators

- [ ] Risk service (velocity, behavior, and device features)
- [ ] Anomaly detection model plus deterministic rules
- [ ] Explainable reason codes and an LLM narration layer (structured facts only)
- [ ] Manual-review queue with analyst actions
- [ ] Hash-chained, tamper-evident audit trail
- [ ] Transactional outbox and outbound merchant webhooks
- [ ] Adversary simulator (low-and-slow, mule chains, threshold probing)

---

## Architecture

```
React UI
   │
   ▼
API server (Node + TypeScript, Fastify)
  ├─ Auth
  ├─ Accounts / Wallets
  ├─ Transfers (idempotent)
  ├─ Ledger service
  └─ Webhook receiver
   │
   ▼
PostgreSQL  ◄──── Worker (background jobs)
                     ├─ webhook processing
                     └─ daily reconciliation
   ▲
   │
Mock Payment Provider (simulated PSP with fault injection)
```

**Phase 2 adds:** a Python risk service, a review queue, Redis, and an outbox for events.

### Design principles

1. **Money is an integer.** All amounts are stored in minor units (paise or cents). Never floats.
2. **Ledger entries are append-only.** Mistakes are fixed with reversing entries, never `UPDATE` or `DELETE`.
3. **Every money-moving endpoint is idempotent.** Same key, same request: same stored response.
4. **Postgres is the source of truth.** Not Redis, not the application's memory.
5. **The risk layer is advisory.** It has no write access to ledger tables.

---

## Tech stack

| Layer | Choice |
|---|---|
| API | Node.js, TypeScript, Fastify, Zod |
| Database | PostgreSQL, plain SQL migrations (`node-pg-migrate`), `pg` |
| Tests | Vitest |
| Frontend | React |
| Local infra | Docker Compose |
| Phase 2 | Python (FastAPI, scikit-learn), Redis, BullMQ |

---

## Repository structure

```
/apps
  /api          Fastify API, ledger service, transfers, idempotency
  /mock-psp     Simulated payment provider with fault injection
  /web          React dashboard
  /worker       Background jobs (webhooks, reconciliation)
/infra
  docker-compose.yml
  /migrations   SQL migrations
/docs
  decisions.md  Short notes on key design decisions
  api.md        Endpoint contracts
```

---

## Getting started

### Prerequisites

- Node.js 20+
- Docker and Docker Compose
- npm or pnpm

### Setup

```bash
# 1. Clone
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>

# 2. Environment
cp .env.example .env

# 3. Start Postgres
docker compose -f infra/docker-compose.yml up -d

# 4. Install dependencies
npm install

# 5. Run migrations
npm run migrate:up

# 6. Start the API
npm run dev:api

# 7. Run tests
npm test
```

---

## Core data model

```
users
accounts             (id, owner_id, type, currency)
ledger_transactions  (id, transfer_id, kind, reversal_of, created_at)
ledger_entries       (id, txn_id, account_id, direction, amount_minor)
transfers            (id, from_acct, to_acct, amount_minor, status, created_at)
transfer_events      (id, transfer_id, from_status, to_status, actor, reason, ts)
idempotency_keys     (key, scope, request_hash, status, response_body)
webhook_events       (id, provider_event_id, payload, processed)
audit_log            (id, actor, action, entity, details, ts)
recon_runs / recon_breaks
```

---

## How a transfer works

1. Client calls `POST /transfers` with an `Idempotency-Key` header.
2. The API checks the key. If it was seen before with the same payload, it returns the stored response and stops.
3. It opens a DB transaction and locks both accounts in a consistent order.
4. It checks that the sender has enough balance.
5. It writes the transfer row and the ledger entries (debit sender, credit receiver).
6. It marks the transfer `succeeded`, stores the response against the key, and commits.
7. It writes an audit row.

If anything fails, the whole transaction rolls back and no money moves.

### Idempotency behavior

| Situation | Result |
|---|---|
| New key | Process and store the response |
| Same key, same payload, completed | Return the stored response |
| Same key, still in progress | `409 Conflict` |
| Same key, different payload | `422 Unprocessable Entity` |

---

## Example API usage

```bash
curl -X POST http://localhost:3000/transfers \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: abc-123" \
  -d '{
        "from_account": "acc_alice",
        "to_account": "acc_bob",
        "amount_minor": 50000,
        "currency": "INR"
      }'
```

Run the same command again with the same key and the response is identical, and the money still moves only once.

---

## Testing and correctness

These are the guarantees the test suite is meant to prove:

| Guarantee | How it is tested |
|---|---|
| Debits always equal credits | Unit tests, a DB constraint, and property-based tests with random transfer sequences |
| No duplicate transfers under retry | Concurrent requests with the same key, plus a "lose the response after commit" test; assert exactly one ledger transaction |
| No overspending | Parallel transfers against the same account |
| Duplicate webhooks are processed once | Replay the same provider event id repeatedly |
| Reconciliation catches errors | Settlement file with deliberately injected faults |

```bash
npm test
```

> Only report numbers (latency, detection rate) that you have actually measured. Add them here once you have them.

---

## Demo

1. Create a transfer with key `abc-123`. The response is dropped after the commit.
2. Retry with `abc-123`. The original response comes back.
3. Open the ledger view: **one** balanced journal transaction, and balances moved once.
4. *(Phase 2)* Trigger a suspicious pattern and watch it land in the review queue with a score and reasons.

A demo video or GIF will go here.

---

## Known limitations

Being upfront about these is deliberate.

- **Simulated provider.** No real money, no real card data, no PCI scope.
- **Synthetic data only (Phase 2).** Any risk-model accuracy will be measured on data we generated, so it is optimistic and not comparable to production fraud systems.
- **No network effect.** Commercial fraud tools see signals across thousands of merchants. This project cannot.
- **No real identity verification (KYC).** Mule and synthetic accounts are easy to create here.
- **Social-engineering fraud is out of scope.** A transaction-only model cannot reliably catch victims who send money willingly.
- **Not security-audited.** Do not use this for real financial activity.

---

## Existing projects worth studying

- [Formance](https://formance.com): open-source programmable ledger and payments platform
- [Blnk](https://blnkfinance.com): open-source double-entry ledger
- [TigerBeetle](https://tigerbeetle.com): financial transactions database
- [Stripe docs](https://docs.stripe.com): reference for idempotency and webhooks

This project is a learning and portfolio implementation, not a competitor to these.

---

## Roadmap

- [x] Project setup and architecture
- [ ] Ledger core and balance invariant tests
- [ ] Transfer API, locking, idempotency
- [ ] Mock provider, deposits, withdrawals, webhooks
- [ ] Refunds and audit log
- [ ] Reconciliation
- [ ] Auth and dashboard
- [ ] Phase 2: risk engine, review queue, explanations

---

## Contributing

This is a two-person project.

- No direct pushes to `main`. Every change goes through a pull request reviewed by the other person.
- Keep PRs small (1 to 3 days of work).
- A task is done only when its tests pass.
- Never commit secrets. `.env` is git-ignored.

---

## Authors

- **Keshav Garg**: ledger, transfers, idempotency
- **Anupam Jain**: mock provider, webhooks, reconciliation

---

## License

MIT. See [LICENSE](LICENSE).
