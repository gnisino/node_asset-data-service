# Feature Specification: HV Transformer Registry

**Feature Branch**: `001-transformer-registry`

**Created**: 2026-11-15

**Status**: Approved (project owner, 2026-11-18)

## Clarifications

### Session 2026-11-15

- Q: Which roles and permissions are needed to access the registry? → A: Three cumulative roles:
  viewer (read, history), editor (+ register, update, status change), administrator (+ remove,
  restore, view removed records).

**Input**: User description: "HV transformer registry: users can create, view, update and list high-voltage power transformers. Each transformer has nameplate data (manufacturer, model, serial number, year of manufacture, rated power in MVA, rated primary/secondary voltages in kV, frequency in Hz, vector group, cooling type, number of phases), an installation location (substation/bay) and a lifecycle status (in service, out of service, decommissioned). Manufacturer + serial number must be unique. Listing supports pagination and filtering by status, manufacturer, voltage and location. Status changes and all modifications are audited; records are never hard-deleted (soft deletion only for erroneous entries). Components/parts are out of scope for this feature."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Register and view a transformer (Priority: P1)

An asset engineer registers a newly acquired or newly surveyed high-voltage power transformer by
entering its nameplate data, installation location and initial lifecycle status. The system
validates the data, assigns a stable identifier, and the engineer can then open the transformer
record to see all of its data.

**Why this priority**: without registered transformers no other capability (listing, updating,
auditing) has any data to work on. Registering and reading back one record is the smallest slice
that delivers value: a single trusted source of nameplate data.

**Independent Test**: register a transformer with valid nameplate data, then retrieve it by its
identifier and verify every field matches what was entered, including units.

**Acceptance Scenarios**:

1. **Given** no transformer with manufacturer "Siemens Energy" and serial number "TR-2024-0042"
   exists, **When** an engineer registers it with complete, valid nameplate data, **Then** the
   system stores it, assigns a unique identifier and returns the full record.
2. **Given** a registered transformer, **When** a user requests it by its identifier, **Then** the
   system returns all nameplate data with explicit units, its location, its lifecycle status and
   when/by whom it was created and last modified.
3. **Given** a transformer with manufacturer "Siemens Energy" and serial number "TR-2024-0042"
   already exists, **When** an engineer registers another transformer with the same manufacturer
   and serial number (ignoring letter case and surrounding spaces), **Then** the system rejects it
   and explains that the transformer is already registered, referencing the existing record.
4. **Given** an engineer submits a transformer whose HV winding voltage is lower than or equal to
   its LV winding voltage, or whose rated power is zero or negative, **When** the registration is
   submitted, **Then** the system rejects it and lists every invalid field with the reason.
5. **Given** a user requests an identifier that does not exist, **When** the request is made,
   **Then** the system reports that the transformer was not found.

---

### User Story 2 - Find transformers in the fleet (Priority: P2)

A maintenance planner browses the transformer fleet, narrowing the list by lifecycle status,
manufacturer, voltage level and location, in order to plan inspections or answer questions such
as "which 380 kV transformers are out of service in substation X?".

**Why this priority**: once several transformers are registered, finding the right ones is the
most frequent daily activity; it is useful even before editing is available.

**Independent Test**: register a set of transformers with varied attributes, then run filtered
and paginated listings and verify that exactly the expected records are returned, in a stable
order, with correct totals.

**Acceptance Scenarios**:

1. **Given** 120 registered transformers, **When** a user lists transformers without filters,
   **Then** the system returns the first page (default page size 20), the total count, and a way
   to retrieve the following pages.
2. **Given** transformers with different statuses, **When** a user filters by status
   "out of service", **Then** only transformers currently in that status are returned.
3. **Given** transformers of several manufacturers, voltages and substations, **When** a user
   combines filters for manufacturer, HV winding voltage range (e.g. 220–400 kV) and substation,
   **Then** only transformers matching all filters are returned.
4. **Given** a filter that matches no transformer, **When** the listing is requested, **Then** the
   system returns an empty page with a total count of zero (not an error).
5. **Given** a user requests a page size above the maximum (100), **When** the listing is
   requested, **Then** the system rejects the request and states the allowed maximum.

---

### User Story 3 - Update data and lifecycle status (Priority: P3)

An asset engineer corrects or completes nameplate data (e.g. a mistyped model), records that a
transformer was moved to another bay, or changes its lifecycle status (e.g. takes it out of
service for repair, or decommissions it at end of life).

**Why this priority**: data changes over the life of the asset; without updates the registry
becomes stale. It builds on Story 1 but is not required for the registry to be useful for lookup.

**Independent Test**: update fields and change statuses of a registered transformer and verify
that the new values are returned, invalid changes are rejected, and the previous values are
preserved in the change history.

**Acceptance Scenarios**:

1. **Given** a registered transformer, **When** an engineer updates its model and bay, **Then**
   the system stores the new values and records the change with author, time, previous and new
   values.
2. **Given** a transformer "in service", **When** an engineer sets it "out of service" with a
   reason, **Then** the status changes and the transition (from, to, reason, author, time) is
   recorded.
3. **Given** a transformer "decommissioned", **When** an engineer tries to set it back to
   "in service" or "out of service", **Then** the system rejects the change because
   decommissioning is final.
4. **Given** two users opened the same transformer version, **When** the first saves a change and
   the second then saves a change based on the older version, **Then** the second change is
   rejected and the user is told the record was modified in the meantime.
5. **Given** an engineer changes the serial number so that manufacturer + serial number would
   duplicate another active transformer, **When** the update is submitted, **Then** it is rejected
   for duplication.

---

### User Story 4 - Remove erroneous entries and review history (Priority: P4)

A data administrator removes a transformer that was registered by mistake (e.g. a duplicate or a
test entry) so it no longer appears in the registry, while its history is preserved; if the
removal itself was a mistake, the administrator can restore it. Any authorized user can review
the complete change history of a transformer.

**Why this priority**: needed for data quality and auditability, but only after data exists and
is being maintained.

**Independent Test**: remove a transformer, verify it no longer appears in listings or lookups,
verify its history is still available to administrators, restore it and verify it reappears
unchanged.

**Acceptance Scenarios**:

1. **Given** an active transformer, **When** an administrator removes it as erroneous with a
   reason, **Then** it no longer appears in listings or lookups for regular users, and the
   removal (author, time, reason) is recorded.
2. **Given** a removed transformer, **When** an administrator restores it with a reason,
   **Then** it appears again with the same identifier and data, and the restoration (author,
   time, reason) is recorded.
3. **Given** a removed transformer whose manufacturer + serial number has since been registered
   again, **When** an administrator tries to restore it, **Then** the restore is rejected for
   duplication.
4. **Given** a transformer that has been created, updated twice and had one status change,
   **When** a user opens its history, **Then** the system shows four entries in chronological
   order with author, time and the fields changed (previous and new values).

### Edge Cases

- Manufacturer or serial number differing only by letter case or leading/trailing spaces is
  treated as the same identity (e.g. "ABB " and "abb").
- Values outside plausible physical ranges (e.g. rated power 50,000 MVA, HV voltage 5,000 kV,
  frequency 55 Hz, year of manufacture in the future) are rejected rather than stored.
- A unit-less or wrongly-united value (e.g. voltage given in V instead of kV) cannot be entered
  ambiguously: every physical value is captured in its documented unit.
- Cooling type is case-insensitive on input and always stored upper-case (e.g. "onan/onaf" is
  stored as "ONAN/ONAF"). Vector group is case-significant in IEC notation (upper case for the HV
  winding, lower case for the others), so it is accepted only in standard notation; any other
  form (e.g. "YN-d-11", "ynd11") is rejected with the expected format. Surrounding whitespace is
  trimmed for both.
- Changing a status to the status it already has, or submitting an update that changes no
  value, is rejected as a no-op (same error for both) so no meaningless history entry is created.
- Requesting a page beyond the last page returns an empty page, not an error.
- A removed (erroneous) transformer does not block registering a new transformer with the same
  manufacturer + serial number.
- Decommissioned transformers remain visible and searchable (they are real assets with a
  history); only erroneous entries are removed.

## Requirements *(mandatory)*

### Functional Requirements

**Registration and identity**

- **FR-001**: System MUST allow authorized users to register a transformer with: manufacturer,
  model, serial number, year of manufacture, rated power (MVA), HV winding rated voltage (kV),
  LV winding rated voltage (kV), optional tertiary winding rated voltage (kV), rated frequency
  (Hz), vector group, cooling type, number of phases, installation location (substation and bay)
  and initial lifecycle status.
- **FR-002**: System MUST assign each transformer a stable, unique, system-generated identifier
  that never changes and is never reused.
- **FR-003**: System MUST enforce uniqueness of manufacturer + serial number among non-removed
  transformers, comparing case-insensitively and ignoring leading/trailing whitespace.
- **FR-004**: System MUST record for each transformer who created it and when, and who last
  modified it and when.

**Validation**

- **FR-005**: System MUST require manufacturer, model, serial number, rated power, HV and LV
  winding voltages, frequency, number of phases, vector group, cooling type, substation and
  lifecycle status; year of manufacture, tertiary voltage and bay are optional.
- **FR-006**: System MUST validate physical values against plausible ranges: rated power greater
  than 0 and at most 2,000 MVA; winding voltages greater than 0 and at most 1,200 kV; HV winding
  voltage strictly greater than LV winding voltage (and than tertiary voltage, if given);
  frequency exactly 50 or 60 Hz; number of phases 1 or 3; year of manufacture between 1900 and
  the current year.
- **FR-007**: System MUST validate the vector group against IEC 60076-1 notation (e.g. `YNd11`,
  `Dyn11`, `YNyn0d1`) and the cooling type against the IEC 60076-2 four-letter codes (e.g.
  `ONAN`, `ONAF`, `OFAF`, `ODAF`, `OFWF`, `ODWF`, `KNAN`, `KNAF`), accepting combined ratings such
  as `ONAN/ONAF`.
- **FR-008**: System MUST reject invalid input as a whole, reporting every invalid field with a
  human-readable reason; it MUST NOT silently correct or truncate values, except normalizing
  letter case and surrounding whitespace of codes and identifiers.

**Retrieval and listing**

- **FR-009**: Users MUST be able to retrieve a single transformer by its identifier, including all
  attributes with their units, its lifecycle status and the creation/modification metadata.
- **FR-010**: Users MUST be able to list transformers with pagination (default page size 20,
  maximum 100), receiving the total number of matching records.
- **FR-011**: Users MUST be able to filter the list by lifecycle status (one or more), manufacturer,
  HV winding voltage (exact value or minimum/maximum range), substation and bay; multiple filters
  combine with AND.
- **FR-012**: Listings MUST be returned in a stable, deterministic order (default: most recently
  registered first), so that paging through the results never skips or repeats a record when the
  data does not change.

**Updates and lifecycle**

- **FR-013**: Authorized users MUST be able to update any attribute of a transformer except its
  identifier, subject to the same validation and uniqueness rules as registration.
- **FR-014**: System MUST support the lifecycle statuses `in service`, `out of service` and
  `decommissioned`, with these allowed transitions: `in service` ↔ `out of service`, and either
  of them → `decommissioned`; `decommissioned` is final.
- **FR-015**: Every status change MUST include a reason and an effective date (defaulting to the
  current date, never in the future).
- **FR-016**: System MUST detect conflicting concurrent updates and reject an update made on an
  outdated version of the record.

**Removal of erroneous entries**

- **FR-017**: System MUST NOT permanently delete transformer records.
- **FR-018**: Administrators MUST be able to mark a transformer as removed (erroneous entry) with
  a mandatory reason; removed transformers MUST be excluded from retrieval and listings for
  non-administrators.
- **FR-019**: Administrators MUST be able to view removed transformers and restore them with a
  mandatory reason, subject to the uniqueness rule in FR-003.

**Audit**

- **FR-020**: System MUST record an audit entry for every registration, update, status change,
  removal and restoration, containing: author, timestamp, action, and for each changed field its
  previous and new value.
- **FR-021**: Audit entries MUST be immutable and retained for as long as the transformer record
  exists.
- **FR-022**: Users allowed to view a transformer MUST be able to view its complete change history
  in chronological order.

**Access control**

- **FR-023**: Only authenticated users MAY access the registry.
- **FR-024**: Permissions MUST be role-based with three cumulative roles:
  - *viewer*: retrieve and list transformers, view change history;
  - *editor*: viewer permissions plus register, update and change lifecycle status;
  - *administrator*: editor permissions plus remove and restore erroneous entries and view
    removed transformers.
- **FR-025**: System MUST reject any action the user's role does not allow, without revealing
  data the user is not permitted to see.

### Key Entities

- **Transformer**: a single high-voltage power transformer as a physical asset. Identified by a
  system identifier and by its natural identity (manufacturer + serial number). Holds nameplate
  data (model, year of manufacture, rated power in MVA, HV/LV/optional tertiary rated voltages in
  kV, rated frequency in Hz, vector group, cooling type, number of phases), installation location,
  current lifecycle status, a removed flag, a version for concurrency control, and
  creation/modification metadata.
- **Installation Location**: where the transformer is installed, described by a substation
  identifier/name and an optional bay identifier. In this feature it is part of the transformer
  record, not a separately managed entity.
- **Lifecycle Status Change**: a transition of a transformer between lifecycle statuses, with
  previous and new status, reason, effective date, author and recording time. Every status
  change is one entry of the transformer's change history (see Audit Entry).
- **Audit Entry**: an immutable record of one change to a transformer: action type, author,
  timestamp and the list of changed fields with previous and new values.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of rejected registrations and updates report every invalid field in the first
  response, so a user never needs repeated attempts to discover further errors in the same input.
- **SC-002**: 100% of registrations with a duplicate manufacturer + serial number, or with values
  outside the documented ranges, are rejected; none are stored.
- **SC-003**: With a fleet of 10,000 transformers, 95% of lookups and filtered listing pages are
  returned to the user in under 1 second.
- **SC-004**: For any transformer, its full change history can be reconstructed: every
  registration, update, status change, removal and restoration is attributable to an author and
  a time, with 0 changes missing from the history.
- **SC-005**: No transformer record is ever permanently lost: 100% of removed entries can be
  found and restored by an administrator.
- **SC-006**: Every filter in FR-011, alone and combined, returns exactly the matching transformers
  in 100% of the acceptance test cases (no missing and no extra records).

## Assumptions

- Users are utility or service-company staff (asset engineers, maintenance planners, data
  administrators) who know transformer nameplate terminology.
- Transformers are two-winding units, optionally with a tertiary winding; autotransformers are
  recorded with the same fields. Multi-rating details beyond a combined cooling code (e.g. power
  per cooling stage) are out of scope.
- "Voltage" filtering refers to the HV winding rated voltage, the value commonly used to classify
  transformers (e.g. "380 kV transformers").
- Installation location is captured as substation and bay identifiers entered by users; a managed
  registry of substations/bays is a possible future feature.
- Units are fixed per field (MVA, kV, Hz); no unit conversion is offered in this feature.
- Authentication is provided by the organization's existing identity provider; this feature only
  consumes the user identity and roles.
- Components and parts (bushings, tap changers, cooling equipment, etc.), measurements,
  maintenance records and bulk import from files or external systems are out of scope for this
  feature.
- Audit history is retained for the lifetime of the transformer record; no automatic purge.
- This feature delivers the service and its API only; no user interface. User-experience targets
  (e.g. time to register a transformer from its nameplate, finding a transformer on the first
  attempt) will be defined with the future client application.
