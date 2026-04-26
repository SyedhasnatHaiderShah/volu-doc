# 04 · Git Workflow

**Status:** 🟢 Approved

How Volu engineers branch, commit, review, and merge.

---

## Branching model

**Trunk-based with short-lived feature branches.** Aim: PRs merged within 1–3 days.

Branches:

- `main` — protected; always green; deployable.
- `feat/<scope>-<short-name>` — new features.
- `fix/<scope>-<short-name>` — bug fixes.
- `chore/<scope>-<short-name>` — refactor, deps, infra changes.
- `docs/<scope>-<short-name>` — docs changes.
- `hotfix/<scope>` — emergency fix off `main`; merges back fast.
- `release/<version>` — mobile release branches.

Examples:
- `feat/coupons-multi-use-redemption`
- `fix/payouts-stripe-retry`
- `chore/deps-bump-nestjs-10.4`

---

## Commit messages — Conventional Commits

```
<type>(<scope>): <subject>

[body]

[footer]
```

Types:
- `feat` — new feature
- `fix` — bug fix
- `chore` — maintenance
- `refactor` — code change without behaviour change
- `docs` — documentation
- `test` — adding/changing tests
- `perf` — performance improvement
- `style` — formatting only
- `build` — build system changes
- `ci` — CI changes

Subject:
- ≤ 72 characters
- Imperative mood ("add", not "added")
- No period at end

Body (optional):
- Explains *why*, not *what*.
- Refs ticket: `Closes VOLU-1234`.
- Note breaking changes prominently.

Examples:

```
feat(redemption): add offline queue with cryptographic signatures

When the merchant device is offline, redemptions are queued
locally with a device-signed payload. On reconnect, the queue
syncs to the server. The server is single source of truth and
rejects any double-spend attempts on sync.

Closes VOLU-845
```

```
fix(payouts): handle Stripe payout failure gracefully

Previously a failed Stripe Connect payout left the merchant
ledger in an inconsistent state. Now the failure rolls back
the payout and re-credits the merchant's available balance.

Closes VOLU-1502
```

`commitlint` runs on every PR; non-conforming subjects are blocked.

---

## Pull request rules

### Size

- Aim for ≤ 400 lines changed.
- Split larger work into stacked PRs.
- Pure refactors can be larger but should be split conceptually if possible.

### Title

Same Conventional Commits format. Squash-merge uses PR title as commit, so keep it tight.

### Description template

```markdown
## What this PR does
<one-paragraph summary>

## Why
<problem being solved; link to ticket / RFC>

## How
<key technical decisions; alternatives considered>

## Testing
- [ ] Added unit tests
- [ ] Added integration tests
- [ ] Manually tested against staging
- [ ] Updated docs

## Related
- Closes VOLU-XXX
- Refs ADR-NNNN
```

### Approvals

- ≥ 1 approval from a code owner (CODEOWNERS file).
- 2 approvals required for changes to: payments, payouts, security, auth, schema migrations.
- The author cannot self-approve.

### CI gates (must pass)

- Lint
- Type check
- Unit + integration tests
- SAST
- Dependency security scan
- Secret scan
- Build

### Merge

- **Squash and merge** is the default.
- The squash commit message is the PR title + description body.
- Direct merge for special cases (multi-commit PRs that are deliberately structured).
- Rebase forbidden by policy on `main`.

### After merge

- Branch auto-deleted by GitHub.
- CI deploys to staging.
- Author shepherds to production.

---

## Code review

### What reviewers look for

In order of importance:

1. **Correctness** — does this do what it claims?
2. **Tests** — are critical paths tested?
3. **Security** — any new attack surface?
4. **Performance** — any new N+1, expensive query, render hot path?
5. **Readability** — could a new joiner understand this in 6 months?
6. **Consistency** — follows our patterns?
7. **Style** — last (linter handles most of this).

### What reviewers don't do

- Bike-shed on naming if both options are reasonable.
- Block merges for non-essential changes (suggest follow-up).
- Re-architect the PR (do it in a separate refactor PR).

### Comment conventions

- `?` — question (no change required by author).
- `nit:` — minor stylistic suggestion (author's discretion).
- `praise:` — explicitly call out something well done.
- (no prefix) — required change.
- `blocking:` — strong objection; must be discussed before merge.

### Tone

- Pull-request review is collaborative, not adversarial.
- Disagreements escalate to a third reviewer if not resolvable.
- Authors thank reviewers for catches; reviewers credit good work.

---

## Special PR types

### Migration PRs

Schema migration PRs:
- Auto-tag schema reviewer.
- Must specify if expand or contract step (see [Migrations Strategy](../04-data-model/03-migrations-strategy.md)).
- Backfills documented separately.
- 2 approvals required.

### Security PRs

Security-related (auth, secrets, encryption):
- 2 approvals; one must be Security Lead.
- Threat-model implications noted.

### Dependency upgrade PRs

- Dependabot opens automatically.
- Auto-merge enabled for safe semver-patch updates after CI green.
- Major-version updates: manual review; document migration considerations.

### Hotfix PRs

- Branch off latest `main` (or release tag).
- Skip the full feature-PR flow only if SEV-1.
- Still requires approval (can be IC).
- After merge, cherry-pick or fast-forward back to all relevant branches.

---

## Code ownership

`.github/CODEOWNERS` declares per-area owners:

```
# Backend modules
/src/modules/identity/    @volu-eng/identity-team
/src/modules/payments/    @volu-eng/payments-team
/src/modules/payouts/     @volu-eng/payments-team
/src/modules/redemption/  @volu-eng/coupons-team

# Schema
/prisma/schema.prisma     @volu-eng/db-reviewers

# Security
/src/shared/auth/         @volu-eng/security-team
/src/shared/secrets/      @volu-eng/security-team

# Mobile
/apps/user_app/           @volu-eng/mobile-team
/apps/merchant_app/       @volu-eng/mobile-team

# Infra
/terraform/               @volu-eng/sre-team

# Docs
/docs/                    @volu-eng/docs-team
```

GitHub auto-assigns reviewers based on touched paths.

---

## Branch protection (production)

`main`:
- Direct push: disallowed.
- Force push: disallowed.
- Require PR review: yes.
- Require status checks: all CI gates.
- Require code owner review: yes.
- Require linear history: yes.

---

## Tags & releases

- Backend: tagged on production deploy: `backend-v<semver>-<sha>`.
- Mobile: tagged when releasing to TestFlight + Play: `mobile-user-v<semver>` and `mobile-merchant-v<semver>`.
- Admin: deployed by Vercel on every merge; tags `admin-v<semver>`.

GitHub Releases include changelog auto-generated from Conventional Commits.

---

## Stacked PRs

For large work split across PRs:

- Branch B branches off A; A targets `main`.
- Open PRs: A → main, B → A.
- Merge in order: when A merges, GitHub re-targets B to `main`.

Tools like Graphite help, but vanilla git is fine.

---

## Local hooks (Husky + lint-staged)

Pre-commit:
- Run formatter on staged files.
- Run linter on staged files.
- Block commit on lint errors.
- Run quick unit tests on changed files.

Pre-push:
- Run typecheck.
- Run full unit test suite (~30s).

---

## See also

- [Coding Standards](./02-coding-standards.md)
- [CI/CD](../07-infrastructure/04-ci-cd.md)
- [Release Process](../10-operations/01-release-process.md)
