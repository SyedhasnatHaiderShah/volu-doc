# 06 · DR & Backup

**Status:** 🟢 Approved

How Volu survives data loss and regional incidents.

---

## Targets

- **RTO** (Recovery Time Objective): ≤ 1 hour for full service restoration after a regional incident.
- **RPO** (Recovery Point Objective): ≤ 5 minutes data loss in worst-case regional failure.

---

## What we protect

| Data                      | Backup approach                                                             | Where               |
| ------------------------- | --------------------------------------------------------------------------- | ------------------- |
| PostgreSQL                | RDS automated daily snapshot + WAL streaming + weekly cross-region snapshot | RDS, S3, me-south-1 |
| PostgreSQL logical backup | Weekly `pg_dump` to S3                                                      | S3                  |
| Redis                     | Daily RDB snapshot (7-day retention)                                        | ElastiCache + S3    |
| Meilisearch               | Nightly snapshot to S3 + can be rebuilt from PostgreSQL                     | S3                  |
| KYC documents             | S3 with versioning + cross-region replication                               | R2/S3               |
| Invoices                  | S3 with versioning + cross-region replication                               | R2/S3               |
| Container images          | ECR with cross-region replication                                           | ECR                 |
| Source code               | GitHub                                                                      | GitHub              |
| Configuration (Terraform) | Git + state in S3 (versioned)                                               | GitHub + S3         |
| Secrets                   | Secrets Manager (versioned)                                                 | AWS                 |

---

## RDS — primary protection

### Multi-AZ failover

- Synchronous replica in AZ-1b (different from primary AZ-1a).
- Automatic failover on primary failure: typically 30–90 seconds.
- App reconnects via DNS (CNAME swung by RDS).
- Worst case lost data: pending in-flight transactions.

### Automated daily snapshots

- 35-day retention in same region.
- Snapshots encrypted with the same KMS key as the database.

### Continuous WAL streaming (PITR)

- Point-in-time recovery to any second within retention.
- Used for "human error" recovery (e.g., bad migration, accidental delete).

### Cross-region snapshots

- Weekly snapshot copied to `me-south-1`.
- Used for full regional disaster.

### Logical backups

- Weekly `pg_dump --schema-only` and `pg_dump --data-only` (custom format) uploaded to S3.
- Defence against logical corruption that could replicate to standbys.

---

## Redis — recovery

Redis is mostly cache + session/queue state. Loss tolerable for:

- Caches: rebuilt from PostgreSQL.
- Sessions: users re-login.
- Rate-limit counters: reset (small disruption).

What we protect:

- BullMQ queue state: jobs in flight could be lost. Mitigation: outbox pattern (durable in PostgreSQL) ensures we can re-enqueue.
- Session revocation list: rebuildable from `identity__sessions`.

Daily snapshots used for "I'd rather not rebuild from scratch" scenarios.

---

## Meilisearch — recovery

Meilisearch is fully derivable from PostgreSQL. In recovery:

1. Restore from latest nightly snapshot (fast: ~minutes).
2. OR, full reindex from PostgreSQL (~20–40 min for 10K deals + 600 merchants).

---

## Object storage (R2/S3)

- Versioning enabled on all non-ephemeral buckets.
- Cross-region replication for KYC and invoice buckets.
- Lifecycle policies move old objects to cheaper tiers.

---

## Disaster scenarios

### Scenario 1: AZ failure

- **What:** Single AZ goes down.
- **Impact:** Minimal. Multi-AZ services failover automatically.
- **Recovery:** Automatic; verify all services healthy on remaining AZs.
- **RTO:** 5 minutes.
- **RPO:** 0 (synchronous replication).

### Scenario 2: Region failure

- **What:** All of `me-central-1` unavailable.
- **Impact:** Full outage.
- **Recovery:**
  1. Activate `me-south-1` standby region (warm standby — Terraform can spin up infra in 30 min).
  2. Restore latest cross-region RDS snapshot.
  3. Rebuild Redis (cold — empty OK).
  4. Rebuild Meilisearch from PostgreSQL.
  5. Update DNS (Cloudflare) to point to new region.
  6. Verify and announce.
- **RTO:** ~1 hour.
- **RPO:** ~7 days (last weekly snapshot) or up to 24h if continuous cross-region WAL is enabled later.

We'll add continuous cross-region WAL once monthly cost is justified by user volume.

### Scenario 3: Logical data corruption

- **What:** A bad migration or bug corrupts data.
- **Recovery:**
  1. Identify scope and time of corruption.
  2. PITR to a sandbox cluster at point just before corruption.
  3. Compare data; surgically restore corrupted rows.
  4. Apply fix; deploy.
- **RTO:** Variable; usually 1–4 hours depending on scope.
- **RPO:** 0 if PITR captures the right point.

### Scenario 4: Ransomware / malicious insider

- **What:** Someone deletes critical data.
- **Recovery:**
  1. Engage incident response (SEV-1).
  2. PITR to before the deletion.
  3. Investigate via audit log; identify actor; revoke access.
  4. Forensic analysis; potential law enforcement.

### Scenario 5: Backup itself corrupted

- **What:** Latest snapshot fails to restore.
- **Recovery:**
  1. Try previous snapshot.
  2. Try logical pg_dump backup.
  3. Try cross-region snapshot.
- **Mitigation:** Monthly automated restore-test drill validates backup integrity.

---

## DR runbook

```
Step 1 — Detect & confirm
├─ Synthetic monitor flags region down
├─ Engineer confirms via AWS console (region health dashboard)
├─ Declare SEV-1
└─ IC + CL assigned

Step 2 — Communicate
├─ Status page: "Investigating regional outage"
├─ Internal: #incident-active opened
└─ External: customer push (if ETA > 30 min)

Step 3 — Activate DR region
├─ Run: terraform workspace select dr && terraform apply
├─ Restore latest RDS cross-region snapshot
├─ Provision Redis (fresh)
├─ Restore Meilisearch from S3 snapshot
├─ Deploy latest container images
├─ Apply Terraform-managed secrets
└─ Verify health endpoints

Step 4 — Cutover
├─ Update Cloudflare DNS for api.volu.ae → DR ALB
├─ Confirm propagation (~2 min via Cloudflare)
└─ Run synthetic full-flow check

Step 5 — Communicate restoration
├─ Status page: "Resolved"
└─ Push to users: "We're back"

Step 6 — Post-mortem
└─ Within 5 business days
```

---

## DR drills

Quarterly: simulate a full regional failure in a sandbox; measure RTO/RPO; identify gaps. Findings drive runbook improvements.

Monthly: backup-restore drill — restore latest snapshot to a sandbox cluster; verify integrity.

---

## Backup integrity checks

- **Daily**: automated query against latest snapshot tests row counts on critical tables match production within tolerance.
- **Weekly**: full restore to a sandbox; smoke tests pass; sandbox torn down.
- **Quarterly**: full DR drill including DNS cutover (in test domain).

---

## Documentation requirements

For DR to actually work, we maintain:

1. **Terraform state** for both regions, version-controlled.
2. **Secrets inventory** with rotation schedule.
3. **Runbook** above, kept current.
4. **Contact list**: AWS support tier, Cloudflare support, key vendors.
5. **Vendor SLAs**: known to IC during incident.

---

## Cost of DR

- Cross-region snapshots: ~$50/month at 200GB.
- DR region warm-standby (cold infrastructure but pre-built Terraform): ~$0/month.
- Active warm region (later, if RTO needs to drop to 15 min): ~$1,500/month.

We start with cold DR (1h RTO is fine for our market; full active-active is overkill).

---

## See also

- [Cloud Topology](./01-cloud-topology.md)
- [Databases](./03-databases.md)
- [Incident Response](../06-security/04-incident-response.md)
- [Runbooks](../10-operations/02-runbooks.md)
