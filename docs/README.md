# crewkit Documentation Index

> The canonical map of all documentation. Start here.
> For AI agents: [`CLAUDE.md`](../CLAUDE.md) (architecture + conventions) and
> [`AGENTS.md`](../AGENTS.md) (testing discipline) are the entry points; component rules
> auto-load from [`.claude/rules/`](../.claude/rules/).

## Start here

| Doc | What it answers |
|---|---|
| [`../README.md`](../README.md) | What is crewkit; quickstart |
| [`product-direction.md`](./product-direction.md) | Strategy: the 2026-04 pivot, four pillars, frozen decisions |
| [`../PLAN.md`](../PLAN.md) | Execution phases A–E and their status |
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | How to contribute |

## Architecture & operations

| Doc | Scope |
|---|---|
| [`../CLAUDE.md`](../CLAUDE.md) | Monorepo architecture, domain model, endpoints, services |
| [`../AGENTS.md`](../AGENTS.md) | Test commands per component — what to run, what NOT to run |
| [`../.claude/rules/`](../.claude/rules/) | Component conventions: api, cli, dashboard, testing, general (auto-loaded by path) |
| [`DEPLOYMENT.md`](./DEPLOYMENT.md) | DigitalOcean App Platform deployment |
| [`SECURITY_MODEL.md`](./SECURITY_MODEL.md) | AuthN/AuthZ, multi-tenancy, JWT scopes |
| [`SENTRY.md`](./SENTRY.md) | Error tracking setup |
| [`api/`](./api/) | API-specific docs |
| [`patterns/`](./patterns/) | Reusable implementation patterns |

## Product & planning

| Doc | Scope |
|---|---|
| [`product-strategy-playbook-enforcement.md`](./product-strategy-playbook-enforcement.md) | Playbook enforcement strategy |
| [`proposal-curated-context-ingestion.md`](./proposal-curated-context-ingestion.md) | Context ingestion proposal |
| [`prd-notification-system.md`](./prd-notification-system.md) | Notification system PRD |
| [`planning/`](./planning/) | Active planning docs |

## Audits

| Doc | Scope |
|---|---|
| [`audits/deep-assessment-2026-06-11.md`](./audits/deep-assessment-2026-06-11.md) | Full-platform assessment: verified findings, hooks-vs-gateway, LSP, KB, sharing DX, TUI/web vision, prioritized roadmap |
| [`audits/`](./audits/) | Earlier audits |

## Archive

[`archive/`](./archive/) holds superseded plans and completed migration docs — historical
context only; nothing in it is current. When a planning doc is finished or contradicted,
move it there rather than deleting.

## Maintenance rules

- One fact, one home: architecture lives in `CLAUDE.md`; this index links, never duplicates.
- Update this index when adding or archiving a doc (CI guard: planned, see audit §11).
- The orchestrator component is currently undocumented — first doc to write (audit §11).
