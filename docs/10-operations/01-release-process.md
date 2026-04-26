# 01 · Release Process

**Status:** 🟢 Approved

How Volu ships changes to production safely and quickly.

---

## Release cadences

| Surface | Cadence | Mechanism |
|---|---|---|
| Backend | Continuous (multiple per day) | Auto-deploy on merge to `main` after staging |
| Admin web | Continuous | Vercel on merge |
| Mobile (iOS + Android) | Every 2 weeks | App Store + Play Console rollout |
| Hotfix mobile (rare) | As needed | Expedited App Store review or Shorebird OTA |
| Database schema | Continuous (within expand-then-contract pattern) | Co-deployed with code |

---

## Backend / admin release

```
1. PR opened
2. CI: lint, types, tests, scans
3. Review + approval
4. Merge to main
5. CI re-runs and builds container image
6. Auto-deploy to staging
7. Smoke tests against staging (synthetic monitor)
8. Manual approval gate (Slack #releases — anyone with role can approve, but not the author)
9. Production deploy:
   - Apply DB migrations
   - Update ECS task definitions
   - Canary 10% → 100% over 10 min (with auto-rollback on error spike)
10. Post-deploy synthetic checks
11. Slack notification: ✅ deployed
```

Rollback: `./scripts/rollback.sh volu-core` reverts to the previous task definition revision.

---

## Mobile release

```
Sprint planning ──▶ Code freeze (T-3 days)
                           │
                           ▼
                    Cut release branch
                           │
                           ▼
                    Build + sign + upload
                           │
                           ▼
                    Internal QA (1 day)
                           │
                           ▼
                    Promote to TestFlight external + Play closed beta
                           │
                           ▼
                    Beta validation (1 day)
                           │
                           ▼
                    Submit to App Store + start Play rollout
                           │
                           ▼
            Apple review (1-3 days) │ Play rollout 1% → 10% → 50% → 100% over 48h
                           │
                           ▼
                    Monitor crash + analytics
                           │
                           ▼
                    Tag release; post release notes
```

Crash-rate threshold for halting rollout: > 0.5% sessions on the new version.

---

## Hotfix process

When a critical bug must ship outside the normal cadence:

1. Branch off latest `main` (or release tag for mobile).
2. Apply minimal fix.
3. PR with `hotfix:` prefix; same review rules apply.
4. Merge → expedited deploy.
5. Backport to any active release branches.

For mobile:
- **App Store expedited review** — submit with justification; usually approved within 24h.
- **Shorebird OTA** — only for true emergencies (security, data loss); each use logged.

---

## Pre-release checklist (mobile)

Before promoting a release branch:

- [ ] All P0/P1 issues resolved
- [ ] Crash rate on previous release < 0.3%
- [ ] No regressions in critical synthetic flows
- [ ] Localisation review (EN + AR) complete
- [ ] Accessibility checks (VoiceOver + TalkBack key flows)
- [ ] Performance: cold start, scroll FPS within targets
- [ ] Bundle size within targets
- [ ] Release notes drafted (EN + AR for store; English for changelog)
- [ ] Marketing aware of new features

---

## Release notes

Two versions:

### Internal changelog (`CHANGELOG.md`)

Auto-generated from Conventional Commits. Detailed; for engineers.

### Public release notes (App Store + Play Console)

Concise; user-facing. Bilingual.

```
v1.4 — April 23, 2026

🆕 Apple Wallet support — add coupons with one tap
🆕 Multi-use coupons — perfect for visit packs
✨ Faster home feed loading
🐛 Fixed: rare crash on ordering during low connectivity

ما الجديد:
🆕 إضافة الكوبونات إلى Apple Wallet بنقرة واحدة
🆕 كوبونات متعددة الاستخدام — مثالية للزيارات المتعددة
✨ تحميل أسرع للصفحة الرئيسية
🐛 تم إصلاح عطل نادر عند الطلب في حالة الشبكة الضعيفة
```

---

## Feature flags

Risky features ship behind a flag. Flags toggled in admin:

- Per-user-cohort
- Per-percentage of traffic
- Globally on/off

Changes propagate within 60s (via Redis pub/sub).

Use flags for:
- New checkout flow rolling out gradually
- Experimental UI variants
- Quick disable for a buggy feature

---

## Database migrations

See [Migrations Strategy](../04-data-model/03-migrations-strategy.md). Summary:

- Expand-then-contract pattern.
- Apply migration → deploy code that uses new schema → wait → contract.
- Migrations run automatically via `npx prisma migrate deploy` before app deploy.
- Long migrations (backfills) run as separate worker jobs, not in deploy pipeline.

---

## Backwards compatibility for mobile

The backend supports the latest **2 minor versions** of mobile clients at any time. Deprecation policy:

- A version is "supported" if active install base ≥ 2%.
- 6 months of advance warning before forcing an upgrade.
- Force-update screen via `app_versions` config.

---

## Communication

### Internal
- Each release has a Slack thread in `#releases` with deploy markers.
- Major features get a brief written summary linked.

### External (mobile)
- Release notes in stores.
- In-app "What's new" splash for major releases (sparingly used).

---

## See also

- [CI/CD](../07-infrastructure/04-ci-cd.md)
- [Git Workflow](../09-development/04-git-workflow.md)
- [Migrations Strategy](../04-data-model/03-migrations-strategy.md)
