# Quickstart: HV Transformer Registry

**Feature**: [spec.md](./spec.md) | **Contract**: [contracts/openapi.yaml](./contracts/openapi.yaml) |
**Data model**: [data-model.md](./data-model.md)

Validation guide proving the feature works end-to-end. Commands are the npm scripts that the
implementation (see `tasks.md`) must provide.

## Prerequisites

- Node.js 24 LTS and npm
- Docker (for local PostgreSQL 18 and for Testcontainers in tests)

## Setup

```bash
npm ci
docker compose up -d postgres
cp .env.example .env          # DATABASE_URL (app role), DATABASE_MIGRATION_URL (owner), AUTH_*
npm run db:migrate            # runs as the schema owner, grants privileges to asset_data_app
```

## Quality gates (must all pass)

```bash
npm run typecheck             # tsc --noEmit, strict
npm run lint                  # ESLint + Prettier check
npm run test:unit             # domain rules
npm run test:contract         # every endpoint/status vs contracts/openapi.yaml
npm run test:integration      # HTTP → PostgreSQL (Testcontainers)
npm run test:coverage         # ≥ 80% on domain + transformers service + audit
```

## Run locally

```bash
npm run start:dev
curl http://localhost:3000/health/ready      # → {"status":"ok",...}
```

For manual calls a token from the configured identity provider is needed; for local
experiments `npm run dev:token -- --role editor` prints a token signed with a local development
key (only enabled when `AUTH_DEV_KEYS=true`, never in production).

## End-to-end validation scenarios

Each scenario maps to the spec's acceptance scenarios and is automated in `tests/integration`.

| # | Scenario | Steps | Expected |
|---|---|---|---|
| 1 | Register (US1-1) | `POST /v1/transformers` as editor with the example body from the contract | 201, `Location`, `ETag: "1"`, body with `id`, units-suffixed fields |
| 2 | Read back (US1-2) | `GET /v1/transformers/{id}` as viewer | 200, same data, `createdBy`/`updatedBy` set |
| 3 | Duplicate (US1-3) | Repeat #1 with `manufacturer: " siemens energy "` | 409 `/problems/duplicate-transformer` with `existingTransformerId` |
| 4 | Invalid data (US1-4) | `hvRatedVoltageKv: 110`, `lvRatedVoltageKv: 132`, `ratedPowerMva: 0` | 422 `/problems/validation-error`, `errors` lists both fields |
| 5 | List & filter (US2) | Seed 120 transformers; `GET /v1/transformers?status=out_of_service&hvVoltageKvMin=220&hvVoltageKvMax=400&substation=SS-Nord` | Only matching items, correct `totalItems`, newest first |
| 6 | Page size limit (US2-5) | `GET /v1/transformers?pageSize=101` | 422 |
| 7 | Update (US3-1) | `PATCH` model and `location.bay` with `If-Match: "1"` | 200, `ETag: "2"`, history has `updated` entry with previous/new |
| 8 | Stale update (US3-4) | `PATCH` again with `If-Match: "1"` | 412 `/problems/precondition-failed` |
| 9 | Status change (US3-2) | `POST …/status-changes` `{status: out_of_service, reason}` | 200, history has `status_changed` |
| 10 | Final status (US3-3) | Decommission, then try `in_service` | 409 `/problems/invalid-status-transition` |
| 11 | Remove (US4-1) | `POST …/removal` as administrator | 200; viewer `GET` → 404; admin `GET` → 200 with `removed: true` |
| 12 | Restore conflict (US4-3) | Register same manufacturer+serial, then `POST …/restoration` | 409 duplicate |
| 13 | History (US4-4) | `GET …/history` | Entries oldest first, each with actor, time, changes |
| 14 | Roles (FR-024) | Viewer `POST /v1/transformers`; editor `POST …/removal` | 403 |
| 15 | No hard delete (FR-017) | Direct SQL `DELETE`/`TRUNCATE` on `transformers`, `UPDATE`/`DELETE`/`TRUNCATE` on `audit_entries` | Rejected: permission denied for `asset_data_app`, triggers for the owner |

## Performance check (SC-003)

`npm run perf:seed -- 10000` then `npm run perf:check`: 95th percentile of lookups and filtered
list pages under 1 second.
