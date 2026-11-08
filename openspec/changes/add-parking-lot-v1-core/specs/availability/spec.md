# Spec Delta

## Purpose

Reports through the API how many spots are free right now, per lot, floor and spot type, so
API clients can avoid sending cars into a full lot or floor.

## ADDED Requirements

### Requirement: REQ-AVL-001 Lot availability summary
The system SHALL return, for a lot, the total and FREE spot counts per spot type and per floor; MAINTENANCE spots SHALL count toward total but not FREE.

#### Scenario: Availability summary
- **WHEN** a lot has 10 COMPACT spots of which 3 are OCCUPIED and 1 is MAINTENANCE
- **THEN** the summary reports COMPACT total 10 and free 6

### Requirement: REQ-AVL-002 Filter by vehicle type
When `vehicleType` is given, the summary SHALL report the FREE count of spots compatible with that vehicle type (per REQ-LOT-004).

#### Scenario: Free count for a van
- **WHEN** availability is requested with `vehicleType=VAN` and 4 LARGE spots are FREE
- **THEN** the response reports 4 free compatible spots

### Requirement: REQ-AVL-003 Counts reflect committed changes
Availability SHALL reflect every committed entry, exit and maintenance change within 2 seconds.

#### Scenario: Count drops after entry
- **WHEN** an entry allocates a COMPACT spot on floor 1
- **THEN** within 2 seconds the floor 1 COMPACT free count is one lower

### Requirement: REQ-AVL-004 Spot listing
The system SHALL list the spots of a floor with their code, type and state, filterable by state and type.

#### Scenario: List free compact spots
- **WHEN** a client lists floor F1 spots with `status=FREE&type=COMPACT`
- **THEN** only FREE COMPACT spots of F1 are returned

### Requirement: REQ-AVL-005 Display-board summary endpoint
Each lot SHALL expose a display-board JSON endpoint returning per-floor free counts by spot type and a `full` flag per vehicle type.

#### Scenario: Board shows full
- **WHEN** no spot compatible with TRUCK is FREE
- **THEN** the display-board endpoint returns `full=true` for TRUCK
