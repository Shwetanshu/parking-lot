# AGENTS.md — Parking Lot System

Guidance for AI coding agents (and humans) working in this repository.
Read this file before writing any spec, design, or code.

## 1. Intent

Build a **Parking Lot Management System** that allocates parking spots to vehicles,
issues tickets on entry, computes fees and collects payment on exit, and exposes
real-time availability — specified first, implemented second.

This project is **spec-driven**: the specification is the source of truth. Code is
derived from the spec. If code and spec disagree, the code is wrong — or the spec must
be changed first through an explicit, reviewed update.

## 2. Goals

1. Produce a clear, testable specification of the parking lot domain before any code.
2. Model the domain cleanly (entities, state machines, strategies) so it works both as an
   LLD/OOP interview answer and as a basis for a service implementation.
3. Define a versioned REST API with contracts that the code must match exactly.
4. Make every requirement traceable: requirement → API/entity → task → test.

## 3. Scope

The authoritative v1 scope is the OpenSpec change
`openspec/changes/add-parking-lot-v1-core/` (after archive: `openspec/specs/`). This section
is a summary only.

### In scope (v1)
- **Backend REST APIs only** (`/api/v1`), tested with Postman. No UI of any kind.
- Multi-level lot addressed by `lotId`: floors, spots, entry/exit gates.
- Spot types: Motorcycle, Compact, Regular, Large, EV, Handicapped.
- Vehicle types: Motorcycle, Car, Van, Truck, EV.
- Entry: plate + gate → allocate nearest compatible spot → issue ticket → return `allowed`.
- Fee quote (valid for a few minutes) → token payment against the quote → ticket PAID.
- Exit: plate/ticket + exit gate → allowed if PAID within the exit window (or fee is zero)
  → ticket CLOSED and spot FREE atomically.
- Pricing: hourly/slab rates per vehicle type, grace period, daily cap, exit window.
- Payments: **payment tokens only** (no raw card data), mock provider, admin refunds.
- Real-time availability per lot, floor and spot type; display-board summary as JSON.
- Admin: manage lots, floors, spots, gates; spot maintenance; rate card.
- Auth (JWT) with roles ADMIN, ATTENDANT, CUSTOMER; idempotency keys; rate limiting.

### Later (v2+, not in v1 unless the spec is amended)
- Lost-ticket flow and penalty fee.
- Audit log / audit trail.
- Reservations (hold, cancel, no-show release).
- Monthly passes / subscriptions.
- Notifications (SMS/email receipts, overstay alerts).
- Reports (occupancy, revenue, peak hours).
- ANPR/MQTT hardware integration, offline gate mode.

### Out of scope
- Any UI: web, mobile, admin portal, kiosk or gate screens, Swagger UI, server-rendered pages.
- Real payment provider integration (use the mock provider).
- Physical hardware drivers.

## 4. Non-functional targets

- **No double allocation**: a spot is assigned to at most one active ticket (atomic,
  conditional update or lock).
- **Idempotency**: every state-changing POST requires an `Idempotency-Key`; retries never
  double-allocate or double-charge.
- **Latency**: entry/exit decision p99 < 300 ms.
- **Money**: decimal arithmetic only, never floating point. Server time only.
- **Security**: role-based access (ADMIN, ATTENDANT, CUSTOMER); no secrets in git;
  configuration from environment variables.
- **Extensibility**: allocation, pricing, and payment use the Strategy pattern so new
  types are added without rewriting existing code.
- **Observability**: structured logs with a `traceId` per request, and basic metrics
  (occupancy, gate latency). No audit log in v1.

## 5. Core domain (summary — the spec is authoritative)

Entities: `ParkingLot`, `Floor`, `ParkingSpot`, `Gate`, `Ticket`, `FeeQuote`, `Payment`,
`RateCard` (+ later: `Reservation`, `Subscription`, `AuditEvent`).

State machines (must be implemented exactly as specified, illegal transitions rejected):
- `ParkingSpot`: FREE → OCCUPIED → FREE; FREE ↔ MAINTENANCE.
- `Ticket`: ACTIVE → PAID → CLOSED; PAID → ACTIVE when the exit window expires.
- `Payment`: INITIATED → SUCCESS | FAILED; SUCCESS → REFUNDED.

Strategies: `SpotAllocationStrategy`, `PricingStrategy`, `PaymentProvider`.

## 6. Workflow for agents

1. **Specify before coding.** All spec work goes through OpenSpec changes
   (`/opsx:propose`, `/opsx:apply`, `/opsx:archive`). Each requirement has a stable ID,
   e.g. `REQ-ALC-003`.
2. **Contracts first.** Endpoint and field names must exist in `contracts/openapi.yaml`
   before code uses them. Change the contract, then the code.
3. **Tasks cite requirements.** Every task in `tasks.md` references the REQ IDs it implements.
4. **Tests cite requirements.** Test names start with the REQ ID,
   e.g. `REQ_ALC_003_rejects_entry_when_lot_full`. One test per state transition plus one
   for an illegal transition.
5. **Small steps.** Implement one task group at a time; keep the build and tests green.
6. **Ask, don't guess.** If a requirement is ambiguous, raise it and update the spec
   rather than inventing behaviour in code.

## 7. Repository layout

```
parking_lot/
  AGENTS.md                      # this file
  openspec/
    config.yaml
    specs/                       # current truth, one folder per capability
    changes/                     # proposed changes (proposal, design, specs, tasks)
  contracts/
    openapi.yaml                 # normative API contract (v1)
  postman/                       # Postman collection + local environment
  services/
    parking-core/                # Python 3.12, FastAPI — domain logic, owns PostgreSQL
    api-gateway/                 # Node.js 20, TypeScript — auth, roles, rate limit, routing
  infra/compose/                 # docker-compose for local run
```

## 8. Hard rules

- Never store or log real card numbers — payment tokens only.
- Never allocate a spot without the atomic "only if FREE" guard.
- Spot release and ticket closure happen in the same transaction.
- Never add a UI; every feature is a JSON API testable from Postman.
- Do not add endpoints, entities, or states that are not in the spec.
- No secrets in the repo; configuration comes from environment variables.

## 9. Definition of done (per task)

- Spec and contract updated (if behaviour changed).
- Code implements exactly the specified behaviour.
- Tests named by REQ ID pass, including illegal-transition tests.
- `tasks.md` item checked off with its REQ IDs.
