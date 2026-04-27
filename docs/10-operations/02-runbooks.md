# 02 · Runbooks

**Status:** 🟢 Approved

The on-call engineer's playbook. Each runbook is short, focused, and tested in drills. Linked from every alert.

---

## RB-001 · API error rate > 5%

**Symptoms:** Sentry spike; Grafana panel shows red; users report errors.

**Steps:**

1. Acknowledge the page.
2. Open Grafana → API Health dashboard. Identify which endpoint is failing.
3. Check Sentry for stack traces; group similar errors.
4. Check recent deploys (last 30 min): is this correlated?
5. If correlated with a deploy:
   - **Rollback**: `./scripts/rollback.sh volu-core`.
   - Verify error rate drops within 5 min.
6. If not deploy-correlated:
   - Check upstream: Stripe / Tabby / Tamara / Unifonic — any provider issues?
   - Check DB: long-running queries? lock waits?
   - Check Redis: connection issues?
7. If contained but not resolved: investigate root cause; communicate ETA.
8. If not contained: declare SEV-1; engage Incident Commander; status page update.

---

## RB-002 · Database CPU sustained > 80%

**Symptoms:** Grafana DB panel red; query latency rising.

**Steps:**

1. Open Grafana → Database dashboard.
2. Identify top slow queries (`pg_stat_statements`).
3. Check for missing indexes (recent feature?).
4. Check connection count: any leaks?
5. If a single query: kill it (`SELECT pg_terminate_backend(<pid>)`); investigate root cause.
6. If load is genuinely high: scale RDS instance up (requires brief failover; 60s downtime).
7. Add new index via `CREATE INDEX CONCURRENTLY` if specific query identified.
8. Open ticket for follow-up if architectural fix needed.

---

## RB-003 · Stripe webhook backlog

**Symptoms:** Inbound webhooks queue depth growing; orders stuck in `pending`.

**Steps:**

1. Open queue dashboard.
2. Check worker health: are `volu-workers-default` tasks running?
3. Check Stripe webhook URL is reachable: `curl -I https://api.volu.ae/api/v1/webhooks/stripe`.
4. Check Stripe dashboard for delivery failures.
5. If workers crashed: scale up via ECS console.
6. If signature verification failing: check secret rotation; update `STRIPE_WEBHOOK_SECRET`.
7. Manually replay missed webhooks via Stripe dashboard if needed.

---

## RB-004 · Payment failure rate > 5%

**Symptoms:** Spike in `volu_payment_failures_total`.

**Steps:**

1. Check Stripe status page.
2. Check Tabby + Tamara status.
3. Identify pattern: specific card type? specific country? specific BNPL?
4. If gateway-wide: communicate via in-app banner: "Payments are temporarily unavailable. Try again shortly."
5. If 3DS-related: check 3DS-enrolment patterns; may need to lower 3DS threshold temporarily.
6. If single provider down: disable that method via feature flag; route to alternatives.
7. After resolution: re-enable; analyse and improve.

---

## RB-005 · DB replication lag > 30s

**Symptoms:** Read replicas drifting from primary.

**Steps:**

1. Check Grafana → DB → replication lag panel.
2. Likely cause: long-running write on primary.
3. Identify long-running queries: `SELECT * FROM pg_stat_activity WHERE state='active' ORDER BY query_start LIMIT 10;`.
4. Kill the offender if non-essential.
5. If lag persists: failover read traffic to primary temporarily.
6. Investigate: was there a large data import? bulk update? schedule them off-peak.

---

## RB-006 · Coupon redemption failures spike

**Symptoms:** Users / merchants reporting redemption failures.

**Steps:**

1. Check Sentry: what error?
2. Check "Could not validate QR" rate.
3. If spike on a single merchant: contact merchant; possible device-config issue.
4. If platform-wide: check `RedemptionService` for recent changes; rollback if correlated.
5. If signature key rotation issue: ensure both old and new keys are valid during rollover.
6. If offline-sync issue: check offline queue health.

---

## RB-007 · Outbox publisher backlog > 5 min

**Symptoms:** Events not flowing to subscribers; downstream effects delayed (notifications late, search not updating).

**Steps:**

1. Check `volu-workers-outbox` task health.
2. Check Redis Streams health.
3. Check `outbox_events` table: count of unpublished rows; oldest unpublished timestamp.
4. Restart outbox worker if stuck.
5. If specific event types failing: check `last_error` column; fix subscriber or DLQ.
6. After resolution: monitor catch-up.

---

## RB-008 · Mobile crash rate > 0.5%

**Symptoms:** Sentry reports new crash spike; release rollout halted.

**Steps:**

1. Open Sentry mobile dashboard.
2. Identify the new crash signature.
3. Identify affected versions, platforms, devices.
4. If reproducible: open a hotfix branch; fix; release via expedited App Store / Shorebird OTA.
5. If unclear: roll back release (Play Console: halt rollout; App Store: pause downloads of latest version).

---

## RB-009 · KYC OCR failure rate spike

**Symptoms:** Merchants reporting KYC stuck; queue building.

**Steps:**

1. Check IDfy / Sumsub provider status.
2. If provider down: switch to manual review until restored; notify ops.
3. If specific document type failing: investigate; may need to update extraction template.

---

## RB-010 · Stripe Connect payout failure

**Symptoms:** Merchant payouts failing; balance frozen; merchants complaining.

**Steps:**

1. Check Stripe Connect dashboard for the affected merchant.
2. Common causes: merchant Connect account requirements (Stripe asks for additional info); IBAN format; banking holiday.
3. If merchant action required: notify merchant via in-app + email with what to fix.
4. If platform issue: check our Connect integration logs.
5. If genuine bank reject: refund via manual bank transfer; investigate IBAN.

---

## RB-011 · Search returning empty / stale results

**Symptoms:** Users report not finding deals.

**Steps:**

1. Check Meilisearch task health.
2. Check `volu-workers-search` task.
3. Check search index size vs DB count of approved-active deals.
4. If significant drift: trigger full reindex (`pnpm cli:reindex-search`).
5. If Meilisearch is down: temporarily disable search UI in mobile; show category browse as fallback.

---

## RB-012 · Massive flash-deal launch (capacity surge)

**Symptoms:** Anticipated traffic spike; ahead of an announced flash deal.

**Steps (proactive):**

1. Pre-scale ECS tasks: `aws ecs update-service --service volu-core --desired-count 10`.
2. Confirm DB connection pool capacity.
3. Confirm Redis memory headroom.
4. Notify on-call to be available.
5. After spike: scale back down based on metrics.

---

## RB-013 · Suspicious authentication activity

**Symptoms:** Spike in failed admin logins or unusual access patterns.

**Steps:**

1. Identify the user / admin account.
2. Force-revoke all sessions for that account: `pnpm cli:revoke-sessions --user-id=<id>`.
3. Reset password; require re-2FA enrolment.
4. Investigate: legitimate user vs attacker? IP geolocation? device fingerprint?
5. If admin account: notify Security Lead immediately; SEV-2.
6. Audit other privileged accounts; consider mandatory password reset for all admins.

---

## RB-014 · Cloudflare WAF blocking legitimate traffic

**Symptoms:** Users reporting "Forbidden" or 403 errors from specific regions/devices.

**Steps:**

1. Check Cloudflare dashboard → Security → Events.
2. Identify the rule firing.
3. If false positive: temporarily disable that rule; investigate root cause.
4. Adjust rule sensitivity or add exception.
5. Notify affected users via support.

---

## RB-015 · Backup restoration drill (monthly)

**Steps:**

1. Pick the latest RDS snapshot.
2. Restore to a sandbox cluster (different name, separate VPC).
3. Connect; run integrity queries:
   - `SELECT count(*) FROM identity__users` matches expected.
   - Sample joins return data.
4. Compare row counts to production (within drift tolerance).
5. Tear down sandbox.
6. Log the result; share in Slack.

---

## RB-016 · Cross-region DR drill (quarterly)

See [DR & Backup](../07-infrastructure/06-dr-backup.md) for full DR runbook.

Quarterly drill:

1. Schedule a 4-hour window in staging.
2. Simulate region failure (terraform destroy primary).
3. Activate DR region.
4. Time-stamp each step.
5. Measure RTO; identify gaps.
6. Update runbook + DR scripts.

---

## RB-017 · GDPR / PDPL data subject request

**Steps:**

1. Verify the requestor's identity (phone/email match account).
2. For **export**: trigger `POST /admin/users/:id/export-data` → user gets signed URL within 24h.
3. For **deletion**: trigger soft-delete; financial records retained per regulation; communicate retention timeline to user.
4. Log in compliance register.

---

## RB-018 · WhatsApp delivery failures

**Symptoms:** WhatsApp messages not reaching users.

**Steps:**

1. Check WhatsApp Business API status (provider dashboard).
2. Check our message-template approval status.
3. If template rejected: revise + resubmit.
4. Fall back to SMS for affected message types.

---

## Runbook hygiene

- Every alert links to its runbook. If an alert fires without a runbook, that's a bug.
- Runbooks reviewed quarterly: outdated steps trimmed.
- Drills validate runbooks every quarter; failures lead to fixes.

---

## See also

- [Observability](../07-infrastructure/05-observability.md)
- [Incident Response](../06-security/04-incident-response.md)
- [DR & Backup](../07-infrastructure/06-dr-backup.md)
- [On-call](./04-on-call.md)
