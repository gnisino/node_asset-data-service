# Implementation Plan: HV Transformer Registry

**Branch**: `001-transformer-registry` | **Date**: 2026-11-15 | **Spec**: [spec.md](./spec.md)

**Status**: Approved (project owner, 2026-11-18), after `/speckit-analyze` remediation

**Input**: Feature specification from `/specs/001-transformer-registry/spec.md`

## Summary

Build the first slice of the Asset Data Service: a versioned REST API (`/v1`) to register,
retrieve, list/filter, update, change the lifecycle status of, remove (soft) and restore
high-voltage power transformers, with a complete immutable audit trail and role-based access
(viewer/editor/administrator). The service is a NestJS application on Node.js 24 + TypeScript,
backed by PostgreSQL 18 through Drizzle ORM. The OpenAPI 3.1 contract is written first and is the
source of truth for request validation (Ajv), generated TypeScript types and contract tests;
domain invariants are additionally enforced by database constraints and triggers.

## Technical Context

**Language/Version**: TypeScript 5.x (`strict`), Node.js 24 LTS, compiled to CommonJS (see R1)

**Primary Dependencies**: NestJS 11 (Express adapter), drizzle-orm + drizzle-kit, pg, ajv +
ajv-formats, openapi-typescript, jose, nestjs-pino, @nestjs/terminus, @nestjs/config + zod

**Storage**: PostgreSQL 18 (UUIDv7 keys, partial unique index, CHECK constraints, triggers)

**Testing**: Vitest + unplugin-swc, supertest, @testcontainers/postgresql, @vitest/coverage-v8

**Target Platform**: Linux container (OCI image), Node.js 24 runtime

**Project Type**: web-service (single backend project, no frontend)

**Performance Goals**: p95 < 1 s for lookup and filtered list pages with 10,000 transformers
(SC-003); expected load: tens of concurrent users

**Constraints**: no hard deletes; every mutation audited in the same transaction; optimistic
concurrency via `ETag`/`If-Match`; all non-health endpoints authenticated; structured JSON logs
without secrets

**Scale/Scope**: ~10k transformers, ~100k audit entries in the first years; 8 HTTP operations
(list, register, get, update, status change, removal, restoration, history) plus 2 health
probes; 2 tables

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle / Rule | Status | Evidence |
|---|---|---|
| I. Spec-First Delivery | ✅ Pass | Approved [spec.md](./spec.md) with clarifications; no code before `tasks.md` |
| II. Domain Model Integrity | ✅ Pass | System UUID + unique manufacturer/serial (partial index); unit-suffixed fields; ranges + IEC notations validated at boundary and in DB ([data-model.md](./data-model.md)) |
| II. Referential integrity of hierarchy | ✅ N/A | Components are out of scope; `audit_entries` FK with `ON DELETE RESTRICT` |
| III. Contract-First API | ✅ Pass | [contracts/openapi.yaml](./contracts/openapi.yaml) written before code; `/v1`; RFC 9457; pagination + documented filters only |
| IV. Test-First | ✅ Pass | Unit, contract and integration (real PostgreSQL via Testcontainers) layers; tests written first per `tasks.md` |
| V. Traceability & Auditability | ✅ Pass | Append-only `audit_entries` (who/when/previous/new), soft removal, app role without DELETE/TRUNCATE privileges plus delete/update/truncate-blocking triggers. Provenance: N/A (no imports in this feature) |
| VI. Simplicity & Observability | ✅ Pass | One project, two tables, no separate status table; pino JSON logs with request id; `/health/live`, `/health/ready` |
| Tech: Node LTS + TS strict, no `any` | ✅ Pass | Node 24, `strict`, ESLint `no-explicit-any` |
| Tech: module system | ✅ Pass | CommonJS output for a NestJS application, allowed by constitution v1.1.0 (*Module system*) |
| Tech: env config validated, no secrets in repo | ✅ Pass | `@nestjs/config` + Zod schema; `.env.example` only |
| Tech: authentication on non-health endpoints | ✅ Pass | Global JWT guard, `@Public()` only on health |
| Quality gates (typecheck, lint, tests, 80% domain/business-logic coverage) | ✅ Pass | npm scripts + coverage threshold on `src/transformers/domain`, `transformers.service.ts` and `src/audit` ([quickstart.md](./quickstart.md)) |

**Post-design re-check (after Phase 1 and `/speckit-analyze` remediation, 2026-11-17)**: all
gates pass. The CommonJS choice is covered by constitution v1.1.0; the custom validation pipe is
recorded in Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/001-transformer-registry/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R1–R14
├── data-model.md        # Phase 1: entities, constraints, state machines
├── quickstart.md        # Phase 1: validation guide
├── contracts/
│   └── openapi.yaml     # Phase 1: OpenAPI 3.1 contract
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 (/speckit-tasks — not created here)
```

### Source Code (repository root)

```text
openapi/
└── openapi.yaml                       # Service contract loaded at runtime (from contracts/)

src/
├── main.ts                            # Bootstrap: JSON + merge-patch parser, shutdown hooks
├── app.module.ts                      # Global guards (APP_GUARD) and filter (APP_FILTER)
├── config/
│   └── config.schema.ts               # Zod schema for environment variables
├── logging/
│   └── logger.config.ts               # nestjs-pino config, header redaction
├── common/
│   ├── problem-details/
│   │   └── problem-details.filter.ts  # RFC 9457 exception filter + error mapping
│   ├── validation/
│   │   ├── contract-validator.ts      # Ajv 2020-12 over openapi/openapi.yaml
│   │   └── validate.pipes.ts          # @ValidateBody / @ValidateQuery
│   ├── http/
│   │   ├── etag.ts                    # ETag / If-Match helpers and errors
│   │   └── request-id.middleware.ts   # X-Request-Id
│   └── pagination.ts
├── auth/
│   ├── jwks.provider.ts               # JWKS_RESOLVER (remote IdP or local dev keys)
│   ├── auth.guard.ts                  # JWT verification via jose
│   ├── roles.guard.ts                 # @Roles() + cumulative RolesGuard
│   ├── public.decorator.ts
│   ├── current-actor.decorator.ts
│   └── actor.ts                       # Actor value object from token claims
├── database/
│   ├── database.module.ts             # pg Pool + Drizzle instance, withTransaction
│   └── schema.ts                      # Drizzle tables: transformers, audit_entries
├── transformers/
│   ├── transformers.module.ts
│   ├── domain/                        # Pure TS, no Nest/Drizzle/pg imports
│   │   ├── transformer.ts             # Entity, registration, merge-patch update
│   │   ├── nameplate-rules.ts         # Cross-field rules, normalization
│   │   ├── lifecycle.ts               # Status state machine
│   │   ├── removal.ts                 # Remove / restore rules
│   │   ├── diff.ts                    # Field-level change computation for audit
│   │   └── errors.ts                  # Domain errors
│   ├── transformer.mapper.ts          # Snapshot → API response
│   ├── transformer-filters.ts         # List filters + query validation
│   ├── transformers.service.ts        # Use cases; one DB transaction per mutation incl. audit
│   ├── transformers.repository.ts     # Drizzle queries, filters, numeric mapping
│   └── transformers.controller.ts     # /v1/transformers routes
├── audit/
│   ├── audit.module.ts
│   ├── audit.repository.ts            # Append + paginated history
│   └── history.controller.ts          # /v1/transformers/{id}/history
├── health/
│   ├── health.module.ts
│   └── health.controller.ts
└── generated/
    └── openapi.d.ts                   # openapi-typescript output (generated, committed)

drizzle/
├── 0000_initial.sql                   # Generated from schema.ts
└── 0001_guards.sql                    # Custom: unique partial index, triggers, grants
drizzle.config.ts

tests/
├── unit/                              # config, validation, http, common, auth, transformers
├── contract/                          # each operation × status code vs openapi/openapi.yaml
├── integration/                       # user-story flows, DB guards, security
└── support/
    ├── global-setup.ts                # Testcontainers postgres:18, roles, migrations
    ├── database.ts                    # resetDatabase()
    ├── tokens.ts                      # local key pair + JWKS, tokenFor(role)
    ├── app.ts                         # createTestApp()
    ├── contract.ts                    # expectMatchesContract()
    ├── fixtures.ts                    # validTransformer()
    └── seed.ts                        # seedTransformers()

scripts/
├── migrate.ts                         # Applies drizzle/ as schema owner
├── openapi-check.ts                   # Contract copy + generated types in sync
├── dev-token.ts                       # Local dev token issuer (AUTH_DEV_KEYS=true only)
├── perf-seed.ts                       # Seeds N transformers for SC-003
└── perf-check.ts                      # Measures p95 of lookups and filtered list pages

docker/postgres/init/01-app-role.sql   # Creates the asset_data_app role locally
docker-compose.yml                     # postgres:18 for local development
.env.example
package.json, package-lock.json, tsconfig.json, tsconfig.build.json, nest-cli.json,
vitest.config.ts, eslint.config.mjs, .prettierrc, .prettierignore
```

**Structure Decision**: single backend project at the repository root. Each Nest feature module
keeps a framework-free `domain/` folder holding the business rules (unit-tested, coverage gate),
with thin service/repository/controller layers around it. No generic repository abstraction or
CQRS: the service calls the Drizzle repository directly inside a transaction.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Custom Ajv validation pipe instead of an off-the-shelf validator | Contract-first validation must report schema **and** domain errors together in one RFC 9457 response (FR-008) | `express-openapi-validator` returns schema errors in its own format before domain rules run; `class-validator` DTOs would duplicate the contract (code-first) |
