# Tasks

## 1. Repository scaffolding

- [ ] 1.1 Create `services/parking-core/` (Python 3.12, FastAPI, SQLAlchemy 2, Alembic, pytest, ruff) with `pyproject.toml`; verify `pytest` runs with zero tests and `ruff check` passes
- [ ] 1.2 Create `services/api-gateway/` (Node 20, TypeScript, Fastify, Vitest, ESLint); verify `npm test` and `npm run build` succeed on an empty app
- [ ] 1.3 Add `infra/compose/docker-compose.yml` (postgres 16, parking-core, api-gateway; only gateway publishes a port) and `.env.example`; verify `docker compose up` starts all three and `GET /healthz` on the gateway returns 200
- [ ] 1.4 Update `AGENTS.md` layout section to `openspec/specs`, `openspec/changes`, `contracts/`, `services/`; verify the file lists the new paths

## 2. Contract (REQ-API-001, REQ-API-005, REQ-API-007)

- [ ] 2.1 Write `contracts/openapi.yaml` v1.0.0 with every path in design D11, schemas for Ticket, FeeQuote, Payment, Availability, DisplayBoard, admin resources and the `{code,message,traceId}` Error; verify `npx @redocly/cli lint contracts/openapi.yaml` passes
- [ ] 2.2 Add a check that every error `code` used in the specs appears in the contract enum; verify the script passes
- [ ] 2.3 Create `postman/parking-lot-v1.postman_collection.json` (folders per capability, rate card set right after lot creation, Idempotency-Key pre-request script, variables) and `postman/local.postman_environment.json` per design D13; verify a script confirms every contract operation has a request (`REQ_API_009_collection_covers_contract`)

## 3. Platform foundations in parking-core (REQ-API-004, REQ-API-005)

- [ ] 3.1 Add DB session, `Clock` interface (system + frozen), settings from env, FastAPI docs/redoc disabled; verify unit tests for settings and frozen clock pass and `/docs` returns 404
- [ ] 3.2 Add error types and the FastAPI handler producing `{code,message,traceId}`; verify `REQ_API_005_error_body_shape` passes
- [ ] 3.3 Add generic state-machine helper raising `IllegalTransition` → 409; verify its unit tests pass
- [ ] 3.4 Add `idempotency_keys` table + middleware (replay, reuse-conflict, 24 h expiry) per design D3; verify `REQ_API_004_*` tests (missing key, replay, reused key) pass against PostgreSQL using a test-only POST route registered in the pytest app fixture (never in the production app), including `REQ_API_004_errors_are_not_replayed` (a 4xx is not stored and a retry is processed again)

## 4. Lot layout and administration (REQ-LOT-001..005, REQ-ADM-001..006)

- [ ] 4.1 Alembic migration for lots, floors, spots (types, states, `distance_rank`, CHECK constraints, unique codes), gates; verify `alembic upgrade head` and `downgrade base` both succeed
- [ ] 4.2 Implement spot/vehicle type enums and vehicle→spot fit rules incl. HANDICAPPED flag; verify `REQ_LOT_003_invalid_vehicle_type` and `REQ_LOT_004_*` table tests pass
- [ ] 4.3 Implement admin endpoints for lots, floors, bulk spots, `DELETE` spot (409 `SPOT_IN_USE` guard), gates (create/disable); verify `REQ_LOT_001/002/005_*` and `REQ_ADM_001/002/003_*` API tests pass
- [ ] 4.4 Implement spot state machine (FREE/OCCUPIED/MAINTENANCE) and the maintenance endpoint; verify one test per arrow plus `REQ_ALC_004_illegal_transition` and `REQ_ADM_004_maintenance_toggle` pass
- [ ] 4.5 Implement rate-card storage, validation and replacement (versioned rows, incl. `exitWindowMinutes`); verify `REQ_ADM_005_*` tests pass

## 5. Pricing rules (REQ-PRC-001..003)

- [ ] 5.1 Implement pure `compute_fee` with Decimal, grace, hourly round-up, slabs, daily cap; verify table-driven tests for every REQ-PRC scenario pass

## 6. Entry, allocation and fee quotes (REQ-ALC-001..003, REQ-TKT-001, REQ-TKT-002, REQ-PRC-004, REQ-PRC-006)

- [ ] 6.1 Migration for tickets with partial unique indexes on active spot and active plate; verify migration applies and indexes exist
- [ ] 6.2 Implement allocation repository with `FOR UPDATE SKIP LOCKED` in fit-rule order; verify `REQ_ALC_001_*` ordering and fallback tests pass
- [ ] 6.3 Implement entry endpoint (gate type/disabled checks, rate-card check, ticket ACTIVE); verify `REQ_TKT_001_successful_entry`, `REQ_TKT_002_vehicle_already_parked`, `REQ_ALC_003_lot_full`, `REQ_PRC_006_entry_before_rate_card_is_set` pass
- [ ] 6.4 Add concurrency test: N parallel entries for 1 free spot; verify `REQ_ALC_002_concurrent_entries` yields exactly one success and N−1 `LOT_FULL`
- [ ] 6.5 Add `fee_quotes` table (incl. `used_at`) and fee-quote endpoint (`quoteId`, `totalFee`, `alreadyPaid`, `amountDue`, `expiresAt`, line items; `RATE_CARD_MISSING` check); verify `REQ_PRC_004_quote_returned_with_breakdown` and `REQ_PRC_004_quote_for_closed_ticket_rejected` pass

## 7. Payment (REQ-PAY-001..005)

- [ ] 7.1 Migration for payments; implement payment state machine; verify one test per arrow plus `REQ_PAY_005_illegal_transition`
- [ ] 7.2 Implement `PaymentProvider` interface and `MockPaymentProvider` (tok_decline → declined); verify its unit tests pass
- [ ] 7.3 Implement pay-ticket endpoint (body `quoteId`/`paymentToken`/`method`; card-field rejection; transaction 1 per design D7 — ticket lock, ACTIVE check, quote checks `QUOTE_EXPIRED`, `QUOTE_MISMATCH`, `QUOTE_ALREADY_USED`, `NOTHING_DUE`, mark quote used, insert INITIATED; then provider call; then SUCCESS/FAILED and ticket→PAID); verify `REQ_PAY_001/002_*`, `REQ_PRC_005_*` and `REQ_PAY_006_quote_paid_twice` tests pass
- [ ] 7.4 Verify idempotent retry: `REQ_PAY_003_retry_after_timeout` charges the mock provider exactly once
- [ ] 7.6 Add partial unique index on INITIATED payments and a concurrency test firing two payments with different quotes for one ticket; verify `REQ_PAY_006_concurrent_payments_for_one_ticket` and `REQ_PAY_006_stale_quote_after_another_payment` — the mock provider is charged exactly once
- [ ] 7.5 Implement refund endpoint `POST /api/v1/lots/{lotId}/payments/{paymentId}/refund` (ADMIN, ticket must be CLOSED); verify `REQ_PAY_004_refund_success` and `REQ_PAY_004_refund_before_exit_rejected` pass and no payment token appears in stored rows or logs

## 8. Exit, exit window and ticket lookup (REQ-TKT-003..006)

- [ ] 8.1 Implement ticket state machine (ACTIVE→PAID, PAID→CLOSED, PAID→ACTIVE); verify one test per arrow plus `REQ_TKT_006_illegal_transition`
- [ ] 8.2 Implement exit endpoint per design D8 (EXIT `gateId` required, `WRONG_GATE_TYPE`, `RATE_CARD_MISSING`; ticketId or plate; 402 when amount due; zero-fee path with no payment row; CLOSED + `exitGateId` + spot FREE in one transaction); verify `REQ_TKT_003_*` tests pass
- [ ] 8.3 Implement ticket lookup with amount paid and amount due, and the CUSTOMER `licensePlate` match rule; verify `REQ_TKT_004_*` tests pass
- [ ] 8.4 Implement the exit-window rule (PAID→ACTIVE persisted only by quote, payment or exit; lookup read-only; amount due = fee − paid); verify `REQ_TKT_005_*` (overstay, read-only lookup, pay the remainder) and `REQ_PRC_004_quote_after_overstay` pass

## 9. Availability (REQ-AVL-001..005)

- [ ] 9.1 Implement availability summary, vehicle-type filter and floor spot listing; verify `REQ_AVL_001/002/004_*` tests pass
- [ ] 9.2 Implement display-board endpoint with `full` flags; verify `REQ_AVL_005_board_shows_full` passes
- [ ] 9.3 Verify counts change as soon as an entry or exit commits: `REQ_AVL_003_count_drops_after_entry`

## 10. API gateway (REQ-API-002, REQ-API-003, REQ-API-004, REQ-API-006)

- [ ] 10.1 Implement JWT verification and forwarding of `X-User-Id`, `X-User-Roles`, `X-Trace-Id`; verify `REQ_API_002_missing_token` and expired-token tests pass
- [ ] 10.2 Implement the route→role table from REQ-API-003 covering every contract operation; verify `REQ_API_003_*` (customer entry, attendant refund, customer display board) and `REQ_ADM_006_attendant_blocked` tests pass, plus a check that no contract operation is missing from the table
- [ ] 10.3 Implement OpenAPI request validation and `Idempotency-Key` presence check; verify `REQ_API_001_*` and `REQ_API_004_missing_key` pass at the gateway
- [ ] 10.4 Implement rate limiting from `RATE_LIMIT_PER_MIN`; verify `REQ_API_006_too_many_requests` passes
- [ ] 10.5 Implement proxy to core with error-body passthrough and JSON 404 for every non-API path; verify gateway tests with a stubbed core pass and `REQ_API_008_no_html_served` passes
- [ ] 10.6 Add `npm run dev-token -- --role <ROLE>` CLI that signs a local JWT; verify a token it prints is accepted by the gateway for that role

## 11. End-to-end integration

- [ ] 11.1 Add seed script (demo lot, rate card set immediately after the lot, 2 floors, mixed spots, gates); verify it runs against compose
- [ ] 11.2 Add e2e suite through the gateway in compose: entry → quote → pay → exit happy path, lot full, unpaid exit 402, overstay after payment, expired quote; verify all pass
- [ ] 11.3 Run the Postman collection with Newman against the compose stack (fresh DB, tokens from dev-token); verify `REQ_API_009_happy_path_runs` — all requests and `pm.test` assertions pass
- [ ] 11.4 Run contract tests (schemathesis against core via gateway); verify zero contract violations
- [ ] 11.5 Write `README.md` with run/test instructions, including importing the Postman collection and environment and generating tokens; verify the documented steps work from a clean checkout

## Workflow follow-up

- Archive the change with `/opsx:archive` after all tasks are checked and reviewed.
- Confirm `openspec/specs/` holds the eight capability specs after archive.
