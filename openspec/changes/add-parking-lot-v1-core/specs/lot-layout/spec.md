# Spec Delta

## Purpose

Defines the physical structure of a parking lot — lots, floors, spots and gates — and which
vehicle types may park in which spot types, so every other capability shares one layout model.

## ADDED Requirements

### Requirement: REQ-LOT-001 Lot identity
Every lot SHALL have a unique `lotId` (UUID), and every lot-scoped API path and record SHALL carry that `lotId`.

#### Scenario: Resource addressed by lotId
- **WHEN** a client requests availability for `lotId` L1
- **THEN** only floors, spots and tickets belonging to L1 are considered

#### Scenario: Unknown lot
- **WHEN** a client calls any lot-scoped endpoint with a `lotId` that does not exist
- **THEN** the system responds `404` with code `LOT_NOT_FOUND`

### Requirement: REQ-LOT-002 Floors and spots hierarchy
A lot SHALL contain one or more floors, each with a unique `floorNumber` within the lot, and each floor SHALL contain spots with a unique `spotCode` within the floor and a `distanceRank` (integer, lower is nearer the entry).

#### Scenario: Duplicate floor number rejected
- **WHEN** an admin adds floor number 2 to a lot that already has floor 2
- **THEN** the system responds `409` with code `DUPLICATE_FLOOR`

#### Scenario: Duplicate spot code rejected
- **WHEN** an admin adds spot `B-12` to a floor that already has `B-12`
- **THEN** the system responds `409` with code `DUPLICATE_SPOT`

### Requirement: REQ-LOT-003 Spot and vehicle types
The system SHALL support spot types `MOTORCYCLE`, `COMPACT`, `REGULAR`, `LARGE`, `EV`, `HANDICAPPED` and vehicle types `MOTORCYCLE`, `CAR`, `VAN`, `TRUCK`, `EV`; any other value SHALL be rejected.

#### Scenario: Invalid vehicle type
- **WHEN** an entry request has `vehicleType` = `BUS`
- **THEN** the system responds `400` with code `VALIDATION_ERROR`

### Requirement: REQ-LOT-004 Vehicle-to-spot fit rules
Each vehicle type SHALL map to an ordered list of compatible spot types (preferred first): MOTORCYCLE → MOTORCYCLE, COMPACT, REGULAR; CAR → COMPACT, REGULAR, LARGE; EV → EV, COMPACT, REGULAR, LARGE; VAN → LARGE; TRUCK → LARGE. HANDICAPPED spots SHALL only be allocated when the entry request sets `accessibleNeeded=true`.

#### Scenario: Car fits compact first
- **WHEN** a CAR enters and both COMPACT and REGULAR spots are free
- **THEN** the compatible-type order offered to allocation is COMPACT, REGULAR, LARGE

#### Scenario: Truck cannot use regular
- **WHEN** a TRUCK enters and only REGULAR spots are free
- **THEN** no spot is compatible and the entry is rejected as lot full

#### Scenario: Handicapped spot requires flag
- **WHEN** a CAR enters with `accessibleNeeded=true` and a HANDICAPPED spot is free
- **THEN** the HANDICAPPED spot is offered before other compatible types

### Requirement: REQ-LOT-005 Gates
A lot SHALL have one or more gates, each with a `gateId` and a type `ENTRY` or `EXIT`; entry requests SHALL only be accepted on ENTRY gates and exit requests only on EXIT gates.

#### Scenario: Entry at exit gate rejected
- **WHEN** an entry request names a gate of type EXIT
- **THEN** the system responds `422` with code `WRONG_GATE_TYPE`
