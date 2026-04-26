# Volu — Engineering Documentation

**Unbeatable. Exclusive. Now.**

This is the source-of-truth documentation for the Volu platform — an exclusive clearance marketplace for the UAE, built with Flutter (iOS + Android + merchant app) and NestJS (backend) with a Next.js admin console.

---

## Who this is for

- **Engineers** — architecture, APIs, data model, coding standards, deployment.
- **Product** — functional and non-functional requirements, user flows, use cases.
- **Design** — UI/UX principles, design system, accessibility.
- **Ops / SRE** — infrastructure, observability, incident response.
- **Security** — threat model, controls, compliance.
- **New joiners** — start with `01-overview/` and follow the breadcrumbs.

---

## Document Index

### 01 · Overview
| Doc | Purpose |
|---|---|
| [`01-overview/00-executive-summary.md`](./01-overview/00-executive-summary.md) | One-page pitch + strategic context |
| [`01-overview/01-glossary.md`](./01-overview/01-glossary.md) | Every domain term used across the platform |
| [`01-overview/02-personas.md`](./01-overview/02-personas.md) | User, merchant, admin personas with real detail |
| [`01-overview/03-success-metrics.md`](./01-overview/03-success-metrics.md) | KPIs, north-star metric, guardrails |

### 02 · Requirements
| Doc | Purpose |
|---|---|
| [`02-requirements/01-functional-requirements.md`](./02-requirements/01-functional-requirements.md) | Every functional requirement, IDs traceable to code |
| [`02-requirements/02-non-functional-requirements.md`](./02-requirements/02-non-functional-requirements.md) | Performance, availability, scalability, UX NFRs |
| [`02-requirements/03-use-cases.md`](./02-requirements/03-use-cases.md) | Full use case catalogue with pre/post-conditions |
| [`02-requirements/04-user-stories.md`](./02-requirements/04-user-stories.md) | Agile user stories mapped to epics |

### 03 · Architecture
| Doc | Purpose |
|---|---|
| [`03-architecture/01-system-architecture.md`](./03-architecture/01-system-architecture.md) | High-level architecture diagram and component responsibilities |
| [`03-architecture/02-tech-stack.md`](./03-architecture/02-tech-stack.md) | Every technology choice with rationale |
| [`03-architecture/03-backend-architecture.md`](./03-architecture/03-backend-architecture.md) | NestJS modular monolith design |
| [`03-architecture/04-mobile-architecture.md`](./03-architecture/04-mobile-architecture.md) | Flutter app architecture, state, navigation |
| [`03-architecture/05-admin-architecture.md`](./03-architecture/05-admin-architecture.md) | Next.js admin console design |
| [`03-architecture/06-event-driven-design.md`](./03-architecture/06-event-driven-design.md) | Domain events and their consumers |

### 04 · Data Model
| Doc | Purpose |
|---|---|
| [`04-data-model/01-erd-overview.md`](./04-data-model/01-erd-overview.md) | Full ERD with relationships |
| [`04-data-model/02-entities-detail.md`](./04-data-model/02-entities-detail.md) | Every table, every column, every index |
| [`04-data-model/03-migrations-strategy.md`](./04-data-model/03-migrations-strategy.md) | How we version and migrate the schema |

### 05 · API
| Doc | Purpose |
|---|---|
| [`05-api/01-api-design-principles.md`](./05-api/01-api-design-principles.md) | REST conventions, versioning, errors |
| [`05-api/02-authentication.md`](./05-api/02-authentication.md) | Auth flows, JWT, OTP, refresh |
| [`05-api/03-endpoints-catalog.md`](./05-api/03-endpoints-catalog.md) | Every endpoint with request/response |
| [`05-api/04-webhooks.md`](./05-api/04-webhooks.md) | Outbound webhooks and payload specs |

### 06 · Security
| Doc | Purpose |
|---|---|
| [`06-security/01-threat-model.md`](./06-security/01-threat-model.md) | STRIDE analysis |
| [`06-security/02-security-controls.md`](./06-security/02-security-controls.md) | Every control and how it's implemented |
| [`06-security/03-pci-pdpl-compliance.md`](./06-security/03-pci-pdpl-compliance.md) | Regulatory posture |
| [`06-security/04-incident-response.md`](./06-security/04-incident-response.md) | Runbook for security incidents |

### 07 · Infrastructure
| Doc | Purpose |
|---|---|
| [`07-infrastructure/01-cloud-topology.md`](./07-infrastructure/01-cloud-topology.md) | AWS me-central-1 topology |
| [`07-infrastructure/02-networking.md`](./07-infrastructure/02-networking.md) | VPC, subnets, security groups |
| [`07-infrastructure/03-databases.md`](./07-infrastructure/03-databases.md) | PostgreSQL, Redis, Meilisearch |
| [`07-infrastructure/04-ci-cd.md`](./07-infrastructure/04-ci-cd.md) | Pipelines and release process |
| [`07-infrastructure/05-observability.md`](./07-infrastructure/05-observability.md) | Logs, metrics, traces, alerts |
| [`07-infrastructure/06-dr-backup.md`](./07-infrastructure/06-dr-backup.md) | Disaster recovery and backup strategy |

### 08 · UI / UX
| Doc | Purpose |
|---|---|
| [`08-uiux/01-design-principles.md`](./08-uiux/01-design-principles.md) | The five principles every screen follows |
| [`08-uiux/02-design-system.md`](./08-uiux/02-design-system.md) | Tokens, components, spacing, motion |
| [`08-uiux/03-accessibility.md`](./08-uiux/03-accessibility.md) | WCAG 2.1 AA implementation |
| [`08-uiux/04-bilingual-rtl.md`](./08-uiux/04-bilingual-rtl.md) | EN + AR + RTL done right |
| [`08-uiux/05-critical-flows.md`](./08-uiux/05-critical-flows.md) | Key screen flows in detail |

### 09 · Development
| Doc | Purpose |
|---|---|
| [`09-development/01-monorepo-structure.md`](./09-development/01-monorepo-structure.md) | How the code is organised |
| [`09-development/02-coding-standards.md`](./09-development/02-coding-standards.md) | Linting, formatting, naming, patterns |
| [`09-development/03-testing-strategy.md`](./09-development/03-testing-strategy.md) | Unit, integration, e2e, the test pyramid |
| [`09-development/04-git-workflow.md`](./09-development/04-git-workflow.md) | Branching, commits, PR review |
| [`09-development/05-dev-environment.md`](./09-development/05-dev-environment.md) | How to spin up locally |

### 10 · Operations
| Doc | Purpose |
|---|---|
| [`10-operations/01-release-process.md`](./10-operations/01-release-process.md) | How we ship |
| [`10-operations/02-runbooks.md`](./10-operations/02-runbooks.md) | Common operational runbooks |
| [`10-operations/03-sla-slos.md`](./10-operations/03-sla-slos.md) | Service levels |
| [`10-operations/04-on-call.md`](./10-operations/04-on-call.md) | On-call rotations and escalation |

---

## How to navigate

If you're new, read in this order:

1. `01-overview/00-executive-summary.md` — what are we building
2. `01-overview/01-glossary.md` — the vocabulary
3. `03-architecture/01-system-architecture.md` — the big picture
4. Jump to your area of interest

---

## Conventions

- All requirements have stable IDs: `FR-XXX` (functional), `NFR-XXX` (non-functional), `UC-XXX` (use case), `US-XXX` (user story). These IDs must never be reused.
- Every significant design decision has an ADR (Architecture Decision Record) stored in `/docs/adr/`.
- Diagrams are Mermaid or PlantUML, committed as source alongside the docs.
- All amounts are AED unless otherwise specified.
- All dates are in UAE time (Asia/Dubai) unless otherwise specified.

---

## Document lifecycle

| Status | Meaning |
|---|---|
| `🟢 Approved` | Signed off by Product + Eng lead. Changes require ADR. |
| `🟡 Draft` | Under review. |
| `🔴 Deprecated` | No longer current; retained for history. |

Every doc has a status badge in its header.

---

## Contact

- Owner: Hasnat
- Tech lead: _[TBD]_
- Product lead: _[TBD]_
- Last updated: 2026-04-23
