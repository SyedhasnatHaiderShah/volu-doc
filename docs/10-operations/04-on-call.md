# 04 · On-Call

**Status:** 🟢 Approved

How Volu staffs, compensates, and supports on-call engineers — and how we keep the practice sustainable rather than corrosive.

---

## Why we have an on-call

Volu's users are buying coupons in real time, redeeming them at merchant counters, and depending on push notifications about expiring deals. We don't have the luxury of "we'll fix it Monday morning." Production must be staffed 24/7.

But on-call carries a real cost in human terms. The goal is to make on-call **rare, predictable, well-supported, and fairly compensated** — not to grind senior engineers into burnout.

---

## Rotation structure

### Roles

- **Primary on-call** — first responder. Receives the page. Acknowledges within 5 minutes (SEV-1) or 15 minutes (SEV-2). Drives initial triage.
- **Secondary on-call** — backup. Paged automatically if primary doesn't ack within the response SLA, or escalated to by primary if hands-on help needed.
- **Manager on-call** — engineering lead or director-level. Available for SEV-1 escalations; coordinates external comms; makes business calls.

### Rotation cadence

- Primary: weekly (7 days, Saturday 09:00 → Saturday 09:00 UAE).
- Secondary: weekly, offset from primary so the same person isn't both.
- Manager: monthly.

### Pool size

- Primary pool: minimum 6 engineers. With 6 people on a weekly rotation, each does primary 1 week every 6 weeks.
- Secondary pool: same size as primary, drawn from the same pool but offset.
- Manager pool: 2–3 leads on monthly rotation.

We avoid rotations smaller than 6 because that pushes individuals to once-a-month or more, which is where on-call burnout starts.

### Joining the rotation

New engineers don't go on-call until:
1. Three months tenure.
2. Shadow at least 2 weeks of someone else's rotation.
3. Demonstrated competence: led at least 2 incident reviews; can deploy + rollback unaided.
4. Familiar with [Runbooks](./02-runbooks.md), [Incident Response](../06-security/04-incident-response.md), and the deployment pipeline.

The week after joining, they shadow the experienced on-call as the secondary, with explicit handoff if a real incident hits.

---

## Tooling

- **PagerDuty** (or OpsGenie) for paging.
- **Slack `#alerts`** — automated alerts post here; non-paging severity.
- **Slack `#incident-active`** — created per active incident; war room.
- **Grafana** + **Sentry** + **CloudWatch** for diagnosis.
- **Statuspage** for external comms.
- **Runbook links** embedded in every alert.
- **Phone + laptop with mobile hotspot capability** — required during your shift. Volu provides a corporate hotspot device if home internet isn't reliable.

---

## Response SLAs

| Severity | Acknowledgement | Engagement | Resolution target |
|---|---|---|---|
| SEV-1 | 5 min | Active triage immediately | 1 hour |
| SEV-2 | 15 min | Active triage within 30 min | 4 hours |
| SEV-3 | Next business hour | Same business day | Within sprint |
| SEV-4 | Next business day | When time permits | Backlog-prioritised |

**Failure to acknowledge** within the SLA escalates automatically:
- SEV-1: Primary not ack in 5 min → Secondary paged; not ack in 10 min → Manager paged.
- SEV-2: Primary not ack in 15 min → Secondary paged; not ack in 30 min → Manager paged.

---

## Escalation path

```
Primary on-call
   │
   │  (no ack OR primary requests help OR scope grows)
   ▼
Secondary on-call
   │
   │  (still need help OR business decision required OR > 30 min into SEV-1)
   ▼
Manager on-call
   │
   │  (regulatory exposure, leadership decision, prolonged outage)
   ▼
CTO / Engineering Director
   │
   ▼
CEO (data breach, public-relations risk, regulatory investigation)
```

Escalating is **encouraged, not stigmatised**. "I need help" is a sign of good judgement, not weakness.

---

## On-call expectations during a shift

You're expected to:

- Carry your phone with sound on, charged, with cellular coverage.
- Be reachable within ~15 minutes of being paged. (Not necessarily at a desk — but able to get to one.)
- Avoid activities that prevent fast response (long flights, deep underwater, performing surgery).
- Respond to pages and drive incidents to acknowledgement, mitigation, and handoff.
- Read the daily ops report at the start of your shift to understand current state.

You're **not** expected to:

- Sit at your desk all week.
- Cancel social plans by default.
- Solve incidents alone — escalation is always available.
- Work full days on top of on-call response — see "Compensation" below.

If life intervenes (illness, family emergency), swap with someone. PagerDuty has a swap feature. Inform the team.

---

## Compensation

On-call is paid work, separate from regular salary.

### Standby compensation

Every on-call shift earns a flat **standby allowance** for being available, regardless of pages received.

Approximate scale (calibrated locally; adjust to market):
- Primary, weekly shift: AED [X] flat.
- Secondary, weekly shift: 50% of primary.
- Manager, monthly shift: AED [Y] flat.

### Active-incident compensation

For time spent actively responding to a SEV-1 or SEV-2 outside business hours:
- Time-and-a-half hourly rate, billed in 30-minute increments.
- Minimum 1-hour billing per page (so a 5-minute page that resolves quickly still earns a full hour).

### Time off after major incidents

After a SEV-1 incident that consumed > 4 hours of out-of-hours response:
- The engineer takes the next business day off (paid).
- Encouraged to skip the rest of the on-call week if needed; a teammate covers.

### Holiday & UAE public-holiday premium

Working on-call during a UAE public holiday or weekend (Saturday/Sunday) adds a 50% premium to the standby + active-incident rates.

---

## Handoff

Every Monday morning, the outgoing primary writes a brief handoff in `#oncall-handoff`:

```
Outgoing: @hasnat
Incoming: @sara

This week's signal:
- 2 SEV-3s related to the new search index (resolved, monitoring)
- Stripe latency briefly elevated Tuesday afternoon (provider issue, no action)
- One brittle alert: outbox-backlog firing on every deploy (ticket VOLU-XXXX created)

Open follow-ups:
- VOLU-XXXX: outbox-backlog alert tuning
- VOLU-XXXY: Investigate flaky redemption-validate test

Anything special I should know:
- We have a flash deal launching Friday 9pm — pre-scaled fleet ready, see RB-012.
- Marketing is pushing a WhatsApp campaign Wednesday — possible OTP-volume spike.
```

The incoming engineer reads, asks questions, accepts the handoff.

---

## Blameless culture

When something breaks:

- We focus on **what failed in our systems and processes**, not who.
- Specific phrasing matters: "the deploy didn't catch the migration issue" not "Hasnat deployed the bad migration."
- Post-mortems name people only neutrally (e.g., "the on-call engineer paged the secondary at 02:14"), never accusatorily.
- Root cause is usually multiple contributing factors — automation, communication, ambiguity — not a single human mistake.

If a person did something genuinely wrong (skipped a required check, ignored an alert), that's handled privately by their manager — never in the post-mortem.

---

## Drills

### Monthly fire drill

Once a month, a chaos-engineering exercise:
- Schedule announced 24h ahead.
- Pick a scenario from a list (kill a service, induce DB lag, fail a webhook).
- Run it during business hours in staging.
- On-call (rotating volunteer) responds as if real.
- Debrief: what worked, what didn't, runbook updates.

### Quarterly game day

Full-day exercise:
- Multiple cascading failures injected.
- Runs in production-replica environment.
- Whole engineering team participates.
- After-action review drives roadmap items.

---

## Post-mortem ownership

For every SEV-1 and SEV-2:
- The Incident Commander owns the post-mortem (or designates an owner).
- Draft within 5 business days.
- Published to the team within 10 business days.
- Action items tracked in a public (internal) register; reviewed monthly.
- Lessons-learned summary shared in the next all-hands.

For SEV-3, post-mortem is optional but encouraged when there's a meaningful learning.

---

## On-call health metrics

Reviewed monthly:

| Metric | Target |
|---|---|
| Pages per primary shift | ≤ 3 (median); ≤ 5 (P90) |
| Out-of-hours pages per shift | ≤ 1 |
| Time spent actively responding (hours per shift) | ≤ 4 |
| % of pages resolved without escalation | ≥ 60% |
| Engineer satisfaction with on-call | Survey > 4/5 |

If these trend negatively for 2 months, we treat it as a problem to solve — not a culture to accept. Likely fixes: tune noisy alerts, add automation, expand the rotation pool.

---

## Time-zone considerations

Volu is UAE-based; our team is mostly in `Asia/Dubai`. As we hire across regions:

- We aim to keep at least one on-call covering each major time zone we operate in.
- "Follow-the-sun" rotation considered once we have 3+ engineers in 3+ time zones.
- For now, with a single time zone, fair compensation for night-time pages matters most.

---

## What we don't do

- **No "always-on" engineers.** We do not assign someone to be implicitly on-call 100% of the time as part of regular salary.
- **No silent expectations.** If we want you to respond, we page you. Slack messages outside hours are not pages.
- **No "responder of last resort" without compensation.** Tech leads who frequently jump in informally get the same compensation as the rotation.
- **No retaliation.** Declining to do extra on-call beyond your assigned shift never affects performance reviews.

---

## Onboarding checklist for new on-call engineers

- [ ] Added to PagerDuty rotation
- [ ] Test page received and acknowledged
- [ ] Read [Incident Response](../06-security/04-incident-response.md)
- [ ] Read all entries in [Runbooks](./02-runbooks.md)
- [ ] Familiar with deploy + rollback scripts
- [ ] Has access to: AWS Console (read), Cloudflare Console (read), Sentry, Grafana, Stripe Dashboard, Tabby/Tamara dashboards, Statuspage admin
- [ ] Has phone with cellular coverage; corporate hotspot if needed
- [ ] Shadowed at least 2 weeks
- [ ] Led at least one drill response
- [ ] Has access to incident communication templates

---

## See also

- [Incident Response](../06-security/04-incident-response.md)
- [Runbooks](./02-runbooks.md)
- [SLA & SLOs](./03-sla-slos.md)
- [Observability](../07-infrastructure/05-observability.md)
