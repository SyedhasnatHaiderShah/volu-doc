# 04 · CI / CD

**Status:** 🟢 Approved

How code goes from a developer's branch to production.

---

## Tools

- **Source:** GitHub.
- **CI:** GitHub Actions.
- **Container registry:** AWS ECR.
- **Mobile builds:** GitHub Actions with macOS runners + Fastlane (iOS), Linux runners (Android).
- **Mobile distribution:** TestFlight (iOS), Play Console internal testing (Android).
- **Web admin:** Vercel.
- **OTA hotfix (mobile):** Shorebird.
- **Deployment to AWS:** GitHub Actions OIDC → AWS IAM role; deploys via Terraform + ECS update.

---

## Branching model

- `main` — protected; deploys to staging then production.
- `feat/<scope>-<short-name>` — short-lived feature branches.
- `fix/<scope>-<short-name>` — bug fixes.
- `hotfix/<scope>` — emergency fixes off main.

Direct push to `main` is blocked. PR with ≥1 code-owner approval required.

---

## Pull request pipeline

On every PR:

```
1. Lint
2. Format check
3. Type check (TS strict)
4. Unit tests
5. Integration tests (with Testcontainers Postgres + Redis)
6. SAST (CodeQL or Semgrep)
7. Dependency scan (Snyk / Dependabot security)
8. Secret scan
9. Build container image (no push)
10. Comment on PR with coverage + lint diff
```

All must pass before merge.

If the PR touches `prisma/schema.prisma`:

- Migration is generated and committed to PR.
- CI runs the migration against an ephemeral DB.
- Schema reviewer is auto-tagged.

---

## Mobile CI

On every PR touching `volu-mobile/`:

```
1. flutter pub get (Melos bootstrap)
2. dart format --set-exit-if-changed
3. flutter analyze
4. flutter test (unit + widget + golden)
5. Per-app builds:
   - User app: build apk for Android, build ios-arm64 for smoke
   - Merchant app: same
6. Patrol/Maestro E2E tests (subset on emulator)
```

---

## Production pipeline

On merge to `main`:

```
1. CI re-runs all PR checks (idempotent)
2. Build production-tagged container images
3. Push to ECR with git-sha tag
4. Run Terraform plan (drift detection)
5. Deploy to staging
   ├── Apply DB migrations (npx prisma migrate deploy)
   ├── Update ECS task definitions
   └── ECS rolling deploy
6. Run smoke tests against staging
7. Manual approval gate (Slack #releases)
8. Deploy to production
   ├── Apply DB migrations
   ├── Update ECS task definitions
   └── Canary deploy (10% traffic for 10 min, then 100%)
9. Post-deploy verification (synthetic checks)
10. Notify Slack #releases on success or failure
```

Total typical deploy time: 12–18 minutes.

---

## Rollback

`./scripts/rollback.sh <service>` reverts to the previous task definition revision in ECS. Takes ~3 minutes.

DB migrations are expand-then-contract (see [Migrations Strategy](../04-data-model/03-migrations-strategy.md)) so app rollback never strands schema.

---

## Mobile release pipeline

Mobile apps don't auto-deploy. Process:

```
1. Engineer cuts a release branch from main: release/v1.4.0
2. Bump version in pubspec.yaml; CI auto-bumps build number
3. Push tag v1.4.0
4. CI builds:
   - iOS .ipa + uploads to TestFlight
   - Android .aab + uploads to Play Console
5. QA tests both apps on staging
6. PM/QA promote to:
   - TestFlight external (1 day)
   - Play Console closed beta (1 day)
7. Promote to production:
   - App Store: submit for review
   - Play Console: rollout to production (1% → 10% → 50% → 100% over 48h)
8. Monitor Sentry crash rate; halt rollout if regression
```

Cadence: every 2 weeks.

---

## Hotfix pipeline (mobile)

For critical bugs that can't wait for a normal release:

**Option A — App Store expedited release**

- Branch off the released tag.
- Apply minimal fix.
- Submit with "expedited review" justification.
- Apple typically approves within 24 hours.

**Option B — Shorebird OTA**

- Push a patch via Shorebird that ships at next app launch.
- Limited: cannot change native code; only Dart code.
- Reserved for true emergencies (security, data loss); each use logged.

---

## Web admin pipeline (Vercel)

- Auto-deploy on merge to `main` (production).
- Auto-deploy preview on every PR.
- Production-environment env vars sourced from AWS Secrets Manager via Vercel CLI integration.

---

## Preview environments

Every PR gets a preview environment for the admin web (Vercel). Backend preview environments are spun up on demand for high-risk PRs (using Terraform workspace).

---

## CI/CD security

- GitHub OIDC → AWS IAM short-lived credentials. No long-lived AWS keys in CI.
- ECR repository policies restrict pulls to ECS task roles only.
- Container images signed with Cosign; ECS verifies signature before pulling (where supported).
- Production deploy approval gate requires a manual approval from a different engineer than the PR author.
- Audit trail of all deploys retained.

---

## Quality gates summary

| Gate                          | When      | Threshold                            |
| ----------------------------- | --------- | ------------------------------------ |
| Lint pass                     | PR        | 100%                                 |
| Tests pass                    | PR + main | 100%                                 |
| Coverage ≥ threshold          | PR        | Backend: 80% statements; Mobile: 70% |
| SAST high findings            | PR        | 0                                    |
| Critical CVE in deps          | PR        | 0 (high CVE: 7-day SLA)              |
| Manual approval (prod deploy) | Release   | 1 person, not the author             |

---

## See also

- [Coding Standards](../09-development/02-coding-standards.md)
- [Testing Strategy](../09-development/03-testing-strategy.md)
- [Git Workflow](../09-development/04-git-workflow.md)
- [Release Process](../10-operations/01-release-process.md)
