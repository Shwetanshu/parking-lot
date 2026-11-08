# Spec Delta

## Purpose

Sets the rules every public API call follows — versioning, authentication, roles, idempotency,
rate limiting and error format — so clients can integrate once and retry safely.

## ADDED Requirements

### Requirement: REQ-API-001 Versioned contract
All public endpoints SHALL be served under `/api/v1` and SHALL match `contracts/openapi.yaml` exactly (paths, fields, status codes); requests that do not match the schema SHALL be rejected with `400` code `VALIDATION_ERROR`.

#### Scenario: Unknown field type
- **WHEN** a client sends `licensePlate` as a number
- **THEN** the system responds `400` code `VALIDATION_ERROR` naming the field

### Requirement: REQ-API-002 Authentication
Every endpoint except health checks SHALL require a valid bearer JWT; a missing, expired or invalid token SHALL be rejected with `401` code `UNAUTHENTICATED`.

#### Scenario: Missing token
- **WHEN** a request has no Authorization header
- **THEN** the system responds `401` code `UNAUTHENTICATED`

### Requirement: REQ-API-003 Roles
The system SHALL enforce roles per endpoint. ADMIN: every endpoint. ATTENDANT: entry, exit, ticket lookup, fee quote, pay, availability, spot listing, display board. CUSTOMER: ticket lookup and fee quote (with matching `licensePlate`, REQ-TKT-004), pay, availability, display board. Admin endpoints and refunds are ADMIN only. Any other call SHALL be rejected with `403` code `FORBIDDEN`.

#### Scenario: Customer cannot run gate entry
- **WHEN** a CUSTOMER calls the entry endpoint
- **THEN** the system responds `403` code `FORBIDDEN`

#### Scenario: Attendant cannot refund
- **WHEN** an ATTENDANT calls the refund endpoint
- **THEN** the system responds `403` code `FORBIDDEN`

#### Scenario: Customer can read the display board
- **WHEN** a CUSTOMER calls the display-board endpoint
- **THEN** the system responds `200` with the board

### Requirement: REQ-API-004 Idempotency keys
Every state-changing POST SHALL require an `Idempotency-Key` header. Only `2xx` responses and `402 PAYMENT_DECLINED` SHALL be stored; the same key with the same body within 24 hours SHALL return a stored response, or be processed again if none was stored. The same key with a different body SHALL be rejected with `409` code `IDEMPOTENCY_KEY_REUSED`.

#### Scenario: Missing key
- **WHEN** an entry POST has no Idempotency-Key
- **THEN** the system responds `400` code `IDEMPOTENCY_KEY_REQUIRED`

#### Scenario: Replay returns original
- **WHEN** an entry POST is repeated with the same key and body
- **THEN** the same ticket is returned and no second spot is allocated

#### Scenario: Errors are not replayed
- **WHEN** an exit POST got `402 PAYMENT_REQUIRED`, the ticket is then paid, and the gate retries the exit with the same key and body
- **THEN** the exit is processed again and the response is `200` with `allowed=true`

#### Scenario: Key reused with different body
- **WHEN** a key is reused with a different license plate
- **THEN** the system responds `409` code `IDEMPOTENCY_KEY_REUSED`

### Requirement: REQ-API-005 Error model
Every error response SHALL have the body `{code, message, traceId}` with a stable machine-readable `code`.

#### Scenario: Error body shape
- **WHEN** any request fails with a 4xx or 5xx status
- **THEN** the body contains `code`, `message` and `traceId`

### Requirement: REQ-API-006 Rate limiting
The gateway SHALL limit each client to a configurable request rate and reject excess requests with `429` code `RATE_LIMITED`.

#### Scenario: Too many requests
- **WHEN** a client exceeds its configured rate
- **THEN** the system responds `429` code `RATE_LIMITED`

### Requirement: REQ-API-007 Money and time formats
Money SHALL be sent as decimal strings with two places (e.g. `"80.00"`) plus a currency code, and timestamps as ISO-8601 UTC.

#### Scenario: Money as string
- **WHEN** a fee quote is returned
- **THEN** `amount` is a string such as `"80.00"` and `currency` is an ISO-4217 code

### Requirement: REQ-API-008 Backend API only, no UI
Every v1 operation SHALL be available as a JSON-over-HTTP endpoint on the gateway, and the system SHALL serve no user interface: no HTML pages, static web assets, or server-rendered views.

#### Scenario: Every operation reachable over HTTP
- **WHEN** a tester uses only an HTTP client such as Postman
- **THEN** they can perform every v1 operation (admin setup, entry, quote, payment, exit, availability) without any other tool or screen

#### Scenario: No HTML served
- **WHEN** a client requests `/` or any non-API path on the gateway
- **THEN** the system responds `404` with the JSON error body and never returns an HTML page

### Requirement: REQ-API-009 Postman collection
The system SHALL ship a Postman collection covering every endpoint in `contracts/openapi.yaml`, plus a local environment file with `baseUrl`, a `lotId` variable and one bearer token per role, where every state-changing request sends a fresh `Idempotency-Key`.

#### Scenario: Collection covers the contract
- **WHEN** the collection's requests are compared with the paths and methods in `contracts/openapi.yaml`
- **THEN** every contract operation has at least one request in the collection

#### Scenario: Happy path runs end to end
- **WHEN** the collection is run in order against a freshly started local stack
- **THEN** the requests create a lot, park a car, quote, pay and exit it, and all their tests pass
