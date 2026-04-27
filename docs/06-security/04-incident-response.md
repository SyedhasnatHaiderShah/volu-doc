# 04 · Incident Response

**Status:** 🟢 Approved

How Volu handles security and operational incidents — from detection through post-mortem.

---

## Severity ladder

| Sev       | Definition                                                                   | Examples                                                           |
| --------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **SEV-1** | Critical: customer data breach, payment system down, > 30% of users affected | Data leak, full outage, payment processor unreachable              |
| **SEV-2** | High: degraded service, security incident with limited blast radius          | API errors > 5%, single region down, suspected unauthorised access |
| **SEV-3** | Medium: feature broken, single merchant affected                             | Search returning stale data, one merchant's deals missing          |
| **SEV-4** | Low: cosmetic, non-urgent                                                    | Typo in copy, broken icon                                          |

---

## Roles during an incident

- **Incident Commander (IC)** — owns the response; makes decisions; ends the incident. The on-call engineer by default.
- **Communications Lead (CL)** — owns external + internal updates.
- **Subject Matter Expert (SME)** — technical lead for the affected area.
- **Scribe** — records the timeline.

For SEV-1 and SEV-2, all four roles are assigned. For SEV-3 and SEV-4, IC may handle communications.

---

## Channels

- **#incident-active** — Slack channel created per incident.
- **Status page** — public.volu.ae/status — updated every 30 minutes during SEV-1/2.
- **War-room call** — Google Meet link in #incident-active for SEV-1.

---

## Response playbook

### 1. Detect

Sources of detection:

- Alert from monitoring (Prometheus, Sentry, CloudWatch).
- User report (Intercom, App Store review, social media).
- Internal observation.

Anyone who notices a potential incident:

1. Posts in `#alerts` Slack channel.
2. Pages the on-call engineer.

### 2. Triage

On-call engineer assesses within 10 minutes:

- Is it real? (Reproduce.)
- What's the severity?
- Is there immediate danger to customers or data?

Decision:

- **Real incident** → declare; spin up `#incident-active`.
- **False alarm** → log; tune the alert.

### 3. Contain

For SEV-1 and SEV-2, the priority order is:

1. **Stop the bleeding.** Use the most aggressive remediation that doesn't create more damage:
   - Rollback the most recent deploy.
   - Disable the affected feature via feature flag.
   - Enable read-only mode on impacted module.
   - Block the offending IP / user / merchant.
2. **Preserve evidence.** Take screenshots, capture logs, snapshot database state if relevant.
3. **Document the timeline.** Scribe records every action with timestamp.

### 4. Communicate

#### Internal (every 30 min during active SEV-1/2):

- Current status (investigating / identified / mitigating / monitoring / resolved)
- Impact summary
- Next update ETA

#### External:

- **SEV-1 customer impact:** status page banner + push notification to affected users (if needed).
- **SEV-2:** status page note.
- **SEV-3/4:** no public communication.

#### Regulatory (data breach):

- **Within 72 hours of confirmed personal-data breach:** notify UAE Data Office (or relevant authority).
- **Without undue delay if high risk:** notify affected users.

Templates kept in `/compliance/incident-templates/`.

### 5. Resolve

- Confirm full restoration with metrics + smoke tests.
- Close the incident in #incident-active.
- Update status page to "Resolved."
- Schedule the post-mortem within 5 business days.

### 6. Post-mortem

Required for every SEV-1 and SEV-2. Optional but encouraged for SEV-3.

Template:

```markdown
# Incident Post-Mortem: [Title]

**Date:** YYYY-MM-DD
**Duration:** HH:MM (start → end)
**Severity:** SEV-X
**Author:** [name]
**Status:** Draft / Reviewed / Approved

## Summary

[2-3 sentence executive summary.]

## Impact

- Users affected: [number / %]
- Revenue impact: [AED estimate]
- Data impact: [yes/no — describe]

## Timeline (UAE time)

- HH:MM — [event]
- HH:MM — [event]
- ...

## Root cause

[Detailed technical explanation.]

## Detection

[How did we detect? Could we have detected sooner?]

## Resolution

[What did we do?]

## What went well

- ...

## What went poorly

- ...

## Action items

| Action | Owner | Due | Status |
| ------ | ----- | --- | ------ |
| ...    | ...   | ... | ...    |
```

Post-mortems are **blameless**: focus on systems and processes, not individuals.

---

## Specific runbooks

### Suspected data breach

1. **Contain.** Identify affected systems; revoke compromised credentials; isolate the affected service.
2. **Assess.** What data was accessed/exfiltrated? How many users? What kind?
3. **Preserve.** Snapshot logs, DB state, authentication records. Engage forensics partner if needed.
4. **Notify.**
   - Internal: legal + leadership immediately.
   - Regulatory: UAE Data Office within 72 hours of confirmation.
   - Users: without undue delay if high risk to rights.
5. **Remediate.** Patch the vulnerability; rotate keys; reset passwords if needed.
6. **Post-mortem.** Public-facing summary if appropriate.

### Payment processor outage (Stripe down)

1. Confirm outage via Stripe status page.
2. Show in-app banner: "Payments are temporarily unavailable. Coupons in your wallet still work."
3. Disable checkout via feature flag; allow browse and redemption to continue.
4. Monitor Stripe status; restore checkout when green.
5. If extended (> 1 hour): consider routing to backup processor (future work).

### Merchant-balance fraud detected

1. Freeze the suspect merchant's payout queue immediately.
2. Pause their active deals via admin.
3. Notify Finance + Operations + Legal.
4. Investigate: pull audit logs, redemption patterns, transaction records.
5. Communicate with merchant through formal channel.
6. Remediate: clawback if applicable; legal action if warranted; permanent suspension.

### Mass coupon fraud (e.g., shared codes circulating online)

1. Identify the affected deal(s).
2. Lock new redemptions on those coupons (temporary).
3. Notify the merchant.
4. Investigate source: leaked QR images? PIN-sharing pattern?
5. If leakage confirmed: invalidate affected coupons, refund users, comp them with a small credit.
6. Tighten controls (e.g., shorter PIN-attempt windows; biometric required for PIN reveal).

### Cashier device compromised

1. Revoke that cashier's session immediately.
2. Audit recent redemptions from that device for suspicious patterns.
3. Notify the merchant owner.
4. Replace device fingerprint binding.
5. Onboard the new device fresh.

### KYC document leak

1. Revoke S3 access keys involved.
2. Audit recent access logs to the KYC bucket.
3. Identify affected merchants.
4. Notify regulator + affected merchants.
5. Rotate KMS keys; re-encrypt affected documents.

### DDoS attack

1. Verify with Cloudflare dashboard.
2. Tighten Cloudflare rate limits / block suspect ranges.
3. Enable Cloudflare "Under Attack" mode if needed.
4. Monitor; if app remains stable, no user impact.
5. If app degrades: scale up fleet; add more aggressive rate limits at app layer.

---

## On-call

- **Primary on-call:** rotating weekly across senior engineers.
- **Secondary on-call:** another engineer; called if primary doesn't acknowledge in 10 min.
- **Manager on-call:** team lead; available for SEV-1 escalation.
- **Pager:** PagerDuty or OpsGenie.

Response time SLAs:

- SEV-1: ack within 5 min.
- SEV-2: ack within 15 min.
- SEV-3: next business day.

Escalation path: Primary → Secondary → Manager → CTO.

---

## Drills

- **Game day** quarterly: simulated incident with full response.
- **Tabletop exercise** annually for data breach scenarios.
- **DR drill** quarterly (restore from backup).

After each drill: post-mortem-style review of what worked and what didn't.

---

## Inventory of "break glass" actions

For SEV-1, the IC may use any of these immediately:

- Roll back the latest deploy (`./scripts/rollback.sh <service>`).
- Toggle any feature flag globally.
- Enable read-only mode on Catalog or Commerce module.
- Force-suspend a merchant or deal.
- Mass-revoke sessions for a user.
- Trigger maintenance mode on the API (returns 503 with localised message).
- Block IPs at Cloudflare.
- Roll Stripe API keys (with cooperation from Finance).

Each action is audit-logged.

---

## Lessons-learned register

Every post-mortem produces action items. They're tracked in a public (internal) register at `/operations/lessons-learned.md` with status. Quarterly review ensures action items don't rot.

---

## Communication templates

In `/compliance/incident-templates/`:

- `internal-status-update.md`
- `customer-notification-data-breach.md`
- `regulator-notification.md`
- `status-page-investigating.md`
- `status-page-resolved.md`
- `post-mortem.md`

All bilingual where customer-facing.

---

## See also

- [Threat Model](./01-threat-model.md)
- [Security Controls](./02-security-controls.md)
- [PCI / PDPL Compliance](./03-pci-pdpl-compliance.md)
- [Runbooks](../10-operations/02-runbooks.md)
- [On-call](../10-operations/04-on-call.md)
