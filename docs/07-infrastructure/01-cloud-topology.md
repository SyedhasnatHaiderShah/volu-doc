# 01 · Cloud Topology

**Status:** 🟢 Approved

The AWS deployment topology for Volu in `me-central-1` (Bahrain).

---

## Region & availability zones

**Primary region:** `me-central-1` (UAE-adjacent, Bahrain).
**Availability zones used:** `me-central-1a`, `me-central-1b`, `me-central-1c`.
**DR region:** `me-south-1` (Bahrain — secondary) — backup-only.

Volu data stays inside the Middle East region for PDPL data residency.

---

## High-level topology

```
                              ┌─────────────────────┐
                              │   Cloudflare Edge   │
                              │  (CDN, WAF, DDoS)   │
                              └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │ AWS Application LB  │
                              │   (multi-AZ)        │
                              └──────────┬──────────┘
                                         │
            ┌────────────────────────────┼────────────────────────────┐
            ▼                            ▼                            ▼
    ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
    │   AZ-1a         │          │   AZ-1b         │          │   AZ-1c         │
    │                 │          │                 │          │                 │
    │  ECS Tasks:     │          │  ECS Tasks:     │          │  ECS Tasks:     │
    │  • API Gateway  │          │  • API Gateway  │          │  • API Gateway  │
    │  • Volu Core    │          │  • Volu Core    │          │  • Volu Core    │
    │  • Workers      │          │  • Workers      │          │  • Workers      │
    │  • Admin BE     │          │  • Admin BE     │          │  • Admin BE     │
    │                 │          │                 │          │                 │
    │  RDS Postgres:  │          │  RDS Postgres:  │          │  RDS Postgres:  │
    │  Primary        │          │  Standby (sync) │          │  Read replica   │
    │                 │          │                 │          │                 │
    │  ElastiCache:   │          │  ElastiCache:   │          │  ElastiCache:   │
    │  Redis primary  │          │  Redis replica  │          │  (no Redis)     │
    │                 │          │                 │          │                 │
    │  Meilisearch:   │          │  Meilisearch:   │          │                 │
    │  ECS task       │          │  ECS task       │          │                 │
    └─────────────────┘          └─────────────────┘          └─────────────────┘

                                         │
                                         ▼
                              ┌─────────────────────┐
                              │  S3 / Cloudflare R2 │
                              │  (multi-AZ object)  │
                              └─────────────────────┘
```

---

## Compute — ECS Fargate

### Service catalogue

| Service | Purpose | Min instances | Max instances |
|---|---|---|---|
| `volu-api-gateway` | Public-facing API for mobile + admin | 2 (one per AZ) | 20 |
| `volu-core` | The modular monolith | 3 (one per AZ) | 30 |
| `volu-admin-backend` | Admin endpoints | 2 | 6 |
| `volu-workers-default` | Misc background jobs | 2 | 10 |
| `volu-workers-notifications` | Notification delivery | 2 | 20 |
| `volu-workers-search` | Meilisearch index updates | 2 | 6 |
| `volu-workers-outbox` | Outbox publisher | 2 | 4 |
| `volu-meilisearch` | Search engine | 2 (across AZs) | 2 |

**Task sizing baseline:**
- `api-gateway`: 0.5 vCPU, 1 GB RAM
- `core`: 1 vCPU, 2 GB RAM
- `workers`: 0.5–1 vCPU, 1–2 GB RAM (per worker type)

### Auto-scaling

- **CPU-based**: scale out at 70% sustained over 3 min; scale in at 30% over 10 min.
- **Custom metric** for workers: scale out on queue depth > 1000 jobs.
- **Predictive scaling**: scale up 30 min before known traffic peaks (Friday 9pm) once we have data.

### Deployment strategy

- **Rolling**: minimum 50% healthy capacity during deploy.
- **Blue/green** for risky releases (controlled by deploy flag).
- Health-check URL: `/ready`.

---

## Data tier

### PostgreSQL — RDS Multi-AZ

- **Engine:** PostgreSQL 16.
- **Instance:** start `db.r7g.large` (2 vCPU, 16 GB RAM); scale up as needed.
- **Storage:** 200 GB gp3 with auto-scaling enabled (max 2 TB).
- **Multi-AZ:** synchronous replica in AZ-1b for HA failover.
- **Read replicas:** 2 across AZ-1a and AZ-1c for analytics + admin reads.
- **Backups:**
  - Automated daily backup, 35-day retention.
  - Point-in-time recovery (PITR) enabled.
  - Manual snapshot before any major migration.
  - Weekly snapshot replicated to `me-south-1`.
- **Encryption:** AES-256 at rest with KMS-managed key.
- **Connection pooling:** PgBouncer in transaction-pooling mode.

### Redis — ElastiCache

- **Engine:** Redis 7.
- **Mode:** Cluster mode disabled, replication group with primary + 1 replica.
- **Instance:** start `cache.r7g.large`.
- **Multi-AZ:** automatic failover.
- **Backups:** daily snapshots, 7-day retention.
- **Encryption:** at rest + in transit.

### Meilisearch — self-hosted on ECS

- 2 tasks across AZs.
- Primary handles writes; replica serves reads.
- Index data on EBS-backed persistent volume per task.
- Nightly snapshots to S3 for recovery.
- Future: migrate to Algolia or fully managed search if ops burden grows.

---

## Storage — S3 / Cloudflare R2

Volu uses **Cloudflare R2** as the primary object store (S3-compatible API; zero egress fees).

| Bucket | Purpose | Access | Lifecycle |
|---|---|---|---|
| `volu-public-images` | Deal photos, merchant logos | Public via Cloudflare CDN; uploaded via signed URL | None (manually pruned) |
| `volu-kyc-documents` | KYC docs (trade licence, Emirates ID) | Private; signed URL TTL 5 min | Retain 7 years post-merchant offboarding |
| `volu-invoices` | Generated invoice + statement PDFs | Private; signed URL TTL 1 hour | Retain 7 years |
| `volu-wallet-passes` | Apple/Google Wallet pass files | Private; signed URL TTL 1 hour | 90 days |
| `volu-backups` | DB snapshots, Meilisearch snapshots | Private; admin-only | 12 months hot, then Glacier |
| `volu-data-exports` | User data exports | Private; signed URL TTL 72 hours | 30 days |
| `volu-logs-archive` | Cold-stored logs | Private; admin-only | 12 months |

All buckets:
- Encrypted at rest.
- Versioning enabled for non-ephemeral buckets.
- Access logged.

---

## CDN — Cloudflare

- **Domains routed**: `api.volu.ae`, `admin.volu.ae`, `volu.ae` (marketing), `cdn.volu.ae` (images).
- **Cache rules**:
  - Static assets: 1 year, immutable.
  - Deal images: 1 year (via Cloudflare Images for resize/format).
  - API responses: never cached at edge (origin headers prevent it).
- **WAF**: OWASP Top 10 ruleset; custom rules for Volu (e.g., block PHP requests, restrict admin paths to UAE IPs).
- **Bot management**: enabled on auth endpoints.
- **DDoS**: free with Cloudflare Pro.
- **Cloudflare Images**: on-the-fly transforms (resize, format conversion) for deal images.

---

## Networking — VPC layout

Detailed in [02-networking.md](./02-networking.md). Summary:

- **VPC**: `10.0.0.0/16`.
- **Public subnets** (`10.0.0.0/24`, `10.0.1.0/24`, `10.0.2.0/24`): ALB, NAT Gateways.
- **Private app subnets** (`10.0.10.0/24`, `10.0.11.0/24`, `10.0.12.0/24`): ECS Fargate tasks.
- **Private data subnets** (`10.0.20.0/24`, `10.0.21.0/24`, `10.0.22.0/24`): RDS, ElastiCache, Meilisearch.

Egress via NAT Gateway per AZ. PrivateLink for AWS service access (Secrets Manager, Parameter Store, KMS, ECR, S3 gateway endpoint).

---

## Logs & metrics — observability

- **Logs**: shipped to Grafana Loki (self-hosted on ECS) + retained 30 days hot, 12 months cold (S3).
- **Metrics**: Prometheus (scraped from ECS tasks); long-term storage in Grafana Mimir.
- **Traces**: OpenTelemetry → Grafana Tempo.
- **Errors**: Sentry (cloud).
- **Dashboards**: Grafana.

Detailed in [05-observability.md](./05-observability.md).

---

## Environments

| Environment | Region | Purpose |
|---|---|---|
| **Production** | me-central-1 | Live customer traffic |
| **Staging** | me-central-1 | Pre-production validation, integration tests |
| **Development** | n/a (local Docker) | Engineer machines |
| **Preview** | me-central-1 | Per-PR ephemeral environments |

Staging is a scaled-down mirror of production: same architecture, fewer instances, separate AWS account (cross-account isolation).

---

## Secrets management

- **AWS Secrets Manager** for all production secrets.
- **Parameter Store** for non-sensitive config.
- **No secrets in container images**; injected at task start time via task-role IAM permissions.
- **Rotation**: critical secrets rotated automatically every 90 days (DB password, JWT signing key); third-party API keys rotated manually with documented quarterly schedule.
- **Local dev**: `.env.local` files (gitignored); `.env.example` checked in with placeholders.

---

## DNS

| Hostname | Purpose | Provider |
|---|---|---|
| `volu.ae` | Marketing site | Cloudflare DNS |
| `api.volu.ae` | API endpoint | Cloudflare DNS → ALB |
| `admin.volu.ae` | Admin console | Cloudflare DNS → ALB |
| `cdn.volu.ae` | Image CDN | Cloudflare DNS → R2 + Images |
| `staging.volu.ae` | Staging frontend | Cloudflare DNS |
| `api-staging.volu.ae` | Staging API | Cloudflare DNS → staging ALB |

Apex domain handled via Cloudflare. TLS certs from Cloudflare or AWS ACM.

---

## Disaster recovery

See [06-dr-backup.md](./06-dr-backup.md). Summary:

- **RTO**: 1 hour for full restoration after a regional incident.
- **RPO**: 5 minutes data loss in worst case.
- **Cross-region snapshots** weekly to `me-south-1`.
- **DR drills** quarterly.

---

## Cost ballpark (steady state, year 1)

| Item | Estimate |
|---|---|
| Compute (ECS Fargate) | $1,500/mo |
| RDS (Multi-AZ + replicas) | $1,200/mo |
| ElastiCache | $400/mo |
| ALB + Data transfer | $300/mo |
| S3/R2 storage | $50/mo |
| Cloudflare Pro | $20/mo |
| Sentry | $80/mo |
| Grafana Cloud (or self-hosted equiv) | $200/mo |
| WhatsApp Business | $100/mo + per-message |
| Misc (KMS, Secrets, CW Logs short-term) | $200/mo |
| **Total infra** | **~$4,000/mo at 50K MAU** |

Excludes payment processing, SMS (variable per OTP), email, and ads spend. Scales roughly linearly with traffic.

---

## See also

- [Networking](./02-networking.md)
- [Databases](./03-databases.md)
- [CI/CD](./04-ci-cd.md)
- [Observability](./05-observability.md)
- [DR & Backup](./06-dr-backup.md)
- [System Architecture](../03-architecture/01-system-architecture.md)
