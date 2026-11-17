# Asset Data Service Constitution

Asset Data Service fetches and manages data about **high-voltage (HV) power transformers and
their related parts** (e.g. bushings, on-load tap changers, cooling equipment, protection
devices, insulating oil). This constitution defines the non-negotiable rules for how the
service is specified, built, and evolved.

## Core Principles

### I. Spec-First Delivery

- Every feature MUST start from a specification in `specs/<NNN-feature-name>/spec.md`
  created with `/speckit-specify`, describing the *what* and *why* without implementation detail.
- No production code MAY be written before the feature has an approved `spec.md`, `plan.md`
  and `tasks.md` (`/speckit-specify` → `/speckit-plan` → `/speckit-tasks` → `/speckit-implement`).
- Ambiguities MUST be resolved (via `/speckit-clarify` or marked `[NEEDS CLARIFICATION]`)
  before planning starts.
- When code and spec diverge, the spec is updated first; the spec is the source of truth.

Rationale: asset data feeds engineering and maintenance decisions on critical grid equipment;
requirements must be explicit and reviewable before they become code.

### II. Domain Model Integrity

- Every asset MUST have a stable, system-generated identifier plus its natural identifiers
  (manufacturer, model, serial number); the combination manufacturer + serial number MUST be unique.
- The asset hierarchy (transformer → components/parts) MUST preserve referential integrity:
  a part cannot reference a non-existent parent, and relationships are explicit, never implied.
- Physical quantities MUST carry explicit units (e.g. kV, MVA, A, °C, kg) and be validated
  against plausible ranges; unitless numeric fields for physical values are forbidden.
- Nameplate and rating terminology SHOULD follow IEC 60076 / IEEE C57 vocabulary so data is
  unambiguous to power engineers.
- All input MUST be validated at the system boundary with a schema; invalid data is rejected,
  never silently coerced.

Rationale: incorrect or ambiguous equipment data (wrong voltage class, unit, or parent) can lead
to unsafe engineering decisions.

### III. Contract-First API

- Every externally exposed endpoint MUST be described in an OpenAPI 3.1 contract under
  `specs/<feature>/contracts/` before it is implemented.
- The public API MUST be versioned (e.g. `/v1/...`); breaking changes require a new major
  API version and a documented migration path.
- Errors MUST use a consistent machine-readable format (RFC 9457 Problem Details).
- Collection endpoints MUST be paginated and support filtering on documented fields only.

Rationale: other systems (maintenance, monitoring, planning) will integrate with this service;
stable contracts protect them.

### IV. Test-First (NON-NEGOTIABLE)

- Tests MUST be written and observed failing before the implementation (Red → Green → Refactor).
- Required test layers:
  - **Unit tests** for domain rules and validation.
  - **Contract tests** verifying every endpoint against its OpenAPI contract.
  - **Integration tests** against a real database instance (no in-memory fakes for persistence).
- A feature is not done while any test is failing or skipped without a linked, justified issue.

Rationale: tests are the executable form of the spec and the guardrail for refactoring.

### V. Traceability & Auditability

- Every create/update/delete on asset data MUST be recorded in an audit trail
  (who, when, what changed, previous and new values).
- Asset records MUST NOT be hard-deleted through the API; use status/lifecycle transitions
  (e.g. in service, out of service, decommissioned) or soft deletion.
- Data imported from external sources MUST record its provenance (source system, import time).

Rationale: asset histories are needed for maintenance decisions, failure analysis and
regulatory or insurance inquiries.

### VI. Simplicity & Observability

- Start with the simplest design that satisfies the spec (YAGNI); every added layer, service
  or dependency MUST be justified in the plan's Complexity Tracking table.
- The service MUST emit structured JSON logs with request/correlation IDs and MUST NOT log
  secrets or credentials.
- The service MUST expose liveness and readiness health endpoints.

Rationale: a small, observable service is easier to operate and evolve than a speculative one.

## Technology & Data Constraints

- **Runtime**: Node.js active LTS (currently 24.x) with **TypeScript** in `strict` mode;
  `any` is forbidden except at justified, documented boundaries.
- **Module system**: ES modules by default. Applications built on NestJS MAY compile to
  CommonJS, because NestJS dependency injection relies on decorator metadata that is unreliable
  under native ESM; ESM-only dependencies are then consumed through Node's `require(esm)`.
  This exception needs no per-feature justification.
- **Tooling**: ESLint + Prettier enforced; a lockfile MUST be committed; dependencies are pinned
  and reviewed before adoption.
- **Framework, persistence and libraries** (HTTP framework, database, ORM/query builder,
  validation library) are selected per feature in `/speckit-plan` and recorded in `plan.md`
  with rationale; once chosen they become project defaults unless a plan justifies a change.
- **Configuration** comes from environment variables validated at startup; secrets MUST never
  be committed to the repository.
- **Security**: all non-health endpoints MUST require authentication; authorization rules are
  defined in the spec of each feature.

## Development Workflow & Quality Gates

- Each feature lives on its own branch named after its spec directory (`NNN-feature-name`),
  created by `/speckit-specify`.
- Before merge, every pull request MUST pass: type check, lint, formatting, all test layers,
  and coverage ≥ 80% on domain/business logic.
- `/speckit-analyze` SHOULD be run after `/speckit-tasks` to check spec/plan/tasks consistency
  before implementation.
- Reviews MUST verify compliance with this constitution; the plan's *Constitution Check* gate
  MUST pass (or violations be justified) before implementation starts.
- Commits follow Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`, `chore:`).

## Governance

- This constitution supersedes all other development practices for this repository.
- Amendments are made through a pull request that updates this file via `/speckit-constitution`,
  includes the Sync Impact Report, and is approved by a maintainer.
- Versioning follows semantic versioning: MAJOR for removed or redefined principles, MINOR for
  new principles or materially expanded guidance, PATCH for clarifications and wording.
- Any justified deviation MUST be recorded in the affected `plan.md` Complexity Tracking table.
- Runtime guidance for AI agents lives in `CLAUDE.md` and MUST stay consistent with this file.

**Version**: 1.1.0 | **Ratified**: 2026-11-14 | **Last Amended**: 2026-11-17
