# 03 · Databases

**Status:** 🟢 Approved

Operational guide for the data tier — PostgreSQL, Redis, Meilisearch.

---

## PostgreSQL (RDS)

### Configuration

- **Engine:** PostgreSQL 16.
- **Instance:** `db.r7g.large` initially (2 vCPU, 16 GB RAM).
- **Storage:** 200 GB gp3, IOPS 6000, throughput 250 MB/s; auto-scaling up to 2 TB.
- **Multi-AZ:** Yes.
- **Read replicas:** 2 cross-AZ replicas.
- **Backup retention:** 35 days.
- **Maintenance window:** Friday 04:00–05:00 UAE (lowest traffic).

### Parameter group highlights

```
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 32MB
maintenance_work_mem = 1GB
max_connections = 200
random_page_cost = 1.1
log_min_duration_statement = 1000   # log queries > 1s
log_lock_waits = on
default_statistics_target = 200
autovacuum = on
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
```

### Connection pooling

PgBouncer in **transaction-pooling mode** sits between app and RDS:

- Max client connections: 1000
- Max DB connections: 200
- Pool mode: transaction (allows safe pooling for non-prepared-statement queries)

App connects to PgBouncer; PgBouncer multiplexes to RDS. This lets us scale app instances horizontally without exhausting DB connections.

### Read-write splitting

App reads (analytics, admin lists, dashboard queries) routed to read replicas. App writes always to primary.

Pattern:

```ts
// In repositories:
this.prismaPrimary.user.findMany(...)   // write path
this.prismaReplica.user.findMany(...)   // read-only, lower priority
```

Replica lag monitored; if > 30s, alerts trigger and reads fail over to primary temporarily.

### Backups

- **Automated:** RDS daily snapshot, 35-day retention.
- **Continuous:** WAL streaming for point-in-time recovery within retention.
- **Cross-region:** Weekly snapshot replicated to `me-south-1`.
- **Logical:** Weekly `pg_dump` to S3 (logical, schema + data) — extra layer for restore flexibility.
- **Test:** Monthly automated restore test to a sandbox cluster validates backup integrity.

### Vacuum & maintenance

- Autovacuum tuned aggressively for write-heavy tables (`coupons`, `orders`, `redemptions`).
- Manual `VACUUM ANALYZE` on partitioned tables before quarterly partition rollover.
- Index bloat monitored monthly; rebuild via `REINDEX CONCURRENTLY` when needed.

### Monitoring

CloudWatch + Grafana:

- CPU utilisation
- DB connections
- Read/write IOPS, throughput
- Replication lag
- Long-running queries (> 5s)
- Lock waits
- Buffer cache hit ratio (target > 99%)

### Security

- Security group restricts to `sg-app-tasks`.
- TLS-only connections (`require_secure_transport=ON`).
- Master credentials in Secrets Manager; rotated 90 days via Lambda rotation.
- IAM authentication available for human ops (preferred over passwords).
- `pg_audit` extension enabled for DDL and security-relevant events.

### Partitioning strategy

High-growth tables partitioned by date when row count reaches ~10M:

| Table             | Strategy                  | Trigger        |
| ----------------- | ------------------------- | -------------- |
| `notifications`   | Range, weekly             | From day 1     |
| `audit_logs`      | Range, monthly            | From day 1     |
| `outbox_events`   | Range, daily (purged 30d) | From day 1     |
| `orders`          | Range, quarterly          | After 6 months |
| `coupons`         | Range, quarterly          | After 6 months |
| `redemptions`     | Range, quarterly          | After 6 months |
| `merchant_ledger` | Range, quarterly          | After 6 months |

Partition creation automated by a scheduled job (3 months ahead).

---

## Redis (ElastiCache)

### Configuration

- **Engine:** Redis 7.
- **Mode:** Cluster mode disabled (single shard); replication group with primary + 1 replica.
- **Instance:** `cache.r7g.large` (13 GB).
- **Multi-AZ:** Yes.
- **Failover:** Automatic.
- **Snapshots:** Daily, 7-day retention.
- **Encryption:** at-rest + in-transit.

### Use cases & key patterns

| Pattern        | Key prefix                     | TTL           | Use                                                     |
| -------------- | ------------------------------ | ------------- | ------------------------------------------------------- |
| Cache          | `cache:deal:{id}`              | 5 min         | Hot deal reads                                          |
| Cache          | `cache:merchant:{id}`          | 10 min        | Merchant profiles                                       |
| Session        | `session:revoked:{jti}`        | 15 min        | Revoked JWT IDs (until natural expiry)                  |
| Rate limit     | `rl:{endpoint}:{key}:{window}` | window length | Sliding-window rate limit counters                      |
| Idempotency    | `idem:{key}`                   | 24 h          | Cached idempotent responses                             |
| Feature flags  | `ff:flags` (single hash)       | 60s           | Cached flag state                                       |
| Locks          | `lock:{resource}:{id}`         | 30s default   | Short distributed locks (Redlock or single-node SET NX) |
| Queue (BullMQ) | `bull:{queueName}:*`           | n/a           | Job queues                                              |
| Pub/Sub        | channel names                  | n/a           | Real-time signals                                       |

### Memory pressure

`maxmemory-policy = volatile-lru`. Caches with TTL evicted; sessions/queues without TTL not evicted.

If we hit memory pressure: scale instance up; consider sharding when single shard insufficient.

### Monitoring

- Memory used vs max
- Cache hit ratio (target > 90% for cache: prefix)
- Evictions
- Client connections
- Replication lag
- Failover events

---

## Meilisearch

### Configuration

- 2 ECS tasks across AZs.
- Each backed by its own EBS gp3 volume (50 GB to start).
- Master key in Secrets Manager.

### Index strategy

| Index       | Documents                            | Size estimate |
| ----------- | ------------------------------------ | ------------- |
| `deals`     | All approved & live + recent expired | ~10K docs     |
| `merchants` | All approved                         | ~600 docs     |

Per-deal document includes:

- id, title (en+ar), description (en+ar), category, tags
- merchant name (en+ar), brand
- city, branches list
- price, deal price, discount %
- valid_until
- popularity score (denormalised)
- geo coordinates (per branch)

### Indexing pipeline

`deal.approved` event → `volu-workers-search` worker → upsert document into Meilisearch.

`deal.expired` / `deal.paused` → remove or downrank.

Full reindex: weekly scheduled job rebuilds from PostgreSQL as source of truth (catches any drift).

### Settings

- **Searchable attributes:** title, description, merchant name, tags, category.
- **Filterable attributes:** category, city, price (ranged), discount %, tags, valid_today.
- **Sortable attributes:** distance, price, discount %, created_at.
- **Stop words:** EN + AR common words.
- **Synonyms:** common UAE-specific (e.g., "spa" ↔ "wellness", "brunch" ↔ "buffet").
- **Typo tolerance:** enabled with sensible thresholds.

### Backups

Nightly snapshot to `volu-backups/meilisearch/` S3 prefix; weekly cross-region replication.

### Monitoring

- Query latency (P50, P95)
- Indexing latency (event → searchable)
- Disk usage
- Failed indexing tasks

---

## Capacity planning

| Resource         | Current | Trigger to scale                      |
| ---------------- | ------- | ------------------------------------- |
| RDS storage      | 200 GB  | 70% used → bump auto-scale            |
| RDS CPU          | 2 vCPU  | sustained > 70% → upgrade instance    |
| RDS replicas     | 2       | replica CPU sustained > 70% → add 3rd |
| Redis memory     | 13 GB   | 70% used → upgrade instance           |
| Meilisearch disk | 50 GB   | 70% used → grow volume                |

---

## Common operations

### Add a column (zero-downtime)

1. Migration: `ALTER TABLE ... ADD COLUMN ... NULL;`
2. App deployed reading new column (with null-handling).
3. (Optional) backfill data via worker job.
4. (Optional, later) `ALTER COLUMN ... SET NOT NULL;` after backfill completes.

### Add an index on a hot table

- Always `CREATE INDEX CONCURRENTLY`.
- Monitor `pg_stat_progress_create_index`.

### Restart a replica

- AWS console: reboot read replica without forcing failover.
- App reroutes reads to remaining replicas + primary fallback.

### Failover primary

- AWS console: "Reboot with failover" or trigger via API.
- ALB sees new primary via DNS update; PgBouncer reconnects.
- Total downtime: typically < 60s.

### Restore to a sandbox

1. RDS console: snapshot → restore to new instance.
2. Apply security groups + parameter group.
3. Connect from sandbox app using temporary credentials.

---

## See also

- [Cloud Topology](./01-cloud-topology.md)
- [Migrations Strategy](../04-data-model/03-migrations-strategy.md)
- [DR & Backup](./06-dr-backup.md)
