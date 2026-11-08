# Proposal

## Why

There is no system yet to run a multi-level parking lot: allocate spots, issue tickets, charge
fees and report live availability. This first change establishes the core domain, the versioned
API contract and the REQ-ID traceability that every later change (reservations, passes,
notifications, reports) builds on.

## What Changes

- Add a multi-level lot model, addressed by `lotId` on every API: lot → floors → spots, plus
  entry/exit gates.
- Add vehicle entry: scan plate → allocate nearest compatible spot → issue ticket → return the barrier decision
  (`allowed`) to the caller.
- Add vehicle exit at an EXIT gate: look up ticket → quote fee → pay against the quote → close
  ticket and free spot atomically → return `allowed=true`. A paid ticket must exit within a
  configurable exit window, otherwise the extra time is due.
- Add pricing: per-vehicle-type hourly/slab rates, grace period, daily cap, exit window, and
  fee quotes valid for 5 minutes.
- Add payment through a mock provider using **payment tokens only** (never card data), with
  idempotent charge and refund.
- Add real-time availability per lot, floor and spot type, and a display-board summary as a JSON endpoint.
- Add lot administration: create lots, floors, spots, gates; remove unused spots; disable gates;
  toggle spot maintenance; set rate card.
- Add the normative contract `contracts/openapi.yaml` (`/api/v1`).
- **Backend APIs only, no UI**: every actor (gate, attendant, driver, admin) is an HTTP API
  client. The system is exercised with Postman; a Postman collection and environment are
  delivered alongside the contract.
- Tech stack: Python (FastAPI) core parking service owning PostgreSQL and the idempotency-key
  store; Node.js (TypeScript) API gateway for auth, roles, rate limiting, request validation,
  `Idempotency-Key` presence check and routing.

## Capabilities

### New Capabilities

- `lot-layout`: Lot, floor, spot and gate structure; spot types, vehicle types and the vehicle→spot fit rules (REQ-LOT).
- `spot-allocation`: Choosing and atomically claiming a spot, lot-full rejection, and the spot state machine (REQ-ALC).
- `parking-session`: Entry, ticket issue, exit, exit window, ticket lookup, and the ticket state machine (REQ-TKT).
- `pricing`: Rate cards, fee calculation rules and time-limited fee quotes (REQ-PRC).
- `payment`: Token-only payment against a quote, refund, and the payment state machine (REQ-PAY).
- `availability`: Free-spot counts per lot/floor/type and the display-board summary endpoint (REQ-AVL).
- `lot-administration`: Admin management of lots, floors, spots, gates, maintenance and rate cards (REQ-ADM).
- `api-access`: Authentication, roles, idempotency keys, versioning, the error model, and API-only access testable via Postman (REQ-API).

### Modified Capabilities

- None (first change; no existing specs).

## Impact

- **New specs**: the eight capabilities above under `openspec/specs/` after archive.
- **New contract**: `contracts/openapi.yaml` (API v1) — endpoint and field names are fixed by it.
- **New code**:
  - `services/parking-core/` — Python 3.12, FastAPI, SQLAlchemy, Alembic, pytest.
  - `services/api-gateway/` — Node.js 20, TypeScript, Fastify, Vitest.
  - `infra/compose/docker-compose.yml` — PostgreSQL 16, parking-core, api-gateway.
- **Postman**: `postman/parking-lot-v1.postman_collection.json` and `postman/local.postman_environment.json`.
- **Dependencies**: PostgreSQL 16; a mock payment provider inside parking-core; Newman (Postman CLI) for running the collection in CI.
- **Docs**: `AGENTS.md` layout section updated to `openspec/specs` + `openspec/changes`.
- **Not in this change**: any UI — web, mobile, admin portal, kiosk or gate screens, server-rendered pages; reservations, monthly passes, notifications, reports, ANPR/MQTT
  hardware, offline gate mode, real payment provider.
