# Design

## Context

Greenfield repository: only `AGENTS.md` and the OpenSpec scaffold exist. See `proposal.md` for
motivation and scope, and `specs/` for the normative requirements (REQ-LOT, ALC, TKT, PRC, PAY,
AVL, ADM, API). Fixed constraints from the user: Python and Node.js, PostgreSQL storage,
`lotId` on every lot-scoped API.

## Goals / Non-Goals

**Goals:**
- Two deployable containers with a single, clear responsibility each.
- Correctness under concurrency (no double allocation, no double charge) enforced by the database,
  not by application luck.
- A contract (`contracts/openapi.yaml`) that both services are tested against.
- Every REQ scenario maps to at least one automated test.

**Non-Goals:**
- Caching layer (Redis), message broker (Kafka) or outbox relay — PostgreSQL alone meets the
  v1 latency and 2-second availability targets; they are added with notifications/reports later.
- Horizontal scaling beyond a single PostgreSQL primary.
- Any UI — web or mobile apps, admin portal, kiosk or gate screens, Swagger UI, static assets or
  server-rendered pages. Gates, attendants, drivers and admins are all modelled as API clients;
  manual testing is done with Postman, automated testing with pytest, Vitest and Newman.

## Decisions

### D1. Two containers: Node.js gateway + Python core
```
client ──HTTPS──▶ api-gateway (Node 20, TypeScript, Fastify) ──HTTP──▶ parking-core (Python 3.12, FastAPI) ──▶ PostgreSQL 16
```
- **api-gateway** — JWT verification, role check per route (REQ-API-002/003), `Idempotency-Key`
  presence check, rate limiting (REQ-API-006), request validation against OpenAPI, error-body
  normalisation (REQ-API-005), `traceId` generation. Stateless; no database.
- **parking-core** — all domain logic and the only owner of PostgreSQL.
- Only the gateway publishes a host port; core and database are on an internal network.
- *Alternative considered*: one Python service doing everything — simpler, but the user asked for
  both stacks, and the edge/domain split is a natural seam. *Alternative*: split core per
  capability — premature for v1; the C3 module boundaries below keep that option open.

### D2. Identity is forwarded, not re-verified
The gateway verifies the JWT (HS256, secret from env `JWT_SECRET`) and forwards `X-User-Id`,
`X-User-Roles` and `X-Trace-Id`. Core rejects requests missing these headers and is unreachable
from outside the internal network. *Alternative*: verify JWT in both — duplicated config for no
gain while core is not exposed.
Core uses `X-User-Roles` only for row-level rules the gateway cannot see: a CUSTOMER ticket
lookup or quote must carry a `licensePlate` query parameter matching the ticket, otherwise core
answers `404 TICKET_NOT_FOUND` (REQ-TKT-004), so ticket ids cannot be probed.

### D3. Idempotency stored in core, in the same transaction
Table `idempotency_keys(key, lot_id, request_hash, response_status, response_body, created_at)`
with a unique key. For entry, exit and admin writes, core inserts the key in the same
transaction as the state change, so a crash can never leave a change without a recorded
response. Payments are the exception because the provider call cannot join a database
transaction: the key is stored with the INITIATED payment and the final response is recorded in
the second transaction (see D7, which covers the crash window). Only `2xx` responses and
`402 PAYMENT_DECLINED` are stored; for any other error the key row is deleted in the same
transaction, so a retry is processed afresh (an exit that got `402 PAYMENT_REQUIRED` succeeds
once the ticket is paid). Same key + same hash → replay stored response; different hash →
`409 IDEMPOTENCY_KEY_REUSED`; keys expire after 24 h.
*Alternative*: idempotency store in the gateway (Redis) — cannot be atomic with the domain write.

### D4. Spot allocation via `FOR UPDATE SKIP LOCKED`
```sql
SELECT s.id FROM spots s JOIN floors f ON f.id = s.floor_id
WHERE f.lot_id = :lot AND s.spot_type = :type AND s.status = 'FREE'
ORDER BY s.distance_rank, f.floor_number, s.spot_code
LIMIT 1 FOR UPDATE OF s SKIP LOCKED;
```
Tried for each compatible type in REQ-LOT-004 order, then `UPDATE spots SET status='OCCUPIED'`
and insert the ticket in the same transaction. Two safety nets in the schema:
- partial unique index `tickets(spot_id) WHERE state IN ('ACTIVE','PAID')` (REQ-ALC-002);
- partial unique index `tickets(lot_id, license_plate) WHERE state IN ('ACTIVE','PAID')` (REQ-TKT-002).
*Alternatives*: optimistic `version` column with retries (more retries under load at busy gates);
advisory locks (harder to reason about). `SKIP LOCKED` lets concurrent gates take different spots
without waiting.

### D5. State machines as explicit transition tables
One module per machine (`spot`, `ticket`, `payment`) holding a `{(from, event): to}` map; every
state change in the code goes through `transition()`, which raises `IllegalTransition` →
`409 ILLEGAL_STATE_TRANSITION`. DB `CHECK` constraints restrict the state column values.
Tests: one per arrow plus one illegal transition per machine (as required by AGENTS.md).

### D6. Pricing as a pure function, quotes as rows
`compute_fee(rate_card, vehicle_type, entry_time, now) -> FeeBreakdown` with no I/O, using
Python `Decimal` and `ROUND_HALF_UP`; DB columns `NUMERIC(12,2)`; API money as strings
(REQ-API-007). A `Clock` interface is injected so tests freeze time. Rate cards are stored as
versioned rows (`rate_cards(lot_id, version, body jsonb, active)`); saving a new card deactivates
the old one in one transaction (REQ-ADM-005). The rate card body also holds the lot-wide
`exitWindowMinutes`.
A quote is persisted as `fee_quotes(id, ticket_id, total_fee, already_paid, amount_due,
quoted_at, expires_at, used_at)` where `already_paid` = sum of the ticket's SUCCESS payments. Payment
reads the quote row, never recomputes the fee, so the charged amount always equals the quoted
one (REQ-PRC-005). *Alternative*: recompute at payment and reject on mismatch — rejected because
an hour boundary between quote and payment would make clients fail randomly.

### D7. Payment provider behind an interface
`PaymentProvider.charge(token, amount, currency, idempotency_key)` and `refund(txn_id)`. v1 ships
`MockPaymentProvider`: tokens starting with `tok_decline` are declined, all others approved.
Order (REQ-PAY-006):
1. Transaction 1: lock the ticket (`SELECT … FOR UPDATE`); require ACTIVE (after the exit-window
   rule), a quote of this ticket that is unexpired, unused (`used_at IS NULL`) and whose
   `already_paid` equals the current sum of SUCCESS payments; set `used_at`; insert the payment
   INITIATED; commit. A partial unique index `payments(ticket_id) WHERE status='INITIATED'` is the
   database safety net against two in-flight charges.
2. Call the provider — only after transaction 1 committed, so a rejected request is never charged.
3. Transaction 2: SUCCESS + ticket PAID, or FAILED (the quote stays used; the client takes a new quote).
The provider receives our idempotency key, so a crash between the two transactions is recovered by
retrying with the same key.
Refunds (REQ-PAY-004) require the ticket to be CLOSED, so a refund can never re-open a free exit.

### D8. Exit closes ticket and frees spot atomically
One transaction, after checking the gate is an enabled EXIT gate:
1. Lock the ticket row (`SELECT … FOR UPDATE`).
2. If PAID and `now > last_paid_at + exitWindowMinutes` → transition PAID→ACTIVE (REQ-TKT-005).
3. If ACTIVE: compute amount due (fee entry→now minus SUCCESS payments). If it is zero →
   ACTIVE→PAID with **no payment row**; otherwise commit the PAID→ACTIVE change (if any) and
   return `402 PAYMENT_REQUIRED` with the amount due.
4. PAID → CLOSED, set `exit_time` and `exit_gate_id`, set spot FREE (REQ-TKT-003).
Ticket lookup is read-only: it reports the effective state (ACTIVE once the window has passed)
without writing. The PAID→ACTIVE change is persisted only by quote, payment or exit, each under
the ticket lock, so no background job is needed and GET never changes data.

### D9. Availability computed from PostgreSQL
`GROUP BY floor, spot_type, status` over `spots` with index `(floor_id, spot_type, status)`.
Reads are always current, so REQ-AVL-003's 2-second bound holds trivially. A cache can be added
later behind the same endpoint.

### D10. Module layout (C3)
```
services/parking-core/src/parking_core/
  api/            # FastAPI routers, request/response models (mirror openapi.yaml)
  layout/         # lots, floors, spots, gates, fit rules        (REQ-LOT, REQ-ADM)
  allocation/     # strategy + SKIP LOCKED repository            (REQ-ALC)
  sessions/       # entry, exit, ticket lookup                   (REQ-TKT)
  pricing/        # rate cards, compute_fee                      (REQ-PRC)
  payments/       # provider interface, mock, refunds            (REQ-PAY)
  availability/   # counts, display board                        (REQ-AVL)
  platform/       # db session, idempotency, clock, errors, state machines
services/api-gateway/src/
  auth/ routes/ idempotency/ ratelimit/ errors/ proxy/
```

### D11. Contract-first workflow
`contracts/openapi.yaml` is written before handlers. Core: FastAPI response models are checked
against the contract with `schemathesis` in CI. Gateway: request validation uses the same file.
Paths: `/api/v1/lots/{lotId}/entries`, `/exits`, `/tickets/{ticketId}`, `/tickets/{ticketId}/fee-quote`,
`/tickets/{ticketId}/payments` (body: `quoteId`, `paymentToken`, `method`), `/availability`,
`/floors/{floorId}/spots`, `/display-board`; admin under `/api/v1/admin/...` including
`DELETE /api/v1/admin/lots/{lotId}/spots/{spotId}` and gate disable;
`/api/v1/lots/{lotId}/payments/{paymentId}/refund`. Every lot-scoped path carries `{lotId}` (REQ-LOT-001).
Entry, quote and exit check for an active rate card first and return `409 RATE_CARD_MISSING`
without one (REQ-PRC-006).

### D12. Configuration and local run
All config from env vars (`DATABASE_URL`, `JWT_SECRET`, `CORE_URL`, `RATE_LIMIT_PER_MIN`, …) with
`.env.example` dev defaults. `infra/compose/docker-compose.yml` runs postgres, parking-core
(with Alembic migrations on start) and api-gateway; only the gateway publishes a port (`8080`, the Postman `baseUrl`).

### D13. Testing with Postman instead of a UI
- `postman/parking-lot-v1.postman_collection.json` — one folder per capability (Admin setup,
  Entry, Tickets & pricing, Payment, Exit, Availability), ordered so a top-to-bottom run is the
  happy path; the Admin folder sets the rate card right after creating the lot. Requests save
  `ticketId`, `quoteId`, `paymentId` etc. into collection variables, and each has `pm.test`
  assertions on status and body.
- A collection-level pre-request script sets `Idempotency-Key: {{$guid}}` on every POST/PUT/PATCH.
- `postman/local.postman_environment.json` — `baseUrl=http://localhost:8080/api/v1`, `lotId`,
  `adminToken`, `attendantToken`, `customerToken` (empty; filled by the dev-token script).
- `services/api-gateway/scripts/dev-token.ts` (`npm run dev-token -- --role ADMIN`) signs a
  short-lived JWT with the local `JWT_SECRET`. It is a CLI script, not an HTTP endpoint, so no
  token-minting route exists in the deployed gateway.
- CI runs the collection headless with Newman against the compose stack (REQ-API-009).
- FastAPI's built-in `/docs` and `/redoc` pages are disabled (`docs_url=None, redoc_url=None`);
  the gateway serves only `/api/v1/*` and `/healthz`, everything else returns a JSON 404 (REQ-API-008).
- *Alternative*: Swagger UI for manual testing — rejected because it is a UI and Postman was requested.

## Risks / Trade-offs

- [Hot rows at peak entry] → `SKIP LOCKED` avoids queuing; spots per type are many, so contention is spread.
- [Gateway/core contract drift] → both validated against the same `openapi.yaml` in CI.
- [Crash between payment INITIATED and provider result] → provider idempotency key + retry; an
  INITIATED payment older than 5 minutes is re-queried before a new charge is allowed.
- [Double charge from two quotes or a double click] → ticket lock + single-use quote + one INITIATED payment per ticket (D7).
- [Driver pays, then lingers] → exit window (D8) turns the ticket ACTIVE again and the extra time is due.
- [Clock skew between gate and server] → only server time is used (REQ-PRC-002).
- [Single PostgreSQL is a single point of failure] → accepted for v1; replicas are a later change.
- [Rounding disputes] → pure `compute_fee` with exhaustive table tests and a line-item breakdown.

## Migration Plan

Greenfield — no data migration. Rollout: `docker compose up` applies Alembic migrations, then a
seed script creates a demo lot. Rollback: `docker compose down -v`.

## Open Questions

- Currency and timezone defaults for the seed lot (INR / Asia/Kolkata assumed) — affects only seed data.
