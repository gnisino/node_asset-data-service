# node_asset-data-service

Service which allows to fetch and manage data about assets: high-voltage power transformers
and their related parts.

## Development methodology

This project follows **Spec-Driven Development** with [GitHub Spec Kit](https://github.com/github/spec-kit).

- Project principles: [`.specify/memory/constitution.md`](.specify/memory/constitution.md)
- Feature specifications, plans and tasks: `specs/<NNN-feature-name>/`
- Agent guidance: [`CLAUDE.md`](CLAUDE.md)

Typical feature flow (as Claude Code skills):

```text
/speckit-specify → /speckit-clarify → /speckit-plan → /speckit-tasks → /speckit-analyze → /speckit-implement
```

### Setup of the Spec Kit CLI (optional, only to re-init or upgrade)

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

## Tech stack

Node.js (LTS) + TypeScript. Framework and persistence choices are recorded per feature in
`specs/*/plan.md`.
