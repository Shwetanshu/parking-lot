# Spec Delta

## Purpose

Defines how a parking fee is calculated from a lot's rate card and a ticket's duration, and
issues short-lived quotes so that the amount a client is shown is exactly the amount charged.

## ADDED Requirements

### Requirement: REQ-PRC-001 Rate card per vehicle type
Each lot SHALL have one active rate card defining, per vehicle type, ordered hourly slabs (`upToMinutes`, `ratePerHour`), a `gracePeriodMinutes` and a `dailyCap`, plus a lot-wide `exitWindowMinutes` (default 15); all money is in the lot currency with two decimal places.

#### Scenario: Rate card applied by vehicle type
- **WHEN** a TRUCK and a CAR stay the same duration
- **THEN** each fee is computed from that vehicle type's own slabs

### Requirement: REQ-PRC-002 Duration and rounding
Duration SHALL be measured from `entryTime` to the quote time using server time, and rounded up to the next whole hour after the grace period is exceeded.

#### Scenario: Within grace period is free
- **WHEN** a ticket is quoted 9 minutes after entry and the grace period is 10 minutes
- **THEN** the fee is 0.00

#### Scenario: Partial hour rounds up
- **WHEN** a CAR ticket is quoted 61 minutes after entry, grace 10, flat rate 40.00/hour
- **THEN** the fee is 80.00 (2 hours)

### Requirement: REQ-PRC-003 Slabs and daily cap
Each billed hour SHALL be charged at the rate of the slab it falls into, and the total for each started 24-hour period SHALL not exceed `dailyCap`.

#### Scenario: Slab pricing
- **WHEN** CAR slabs are first 2 hours at 40.00/h then 30.00/h, and the billed duration is 4 hours
- **THEN** the fee is 140.00

#### Scenario: Daily cap applied
- **WHEN** the billed duration is 20 hours and the computed fee exceeds a dailyCap of 300.00
- **THEN** the fee is 300.00

### Requirement: REQ-PRC-004 Fee quote
The system SHALL return a fee quote for an ACTIVE ticket containing `quoteId`, `totalFee`, `alreadyPaid`, `amountDue` (= totalFee − alreadyPaid), `currency`, `billedMinutes`, `quotedAt`, `expiresAt` and a line-item breakdown; all money values SHALL be exact decimals with two places, never floating point.

#### Scenario: Quote returned with breakdown
- **WHEN** a client requests a quote for an ACTIVE ticket with nothing paid yet
- **THEN** the response contains all quote fields, `alreadyPaid` is 0.00, and the line items sum to `totalFee`

#### Scenario: Quote after overstay
- **WHEN** a ticket that already paid 80.00 returns to ACTIVE and its total fee is now 120.00
- **THEN** the quote shows `alreadyPaid` 80.00 and `amountDue` 40.00

#### Scenario: Quote for closed ticket rejected
- **WHEN** a client requests a quote for a CLOSED ticket
- **THEN** the system responds `409` code `ILLEGAL_STATE_TRANSITION`

### Requirement: REQ-PRC-005 Quote validity
A quote SHALL be valid for 5 minutes (`expiresAt` = `quotedAt` + 5 min), and payment against it SHALL charge exactly its `amountDue` even if a new billing hour starts before payment.

#### Scenario: Pay within validity across an hour boundary
- **WHEN** a quote of 80.00 is taken at 59 minutes and paid at 62 minutes
- **THEN** exactly 80.00 is charged

#### Scenario: Expired quote
- **WHEN** a client pays against a quote whose `expiresAt` has passed
- **THEN** the system responds `409` code `QUOTE_EXPIRED` and nothing is charged

### Requirement: REQ-PRC-006 Lot without a rate card
While a lot has no active rate card, entry, fee quote and exit requests for that lot SHALL be rejected with `409` code `RATE_CARD_MISSING`, so no ticket is created that cannot be priced.

#### Scenario: Entry before rate card is set
- **WHEN** a vehicle enters a newly created lot that has no rate card yet
- **THEN** the system responds `409` code `RATE_CARD_MISSING` and no spot is allocated
