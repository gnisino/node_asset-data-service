# Data Model: HV Transformer Registry

**Feature**: [spec.md](./spec.md) | **Research**: [research.md](./research.md) | **Date**: 2026-11-15

API field names carry their unit as a suffix (`…Mva`, `…Kv`, `…Hz`), so no physical value is
unit-less (constitution II). Database columns use the same names in `snake_case`.

## Entity: Transformer

| Field (API) | Column | Type (DB) | Required | Rules |
|---|---|---|---|---|
| `id` | `id` | `uuid` PK, default `uuidv7()` | system | Immutable, never reused (FR-002) |
| `manufacturer` | `manufacturer` | `text` | yes | Trimmed, 1–100 chars |
| `model` | `model` | `text` | yes | Trimmed, 1–100 chars |
| `serialNumber` | `serial_number` | `text` | yes | Trimmed, 1–100 chars |
| `yearOfManufacture` | `year_of_manufacture` | `smallint` null | no | 1900 ≤ y ≤ current year |
| `ratedPowerMva` | `rated_power_mva` | `numeric(8,3)` | yes | 0 < v ≤ 2000 |
| `hvRatedVoltageKv` | `hv_rated_voltage_kv` | `numeric(7,3)` | yes | 0 < v ≤ 1200; > LV and > tertiary |
| `lvRatedVoltageKv` | `lv_rated_voltage_kv` | `numeric(7,3)` | yes | 0 < v ≤ 1200; < HV |
| `tertiaryRatedVoltageKv` | `tertiary_rated_voltage_kv` | `numeric(7,3)` null | no | 0 < v ≤ 1200; < HV |
| `ratedFrequencyHz` | `rated_frequency_hz` | `smallint` | yes | 50 or 60 |
| `phases` | `phases` | `smallint` | yes | 1 or 3 |
| `vectorGroup` | `vector_group` | `text` | yes | Pattern VG (below) |
| `coolingType` | `cooling_type` | `text` | yes | Pattern CT (below), upper-cased |
| `location.substation` | `substation` | `text` | yes | Trimmed, 1–100 chars |
| `location.bay` | `bay` | `text` null | no | Trimmed, 1–50 chars |
| `status` | `status` | `text` | yes | `in_service` \| `out_of_service` \| `decommissioned` |
| `statusEffectiveDate` | `status_effective_date` | `date` | yes | ≤ today; defaults to today |
| `version` | `version` | `integer` | system | Starts at 1, +1 per change; exposed as `ETag` |
| `createdAt` / `createdBy` | `created_at` / `created_by` | `timestamptz` / `text` | system | FR-004 |
| `updatedAt` / `updatedBy` | `updated_at` / `updated_by` | `timestamptz` / `text` | system | FR-004 |
| `removedAt` / `removedBy` | `removed_at` / `removed_by` | `timestamptz` / `text` null | system | Set on removal, cleared on restore |
| `removalReason` | `removal_reason` | `text` null | system | Mandatory when removed |

`createdBy`, `updatedBy`, `removedBy` store the token subject; display names live in audit
entries.

**Validation patterns**

- **VG** (vector group, IEC 60076-1):
  `^(?:D|YN|Y|ZN|Z|I)(?:(?:d|yn|y|zn|z|a|i)(?:1[01]|[0-9])){1,2}$`
- **CT** (cooling type, IEC 60076-2): `^[OKL][NFD][AW][NF](?:/[OKL][NFD][AW][NF]){0,2}$`

**Cross-field rules (domain layer)**: `hvRatedVoltageKv > lvRatedVoltageKv`;
`hvRatedVoltageKv > tertiaryRatedVoltageKv` when present; `yearOfManufacture ≤ current year`;
`statusEffectiveDate ≤ today`.

**Database constraints and indexes**

- `CHECK` constraints mirroring every range and enum above (defense in depth).
- `CHECK (hv_rated_voltage_kv > lv_rated_voltage_kv)` and the same for tertiary.
- `CHECK ((removed_at IS NULL) = (removal_reason IS NULL))`.
- Unique partial index `ux_transformers_identity` on
  `(lower(manufacturer), lower(serial_number)) WHERE removed_at IS NULL` (FR-003).
- Indexes: `status`; `lower(manufacturer)`; `hv_rated_voltage_kv`; `(lower(substation), lower(bay))`;
  `(created_at DESC, id DESC)` for default ordering.
- Trigger `trg_transformers_no_delete`: row-level `BEFORE DELETE` → raise exception (FR-017).
- Trigger `trg_transformers_no_truncate`: statement-level `BEFORE TRUNCATE` → raise exception.
- Privileges: role `asset_data_app` (used by the service) has `SELECT, INSERT, UPDATE` only.

## Lifecycle state machine (FR-014, FR-015)

The spec's **Lifecycle Status Change** entity is intentionally not a separate table: each status
change is stored as an [Audit Entry](#entity-audit-entry-fr-020fr-022) with action
`status_changed`, `reason`, author, timestamp and changes for `status` (previous → new) and
`statusEffectiveDate`. This holds every attribute the spec requires without duplicating data
(research R8).

```text
           ┌──────────────┐  status change  ┌────────────────┐
           │  in_service  │ ◀─────────────▶ │ out_of_service │
           └──────┬───────┘                 └───────┬────────┘
                  │                                 │
                  └──────────────┬──────────────────┘
                                 ▼
                        ┌────────────────┐
                        │ decommissioned │  (final: no outgoing transitions)
                        └────────────────┘
```

- Any initial status is allowed at registration (historic assets may be registered as
  `decommissioned`).
- A status change requires `reason` (1–500 chars) and `effectiveDate` (≤ today, default today).
- Changing to the current status → 422 `no-changes` (no-op rejected, same error as a `PATCH`
  that changes nothing); leaving `decommissioned` → 409 `invalid-status-transition`.
- Status is changed **only** via the status-change operation, never via `PATCH`.

## Removal state (FR-017…FR-019)

Orthogonal to the lifecycle status:

| State | `removed_at` | Visible to viewer/editor | Visible to administrator |
|---|---|---|---|
| active | `NULL` | yes | yes |
| removed | set | no (404) | yes, with `visibility=removed\|all` or on direct lookup |

- Remove: administrator, `reason` required, record must be active.
- Restore: administrator, `reason` required, record must be removed, and no active record may share its
  manufacturer + serial number (409 `duplicate-transformer`).

## Entity: Audit Entry (FR-020…FR-022)

Table `audit_entries` (append-only).

| Field (API) | Column | Type (DB) | Notes |
|---|---|---|---|
| `id` | `id` | `uuid` PK, default `uuidv7()` | |
| `transformerId` | `transformer_id` | `uuid` FK → `transformers.id` | `ON DELETE RESTRICT` |
| `action` | `action` | `text` | `created` \| `updated` \| `status_changed` \| `removed` \| `restored` |
| `actor.subject` | `actor_subject` | `text` | Token `sub` |
| `actor.name` | `actor_name` | `text` null | Token `name` / `preferred_username` |
| `occurredAt` | `occurred_at` | `timestamptz` | Default `now()` |
| `reason` | `reason` | `text` null | Required for `status_changed`, `removed`, `restored` |
| `changes` | `changes` | `jsonb` | Array of `{ field, previous, new }` |

- `created`: `changes` lists every initial field with `previous: null`.
- `updated`: only fields whose value actually changed; an update changing nothing is rejected
  as a no-op (no entry written).
- `status_changed`: changes for `status` and `statusEffectiveDate`.
- `removed` / `restored`: change for `removed` (`false`→`true` or vice versa).
- Written in the **same transaction** as the transformer change.
- Triggers `trg_audit_no_update`, `trg_audit_no_delete` (row-level) and `trg_audit_no_truncate`
  (statement-level `BEFORE TRUNCATE`): raise exception (FR-021).
- Privileges: role `asset_data_app` has `SELECT, INSERT` only.
- Index: `(transformer_id, occurred_at, id)` for chronological history.

## Value object: Installation Location

`{ substation: string, bay: string | null }` embedded in Transformer (columns `substation`,
`bay`). Matching in filters is case-insensitive exact.

## Value object: Actor

Derived from the verified JWT: `{ subject, name, roles }`. Roles are cumulative:
`viewer` ⊂ `editor` ⊂ `administrator`.
