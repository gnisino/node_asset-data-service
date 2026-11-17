---

description: "Task list for feature 001 — HV Transformer Registry"
---

# Tasks: HV Transformer Registry

**Input**: Design documents from `/specs/001-transformer-registry/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/openapi.yaml](./contracts/openapi.yaml),
[quickstart.md](./quickstart.md)

**Status**: Approved (project owner, 2026-11-18), after `/speckit-analyze` remediation

**Tests**: REQUIRED. The constitution (principle IV, Test-First, NON-NEGOTIABLE) mandates unit,
contract and integration tests written **before** the implementation and observed failing.

**Organization**: Tasks are grouped by user story so each story can be implemented, tested and
demonstrated independently.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: User story the task belongs to (US1–US4 from spec.md)
- Paths are relative to the repository root (single project, see plan.md)

## Conventions used by every task

- TypeScript `strict`, no `any`; CommonJS output (research R1).
- API field names and shapes come **only** from `openapi/openapi.yaml`; TypeScript types are
  imported from `src/generated/openapi.d.ts` (never hand-written duplicates).
- Domain code in `src/transformers/domain/` MUST NOT import NestJS, Drizzle or `pg`.
- Every mutation runs in one DB transaction together with its audit entry (research R8).
- Errors are thrown as domain errors and rendered as RFC 9457 problems by the global filter.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and tooling

- [ ] T001 Create `package.json` (name `node_asset-data-service`, `"private": true`, `engines.node: ">=24"`, **no** `"type": "module"`) with scripts: `build` (`nest build`), `start` (`node dist/main.js`), `start:dev` (`nest start --watch`), `typecheck` (`tsc --noEmit -p tsconfig.json`), `lint` (`eslint . && prettier --check .`), `format` (`prettier --write .`), `test:unit`, `test:contract`, `test:integration` (`vitest run --project <name>`), `test:coverage` (`vitest run --coverage`), `db:generate` (`drizzle-kit generate`), `db:migrate` (`tsx scripts/migrate.ts`), `openapi:types` (`openapi-typescript openapi/openapi.yaml -o src/generated/openapi.d.ts`), `openapi:check`, `dev:token`, `perf:seed`, `perf:check` (`tsx scripts/<name>.ts`) in package.json
- [ ] T002 Install dependencies and commit `package-lock.json`: runtime `@nestjs/common @nestjs/core @nestjs/platform-express @nestjs/config @nestjs/terminus reflect-metadata rxjs drizzle-orm pg ajv ajv-formats yaml jose nestjs-pino pino-http zod`; dev `typescript @nestjs/cli @nestjs/testing @types/node @types/pg @types/express drizzle-kit openapi-typescript tsx vitest @vitest/coverage-v8 unplugin-swc @swc/core supertest @types/supertest @testcontainers/postgresql eslint typescript-eslint @eslint/js eslint-config-prettier prettier` in package.json
- [ ] T003 [P] Create `tsconfig.json` (`strict: true`, `target: ES2023`, `module: nodenext`, `moduleResolution: nodenext`, `experimentalDecorators: true`, `emitDecoratorMetadata: true`, `esModuleInterop: true`, `skipLibCheck: true`, `noUncheckedIndexedAccess: true`, `outDir: dist`, includes `src`, `tests`, `scripts`), `tsconfig.build.json` (only `src`, excludes tests) and `nest-cli.json` (`sourceRoot: src`, `compilerOptions.tsConfigPath: tsconfig.build.json`) in repository root
- [ ] T004 [P] Create `eslint.config.mjs` (flat config: `@eslint/js` recommended + `typescript-eslint` `strictTypeChecked`, `@typescript-eslint/no-explicit-any: error`, `eslint-config-prettier` last; ignore `dist`, `coverage`, `src/generated`, `drizzle`), `.prettierrc` (`singleQuote: true`, `printWidth: 100`, `trailingComma: all`) and `.prettierignore` in repository root
- [ ] T005 [P] Create `vitest.config.ts` with `unplugin-swc` (decorator metadata) and three projects: `unit` (`tests/unit/**/*.test.ts`), `contract` (`tests/contract/**/*.test.ts`) and `integration` (`tests/integration/**/*.test.ts`), the latter two with `globalSetup: tests/support/global-setup.ts`, `testTimeout: 60000`, `fileParallelism: false`; coverage provider `v8` with thresholds `lines/functions/branches/statements: 80` on `include: ['src/transformers/domain/**', 'src/transformers/transformers.service.ts', 'src/audit/**']` (domain rules plus the business rules living in services: visibility, removed → 404, transactions with audit), measured over all three projects, in vitest.config.ts
- [ ] T006 [P] Create `docker-compose.yml` (service `postgres`, image `postgres:18`, port 5432, db/owner user/password `asset_data`, named volume, mounting `docker/postgres/init/`), `docker/postgres/init/01-app-role.sql` (creates login role `asset_data_app` with password `asset_data_app`, local development only, no privileges) and `.env.example` (`DATABASE_URL=postgres://asset_data_app:asset_data_app@localhost:5432/asset_data`, `DATABASE_MIGRATION_URL=postgres://asset_data:asset_data@localhost:5432/asset_data`, `AUTH_ISSUER`, `AUTH_AUDIENCE`, `AUTH_JWKS_URI`, `AUTH_ROLES_CLAIM=roles`, `PORT=3000`, `LOG_LEVEL=info`, `AUTH_DEV_KEYS=false`); add `!.env.example` and `.dev-keys/` to `.gitignore` (the existing `.env.*` rule would otherwise ignore the example)
- [ ] T007 Copy `specs/001-transformer-registry/contracts/openapi.yaml` to `openapi/openapi.yaml`, run `npm run openapi:types` to create `src/generated/openapi.d.ts`, and implement `openapi:check` in `scripts/openapi-check.ts` (fails if `openapi/openapi.yaml` and the feature contract differ, or if `src/generated/openapi.d.ts` is stale)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Infrastructure every user story needs: config, database schema and guards, test
harness, error format, validation, concurrency helpers, auth, health, audit append, diffing.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Tests first (must fail before the corresponding implementation)

- [ ] T008 [P] Write unit tests for the environment schema (the service schema does not read `DATABASE_MIGRATION_URL`; required `DATABASE_URL`, `AUTH_ISSUER`, `AUTH_AUDIENCE`, `AUTH_JWKS_URI`; defaults `AUTH_ROLES_CLAIM=roles`, `PORT=3000`, `LOG_LEVEL=info`, `AUTH_DEV_KEYS=false`; invalid URL rejected) in tests/unit/config/config.schema.test.ts
- [ ] T009 [P] Write unit tests for the contract validator: loads `openapi/openapi.yaml`; `TransformerCreate` with several invalid fields returns **all** errors as `{ field, message }` with dotted paths (e.g. `location.substation`); `additionalProperties: false` rejects unknown fields; `ratedPowerMva: 0` and `hvRatedVoltageKv: 1201` rejected; vector group `Dyn11`, `YNyn0d1`, `YNa0d1`, `Ii0` accepted and `YNd12`, `yNd1`, `Dyn` rejected; cooling `onan/onaf` accepted, `ONA` rejected; query coercion of `page=2` and repeated `status`; **no type coercion in bodies**: `ratedPowerMva: "250"`, `phases: "3"` and `yearOfManufacture: true` are rejected (not converted) in tests/unit/validation/contract-validator.test.ts
- [ ] T010 [P] Write unit tests for ETag helpers: `formatEtag(3) === '"3"'`; `requireIfMatch` accepts the strong tag `"3"` and treats a weak tag (`W/"3"`) as not matching (412); missing header → `PreconditionRequiredError` (428); non-matching version → `PreconditionFailedError` (412) in tests/unit/http/etag.test.ts
- [ ] T011 [P] Write unit tests for pagination: defaults `page=1`, `pageSize=20`; `totalPages = ceil(totalItems / pageSize)`; `offset = (page-1) * pageSize`; page beyond last yields empty items with correct totals in tests/unit/common/pagination.test.ts
- [ ] T012 [P] Write unit tests for the problem-details filter: each domain error maps to its status and `type` (`/problems/validation-error` 422 with `errors`, `/problems/duplicate-transformer` 409 with `existingTransformerId`, `/problems/invalid-status-transition` 409, `/problems/invalid-removal-state` 409, `/problems/no-changes` 422, `/problems/not-found` 404, `/problems/precondition-failed` 412, `/problems/precondition-required` 428, `/problems/unauthorized` 401, `/problems/forbidden` 403, malformed JSON `/problems/bad-request` 400, unknown error `/problems/internal-error` 500 without stack trace); content type `application/problem+json`; `requestId` and `instance` set in tests/unit/common/problem-details.filter.test.ts
- [ ] T013 [P] Write unit tests for auth: roles are cumulative (`administrator` satisfies `editor` and `viewer`; `editor` satisfies `viewer`; `viewer` does not satisfy `editor`); roles read from the configured claim (string array or space-separated string); `Actor` built from `sub` and `name` → `preferred_username` → `null`; token with wrong `iss`/`aud`, expired, or `alg` outside `RS256`/`ES256` → 401 in tests/unit/auth/auth.test.ts
- [ ] T014 [P] Write unit tests for the audit diff: `diffCreated(snapshot)` lists every field with `previous: null`; `diffUpdated(before, after)` lists only changed fields using dotted paths (`location.bay`), compares numbers by value (`380` vs `380.000`), and returns an empty array when nothing changed in tests/unit/transformers/diff.test.ts

### Implementation

- [ ] T015 Implement the Zod environment schema and `ConfigModule.forRoot({ validate })` wiring in src/config/config.schema.ts
- [ ] T016 Define Drizzle tables in src/database/schema.ts exactly as data-model.md: `transformers` — `id uuid PK default uuidv7()`, `manufacturer text` "Trimmed, 1–100 chars", `model text` "Trimmed, 1–100 chars", `serial_number text` "Trimmed, 1–100 chars", `year_of_manufacture smallint null` "1900 ≤ y ≤ current year", `rated_power_mva numeric(8,3)` "0 < v ≤ 2000", `hv_rated_voltage_kv numeric(7,3)` "0 < v ≤ 1200; > LV and > tertiary", `lv_rated_voltage_kv numeric(7,3)` "0 < v ≤ 1200; < HV", `tertiary_rated_voltage_kv numeric(7,3) null` "0 < v ≤ 1200; < HV", `rated_frequency_hz smallint` "50 or 60", `phases smallint` "1 or 3", `vector_group text`, `cooling_type text`, `substation text` "Trimmed, 1–100 chars", `bay text null` "Trimmed, 1–50 chars", `status text` "`in_service` | `out_of_service` | `decommissioned`", `status_effective_date date`, `version integer` "Starts at 1, +1 per change", `created_at timestamptz`, `created_by text`, `updated_at timestamptz`, `updated_by text`, `removed_at timestamptz null`, `removed_by text null`, `removal_reason text null`, with CHECK constraints for every range/enum, `CHECK (hv_rated_voltage_kv > lv_rated_voltage_kv)`, the same for tertiary, and `CHECK ((removed_at IS NULL) = (removal_reason IS NULL))`; `audit_entries` — `id uuid PK default uuidv7()`, `transformer_id uuid FK → transformers.id ON DELETE RESTRICT`, `action text` "`created` | `updated` | `status_changed` | `removed` | `restored`", `actor_subject text`, `actor_name text null`, `occurred_at timestamptz default now()`, `reason text null`, `changes jsonb`; indexes `status`, `lower(manufacturer)`, `hv_rated_voltage_kv`, `(lower(substation), lower(bay))`, `(created_at DESC, id DESC)`, `(transformer_id, occurred_at, id)`
- [ ] T017 Create `drizzle.config.ts` and generate the initial migration with `npm run db:generate` into drizzle/0000_initial.sql
- [ ] T018 Write the custom migration (`drizzle-kit generate --custom`) adding unique partial index `ux_transformers_identity` on `(lower(manufacturer), lower(serial_number)) WHERE removed_at IS NULL`; row-level triggers `trg_transformers_no_delete` (BEFORE DELETE) and `trg_audit_no_update` / `trg_audit_no_delete` (BEFORE UPDATE / BEFORE DELETE on `audit_entries`); statement-level triggers `trg_transformers_no_truncate` and `trg_audit_no_truncate` (BEFORE TRUNCATE ... FOR EACH STATEMENT); all raise an exception; and grants for role `asset_data_app`: `GRANT SELECT, INSERT, UPDATE ON transformers` and `GRANT SELECT, INSERT ON audit_entries` (no DELETE, no TRUNCATE; migration fails with a clear message if the role does not exist) in drizzle/0001_guards.sql
- [ ] T019 Implement `scripts/migrate.ts` (applies `drizzle/` with drizzle-orm's migrator using `DATABASE_MIGRATION_URL`, the schema owner; never `DATABASE_URL`) and the `DatabaseModule` providing a `pg.Pool`, the Drizzle instance and `withTransaction(fn)` in src/database/database.module.ts
- [ ] T020 Build the test harness: Testcontainers `postgres:18` started once, role `asset_data_app` created, migrations applied as the container superuser, and the app pointed at `asset_data_app` in tests/support/global-setup.ts; `resetDatabase()` using the superuser session with `SET session_replication_role = replica` (disables triggers for that session only) followed by `DELETE` of both tables, never via the app role, in tests/support/database.ts, local key pair + JWKS + `tokenFor(role, { sub, name })` in tests/support/tokens.ts, `createTestApp()` (Nest app with the JWKS provider overridden) in tests/support/app.ts, `expectMatchesContract(operationId, status, response)` (Ajv response validation against `openapi/openapi.yaml`, including `application/problem+json`) in tests/support/contract.ts, and `validTransformer(overrides)` builder based on the contract example in tests/support/fixtures.ts
- [ ] T021 Write integration tests for database guards: as the owner, `DELETE FROM transformers` and `TRUNCATE transformers` fail; `UPDATE`/`DELETE`/`TRUNCATE` on `audit_entries` fail; as `asset_data_app`, `DELETE` and `TRUNCATE` on both tables fail with permission denied; two active rows with manufacturer/serial differing only by case violate `ux_transformers_identity`; a removed row does not; CHECK constraints reject `rated_power_mva = 0` and `hv <= lv` in tests/integration/database-guards.test.ts
- [ ] T022 [P] Implement the contract validator (Ajv 2020-12 via `ajv/dist/2020`, `allErrors: true`, `ajv-formats`, OpenAPI keywords tolerated, component schemas compiled by name, error paths converted to dotted field names; two Ajv instances: one for request bodies with `coerceTypes: false` and `useDefaults: false`, one for query/path parameters with `coerceTypes: 'array'` because those values always arrive as strings — constitution II forbids silently coercing body values) and `ValidateBody(schemaName, domainCheck?)` / `ValidateQuery(operationId, domainCheck?)` pipes that merge schema and domain errors into one `ValidationError` in src/common/validation/contract-validator.ts and src/common/validation/validate.pipes.ts
- [ ] T023 [P] Implement `formatEtag`, `requireIfMatch(header, currentVersion)` and the `PreconditionRequiredError` / `PreconditionFailedError` classes in src/common/http/etag.ts
- [ ] T024 [P] Implement `PageRequest`, `toOffset()` and `buildPage(items, totalItems, pageRequest)` in src/common/pagination.ts
- [ ] T025 [P] Implement domain errors (`ValidationError(errors[])`, `DuplicateTransformerError(existingTransformerId)`, `InvalidStatusTransitionError(from, to)`, `InvalidRemovalStateError`, `TransformerNotFoundError`, `NoChangesError` mapped to 422 `/problems/no-changes`) in src/transformers/domain/errors.ts
- [ ] T026 Implement the RFC 9457 global exception filter and error→problem mapping (depends on T023, T025) in src/common/problem-details/problem-details.filter.ts
- [ ] T027 [P] Implement request-id middleware (reads/creates `X-Request-Id`, echoes it in the response) in src/common/http/request-id.middleware.ts and the `nestjs-pino` configuration (JSON logs, `LOG_LEVEL`, request id as `reqId`, redact `req.headers.authorization` and `req.headers.cookie`) in src/logging/logger.config.ts
- [ ] T028 [P] Implement auth: `JWKS_RESOLVER` provider (`createRemoteJWKSet(AUTH_JWKS_URI)`) in src/auth/jwks.provider.ts, `Actor` in src/auth/actor.ts, global `AuthGuard` (jose `jwtVerify` with issuer, audience, algorithms `RS256`/`ES256`; skips `@Public()`) in src/auth/auth.guard.ts, `@Roles()` + cumulative `RolesGuard` in src/auth/roles.guard.ts, `@Public()` in src/auth/public.decorator.ts, `@CurrentActor()` param decorator in src/auth/current-actor.decorator.ts
- [ ] T029 [P] Implement `diffCreated` and `diffUpdated` in src/transformers/domain/diff.ts
- [ ] T030 Implement `AuditRepository.append(tx, { transformerId, action, actor, reason, changes })` and `AuditModule` in src/audit/audit.repository.ts and src/audit/audit.module.ts
- [ ] T031 Write contract tests for `liveness` and `readiness` (200 with running DB, no token required) in tests/contract/health.contract.test.ts
- [ ] T032 Implement `HealthController` (`GET /health/live`, `GET /health/ready` with terminus DB ping, `@Public()`) in src/health/health.controller.ts and src/health/health.module.ts
- [ ] T033 Implement `AppModule` (config, logger, database, auth guards as `APP_GUARD`, filter as `APP_FILTER`, health, audit) in src/app.module.ts and `main.ts` (express JSON parser also accepting `application/merge-patch+json`, `enableShutdownHooks`, port from config) in src/main.ts

**Checkpoint**: `npm run typecheck && npm run lint && npm run test:unit && npm run test:contract && npm run test:integration` pass; the app starts and `/health/ready` returns 200.

---

## Phase 3: User Story 1 — Register and view a transformer (Priority: P1) 🎯 MVP

**Goal**: Editors register transformers with validated nameplate data; viewers read them back.

**Independent Test**: register a transformer with valid data, retrieve it by id and verify every
field and unit; duplicates and invalid data are rejected with all reasons (quickstart scenarios
1–4).

### Tests for User Story 1 (write first, must fail) ⚠️

- [ ] T034 [P] [US1] Write unit tests for nameplate rules and registration: `hvRatedVoltageKv > lvRatedVoltageKv`; `hvRatedVoltageKv > tertiaryRatedVoltageKv` when present; `yearOfManufacture ≤ current year`; `statusEffectiveDate ≤ today` and defaults to today; strings trimmed and blank-after-trim rejected; `coolingType` upper-cased; all violated rules reported together; new entity has `version: 1` and `createdBy`/`updatedBy` = actor subject in tests/unit/transformers/registration.test.ts
- [ ] T035 [P] [US1] Write contract tests for `registerTransformer` (201 with `Location` and `ETag: "1"`, 400 malformed JSON, 401 no token, 403 viewer, 409 duplicate, 422 invalid) and `getTransformer` (200 with `ETag`, 401, 404 unknown id, 404 non-UUID id) in tests/contract/register-get.contract.test.ts
- [ ] T036 [P] [US1] Write integration tests for US1 acceptance scenarios 1–5 and edge cases: duplicate with `" siemens energy "` / `"tr-2024-0042"` → 409 with `existingTransformerId`; HV ≤ LV plus `ratedPowerMva: 0` → one 422 listing both fields; out-of-range values (50,000 MVA, 5,000 kV, 55 Hz, future year) rejected; a `created` audit entry with every field (`previous: null`) and the actor is stored in the same transaction in tests/integration/us1-register-view.test.ts

### Implementation for User Story 1

- [ ] T037 [P] [US1] Implement cross-field nameplate rules and normalization (trim, upper-case `coolingType`, default `statusEffectiveDate` = today) as pure functions taking `today` as a parameter in src/transformers/domain/nameplate-rules.ts
- [ ] T038 [US1] Implement the `Transformer` entity type and `registerTransformer(input, actor, today)` returning the new snapshot or throwing `ValidationError` (depends on T037) in src/transformers/domain/transformer.ts
- [ ] T039 [US1] Implement `TransformersRepository.insert(tx, snapshot)` and `findById(id, { includeRemoved })` with numeric string→number mapping and translation of unique violation on `ux_transformers_identity` (SQLSTATE `23505`) into `DuplicateTransformerError` carrying the existing id in src/transformers/transformers.repository.ts
- [ ] T040 [US1] Implement `toTransformerResponse(snapshot, actor)` (nested `location`, `removed`, `removal` only for administrators) in src/transformers/transformer.mapper.ts
- [ ] T041 [US1] Implement `TransformersService.register(input, actor)` (transaction: insert + `created` audit entry via `diffCreated`) and `get(id, actor)` (removed records → `TransformerNotFoundError` unless administrator) in src/transformers/transformers.service.ts
- [ ] T042 [US1] Implement `POST /v1/transformers` (`@Roles('editor')`, `ValidateBody('TransformerCreate', nameplate domain check)`, 201 + `Location: /v1/transformers/{id}` + `ETag`) and `GET /v1/transformers/:transformerId` (`@Roles('viewer')`, non-UUID → 404, `ETag`) in src/transformers/transformers.controller.ts
- [ ] T043 [US1] Create `TransformersModule` and register it in `AppModule` in src/transformers/transformers.module.ts and src/app.module.ts

**Checkpoint**: US1 tests pass; quickstart scenarios 1–4 succeed. MVP deliverable.

---

## Phase 4: User Story 2 — Find transformers in the fleet (Priority: P2)

**Goal**: Viewers list transformers with pagination and filters.

**Independent Test**: seed varied transformers, run filtered/paginated listings and verify exact
results, totals and stable order (quickstart scenarios 5–6).

### Tests for User Story 2 (write first, must fail) ⚠️

- [ ] T044 [P] [US2] Write contract tests for `listTransformers`: 200 page shape (`items`, `page`, `pageSize`, `totalItems`, `totalPages`), 401, 403 for `visibility=removed` as editor, 422 for `pageSize=101`, `page=0`, unknown `status` value, and `hvVoltageKvMin > hvVoltageKvMax` in tests/contract/list.contract.test.ts
- [ ] T045 [P] [US2] Write integration tests for US2 acceptance scenarios 1–5 plus: each filter alone (`status` single and repeated, `manufacturer` case-insensitive, `hvVoltageKv` exact, `hvVoltageKvMin`/`hvVoltageKvMax` inclusive, `substation`, `bay` case-insensitive); combined filters (AND); page beyond last → empty page with totals; ordering newest first with id tiebreak and no duplicates/skips across pages; seeding helper `seedTransformers(n, variations)` in tests/support/seed.ts, tests in tests/integration/us2-list-filter.test.ts

### Implementation for User Story 2

- [ ] T046 [US2] Add the `listTransformers` query validation (`ValidateQuery('listTransformers', check hvVoltageKvMin ≤ hvVoltageKvMax)`) and the `TransformerFilters` type in src/transformers/transformer-filters.ts
- [ ] T047 [US2] Implement `TransformersRepository.list(filters, pageRequest, visibility)` (case-insensitive equality via `lower()`, `visibility` `active` → `removed_at IS NULL`, `removed` → `IS NOT NULL`, `all` → no condition; `ORDER BY created_at DESC, id DESC`; `COUNT(*)` with the same filters) in src/transformers/transformers.repository.ts
- [ ] T048 [US2] Implement `TransformersService.list(query, actor)` (`visibility` other than `active` requires administrator → `ForbiddenException` problem 403) in src/transformers/transformers.service.ts
- [ ] T049 [US2] Implement `GET /v1/transformers` (`@Roles('viewer')`, returns `buildPage`) in src/transformers/transformers.controller.ts

**Checkpoint**: US1 and US2 tests pass independently; quickstart scenarios 5–6 succeed.

---

## Phase 5: User Story 3 — Update data and lifecycle status (Priority: P3)

**Goal**: Editors update nameplate/location data and change lifecycle status with optimistic
concurrency and full audit.

**Independent Test**: update fields and statuses of a transformer, verify new values, rejected
invalid changes, 412 on stale versions and history entries (quickstart scenarios 7–10).

### Tests for User Story 3 (write first, must fail) ⚠️

- [ ] T050 [P] [US3] Write unit tests for the lifecycle state machine: allowed `in_service` ↔ `out_of_service`, both → `decommissioned`; `decommissioned` → anything rejected; change to the same status → `NoChangesError` (422, not 409); `reason` 1–500 chars required; `effectiveDate` ≤ today, defaults to today in tests/unit/transformers/lifecycle.test.ts
- [ ] T051 [P] [US3] Write unit tests for merge-patch updates: `null` clears optional fields (`yearOfManufacture`, `tertiaryRatedVoltageKv`, `location.bay`); `null` on a required field → `ValidationError`; nested `location` merge keeps untouched keys; cross-field rules re-checked on the merged result (e.g. patching `lvRatedVoltageKv` above the stored HV); a patch producing no change → `NoChangesError`; `version` +1 and `updatedBy` set in tests/unit/transformers/update.test.ts
- [ ] T052 [P] [US3] Write contract tests for `updateTransformer` (200 + new `ETag`, 400, 401, 403 viewer, 404, 409 duplicate serial, 412 stale, 422 invalid/`status` in body/no-op, 428 missing `If-Match`) and `changeTransformerStatus` (200, 401, 403, 404, 409 leaving `decommissioned`, 412, 422 missing reason/future date, 422 `/problems/no-changes` for the current status, 428) in tests/contract/update-status.contract.test.ts
- [ ] T053 [P] [US3] Write integration tests for US3 acceptance scenarios 1–5: audit `updated` entry contains only changed fields with previous/new; `status_changed` entry has `reason` and changes for `status` and `statusEffectiveDate`; two concurrent `PATCH` requests with the same `If-Match` → exactly one 200 and one 412 in tests/integration/us3-update-status.test.ts

### Implementation for User Story 3

- [ ] T054 [P] [US3] Implement `assertTransition(from, to)` and `changeStatus(snapshot, { status, reason, effectiveDate }, actor, today)` in src/transformers/domain/lifecycle.ts
- [ ] T055 [US3] Implement `applyPatch(snapshot, patch, actor, today)` (RFC 7396 semantics, re-validation through nameplate rules, `NoChangesError` on no-op) in src/transformers/domain/transformer.ts
- [ ] T056 [US3] Implement `TransformersRepository.updateIfVersion(tx, id, expectedVersion, snapshot)` (`UPDATE … WHERE id = $1 AND version = $2 AND removed_at IS NULL`; 0 rows → re-read to distinguish `TransformerNotFoundError` from `PreconditionFailedError`; unique violation → `DuplicateTransformerError`) in src/transformers/transformers.repository.ts
- [ ] T057 [US3] Implement `TransformersService.update(id, patch, ifMatch, actor)` and `changeStatus(id, request, ifMatch, actor)` (transaction: version check, update, audit `updated` / `status_changed` with `diffUpdated`) in src/transformers/transformers.service.ts
- [ ] T058 [US3] Implement `PATCH /v1/transformers/:transformerId` (`@Roles('editor')`, `application/merge-patch+json`, `ValidateBody('TransformerPatch')`, `If-Match`) and `POST /v1/transformers/:transformerId/status-changes` (`@Roles('editor')`, `ValidateBody('StatusChangeRequest')`, `If-Match`), both returning the transformer with the new `ETag`, in src/transformers/transformers.controller.ts

**Checkpoint**: US1–US3 tests pass; quickstart scenarios 7–10 succeed.

---

## Phase 6: User Story 4 — Remove erroneous entries and review history (Priority: P4)

**Goal**: Administrators soft-remove and restore erroneous entries; users read the full
chronological history.

**Independent Test**: remove a transformer, verify it disappears for non-admins while its history
remains for admins, restore it, and read the history (quickstart scenarios 11–13).

### Tests for User Story 4 (write first, must fail) ⚠️

- [ ] T059 [P] [US4] Write unit tests for removal rules: remove requires an active record and `reason` 1–500 chars; restore requires a removed record and `reason` 1–500 chars; both increment `version`; removal fields set/cleared together in tests/unit/transformers/removal.test.ts
- [ ] T060 [P] [US4] Write contract tests for `removeTransformer` and `restoreTransformer` (200, 401, 403 editor, 404, 409 wrong state/duplicate on restore, 412, 422 missing reason, 428) and `getTransformerHistory` (200 page of `AuditEntry`, 401, 404 removed transformer for viewer, 422 bad paging) in tests/contract/removal-history.contract.test.ts
- [ ] T061 [P] [US4] Write integration tests for US4 acceptance scenarios 1–4: removed record → viewer `GET`/history 404, absent from default listing, visible to administrator via `GET` and `visibility=removed`; same manufacturer+serial can be registered again after removal; restore then conflicts with 409; history after create + 2 updates + 1 status change has 4 entries oldest first with actor, time and changes in tests/integration/us4-removal-history.test.ts

### Implementation for User Story 4

- [ ] T062 [P] [US4] Implement `removeTransformer(snapshot, reason, actor, now)` and `restoreTransformer(snapshot, actor)` in src/transformers/domain/removal.ts
- [ ] T063 [US4] Extend `TransformersRepository` with `updateRemovalIfVersion(tx, id, expectedVersion, snapshot)` (no `removed_at IS NULL` filter; unique violation on restore → `DuplicateTransformerError`) in src/transformers/transformers.repository.ts
- [ ] T064 [US4] Implement `TransformersService.remove(id, reason, ifMatch, actor)` and `restore(id, reason, ifMatch, actor)` (transaction + audit `removed` / `restored` with `reason` and change `removed: false→true` / `true→false`) in src/transformers/transformers.service.ts
- [ ] T065 [US4] Implement `POST /v1/transformers/:transformerId/removal` and `POST /v1/transformers/:transformerId/restoration` (`@Roles('administrator')`, `ValidateBody('ReasonRequest')`, `If-Match`) in src/transformers/transformers.controller.ts
- [ ] T066 [US4] Implement `AuditRepository.history(transformerId, pageRequest)` (`ORDER BY occurred_at ASC, id ASC` + count) and `GET /v1/transformers/:transformerId/history` (`@Roles('viewer')`, removed transformer → 404 unless administrator) in src/audit/audit.repository.ts and src/audit/history.controller.ts

**Checkpoint**: all four stories pass their tests; quickstart scenarios 11–15 succeed.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T067 [P] Write a meta contract test asserting every `operationId` × documented response status in `openapi/openapi.yaml` is exercised by at least one contract test (tests register coverage via `expectMatchesContract`) in tests/contract/contract-coverage.test.ts
- [ ] T068 [P] Implement the development token issuer: `scripts/dev-token.ts` creates/reuses an ES256 key pair in `.dev-keys/` and prints a token for `--role <viewer|editor|administrator>`; when `AUTH_DEV_KEYS=true` (refused if `NODE_ENV=production`) the `JWKS_RESOLVER` uses that local key in src/auth/jwks.provider.ts
- [ ] T069 [P] Implement `scripts/perf-seed.ts` (inserts N varied transformers through the repository) and `scripts/perf-check.ts` (measures p95 of 200 lookups and 200 filtered list pages; exits non-zero if ≥ 1 s) for SC-003
- [ ] T070 [P] Document local setup, environment variables, npm scripts and the API contract location in README.md, and add the build/test commands to CLAUDE.md
- [ ] T071 Security pass: confirm logs never contain tokens (test with a captured logger) and 500 responses expose no stack traces or SQL in tests/integration/security.test.ts
- [ ] T072 Run all gates (`npm run openapi:check`, `typecheck`, `lint`, `test:unit`, `test:contract`, `test:integration`, `test:coverage`), then quickstart.md scenarios 1–15 and `perf:seed 10000` + `perf:check`; record results in specs/001-transformer-registry/quickstart.md

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies. T001 → T002 → (T003–T006 in parallel) → T007.
- **Foundational (Phase 2)**: depends on Setup; blocks all stories. Tests T008–T014 first (parallel),
  then T015 → T016 → T017 → T018 → T019 → T020 → T021; T022–T025, T027–T029 in parallel after T020;
  T026 after T023+T025; T030 after T019; T031 → T032 → T033.
- **User Stories (Phases 3–6)**: depend on Foundational. Recommended order P1 → P2 → P3 → P4.
- **Polish (Phase 7)**: after the desired stories are complete.

### User Story Dependencies

- **US1 (P1)**: only Foundational.
- **US2 (P2)**: Foundational; uses the repository and mapper created in US1 (T039, T040) — implement after US1 or create those files first.
- **US3 (P3)**: Foundational + US1 (entity, repository, service, controller files).
- **US4 (P4)**: Foundational + US1; its "absent from listing" test needs US2's list endpoint.

### Within Each User Story

- Tests are written first and must fail.
- Domain → repository → service → controller → module wiring.
- Story complete (checkpoint green) before moving to the next priority.

### Parallel Opportunities

- Phase 1: T003, T004, T005, T006.
- Phase 2: tests T008–T014; implementations T022, T023, T024, T025, T027, T028, T029.
- Each story: all test tasks marked [P] together; the pure domain task marked [P] alongside tests.
- Phase 7: T067, T068, T069, T070.

---

## Parallel Example: User Story 1

```bash
# Tests first, in parallel:
Task: "T034 Unit tests for nameplate rules in tests/unit/transformers/registration.test.ts"
Task: "T035 Contract tests in tests/contract/register-get.contract.test.ts"
Task: "T036 Integration tests in tests/integration/us1-register-view.test.ts"

# Then the pure domain rules (no shared files with the tests):
Task: "T037 Nameplate rules in src/transformers/domain/nameplate-rules.ts"
```

## Parallel Example: User Story 3

```bash
Task: "T050 Lifecycle unit tests in tests/unit/transformers/lifecycle.test.ts"
Task: "T051 Merge-patch unit tests in tests/unit/transformers/update.test.ts"
Task: "T052 Contract tests in tests/contract/update-status.contract.test.ts"
Task: "T053 Integration tests in tests/integration/us3-update-status.test.ts"
Task: "T054 Lifecycle state machine in src/transformers/domain/lifecycle.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 Setup → Phase 2 Foundational (checkpoint green).
2. Phase 3 US1 → validate with quickstart scenarios 1–4.
3. **Stop and review**: the registry can already register and read transformers.

### Incremental Delivery

1. Setup + Foundational → foundation ready.
2. US1 → MVP (register/view).
3. US2 → fleet search.
4. US3 → maintenance of data and lifecycle.
5. US4 → data-quality tools and history.
6. Polish → contract coverage, dev tooling, performance and security checks.

Each increment is merged through its own pull request from a branch off `main` (CLAUDE.md Git
workflow); commit after each task or logical group.

---

## Notes

- [P] tasks touch different files and have no dependency on incomplete tasks.
- Verify each test fails for the right reason before implementing.
- Any change to the API first updates `specs/001-transformer-registry/contracts/openapi.yaml`, then
  `openapi/openapi.yaml` and the generated types (`npm run openapi:check` enforces this).
