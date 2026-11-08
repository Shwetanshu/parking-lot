# Spec Delta

## Purpose

Lets administrators set up and maintain a lot — its floors, spots, gates, maintenance state and
rate card — without code changes or downtime.

## ADDED Requirements

### Requirement: REQ-ADM-001 Manage lots
An ADMIN SHALL be able to create a lot (name, address, currency, timezone) and update its name and address.

#### Scenario: Create lot
- **WHEN** an ADMIN creates a lot named "City Centre"
- **THEN** the system responds `201` with a new `lotId`

### Requirement: REQ-ADM-002 Manage floors and spots
An ADMIN SHALL be able to add floors to a lot and add spots to a floor in bulk; a spot with an ACTIVE or PAID ticket SHALL not be removable.

#### Scenario: Bulk add spots
- **WHEN** an ADMIN adds 20 COMPACT spots to floor F1
- **THEN** 20 FREE spots exist on F1 and availability reflects them

#### Scenario: Remove occupied spot rejected
- **WHEN** an ADMIN removes a spot that is OCCUPIED
- **THEN** the system responds `409` code `SPOT_IN_USE`

### Requirement: REQ-ADM-003 Manage gates
An ADMIN SHALL be able to add ENTRY and EXIT gates to a lot and disable a gate; a disabled gate SHALL reject entry and exit requests with `422` code `GATE_DISABLED`.

#### Scenario: Disabled gate
- **WHEN** an entry request names a disabled gate
- **THEN** the system responds `422` code `GATE_DISABLED`

### Requirement: REQ-ADM-004 Spot maintenance
An ADMIN SHALL be able to move a spot between FREE and MAINTENANCE following the spot state machine (REQ-ALC-004).

#### Scenario: Maintenance toggle
- **WHEN** an ADMIN sets a FREE spot to MAINTENANCE
- **THEN** the spot is excluded from allocation and from FREE counts

### Requirement: REQ-ADM-005 Configure rate card
An ADMIN SHALL be able to replace a lot's rate card; the new card SHALL apply to quotes made after it is saved, and invalid cards (negative amounts, unordered slabs, missing vehicle type) SHALL be rejected with `400` code `VALIDATION_ERROR`.

#### Scenario: New rate card applies to later quotes
- **WHEN** an ADMIN saves a rate card raising the CAR rate
- **THEN** quotes made afterwards use the new rate

#### Scenario: Invalid rate card
- **WHEN** an ADMIN submits a rate card with a negative dailyCap
- **THEN** the system responds `400` code `VALIDATION_ERROR` and the old card stays active

### Requirement: REQ-ADM-006 Admin only
Every administration endpoint SHALL require the ADMIN role.

#### Scenario: Attendant blocked
- **WHEN** an ATTENDANT calls an administration endpoint
- **THEN** the system responds `403` code `FORBIDDEN`
