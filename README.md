# volu-docs

The technical specification, architecture, and operational handbook for **Volu** — an AI-assisted coupon marketplace SaaS for the UAE.

The full documentation lives in [`/docs`](./docs).

Start here: [`docs/README.md`](./docs/README.md).

## What's in here

A complete engineering-grade specification covering:

- Product overview, personas, success metrics
- Functional and non-functional requirements
- System, backend, mobile, and admin architecture
- Full data model with entity-level schema
- API design, authentication, and endpoint catalogue
- Security: threat model, controls, PCI/PDPL compliance, incident response
- Infrastructure: AWS topology, networking, databases, CI/CD, observability, DR
- UI/UX: design principles, design system, accessibility, bilingual/RTL, critical flows
- Development: monorepo structure, coding standards, testing, git workflow, dev setup
- Operations: release process, runbooks, SLAs, on-call

## Status

🟢 Approved — v1.0 (April 2026)

## Brand

**Volu** — Unbeatable. Exclusive. Now.

## Conventions

- Docs use stable IDs that never get reused: `FR-XXX`, `NFR-XXX`, `UC-XXX`, `US-XXX`, `T-XXX`, `SC-XXX`, `RB-XXX`.
- Money in fils (1 AED = 100 fils), always integer, never floats.
- Timestamps in UTC (TIMESTAMPTZ); rendered in `Asia/Dubai`.
- IDs are UUID v7.
- Bilingual EN + AR throughout, RTL parity required.

## Maintenance

- Each doc carries a status badge.
- Architectural decisions captured in `/adr` (Architecture Decision Records).
- Updates flow through PRs with the same review rules as code.
