# 05 · Dev Environment

**Status:** 🟢 Approved

How to set up Volu locally on day one.

---

## Prerequisites

| Tool           | Version       | Why                          |
| -------------- | ------------- | ---------------------------- |
| Node.js        | 20 LTS        | Backend, admin               |
| pnpm           | 9.x           | Workspace + faster than npm  |
| Docker Desktop | latest        | Postgres, Redis, Meilisearch |
| Flutter        | latest stable | Mobile                       |
| Xcode          | latest        | iOS builds (macOS only)      |
| Android Studio | latest        | Android emulator             |
| Terraform      | 1.6+          | Infra                        |
| AWS CLI        | v2            | Deploys                      |
| direnv         | latest        | env-var management           |
| GitHub CLI     | latest        | PR workflow                  |

A `Brewfile` (macOS) and a `setup-linux.sh` script automate the install.

---

## Cloning the repos

```bash
mkdir -p ~/volu && cd ~/volu

gh repo clone volu/volu-backend
gh repo clone volu/volu-admin
gh repo clone volu/volu-mobile
gh repo clone volu/volu-infra
gh repo clone volu/volu-docs
```

---

## Backend setup

```bash
cd ~/volu/volu-backend
cp .env.example .env.local
# fill in local secrets — see comments in .env.example
pnpm install
docker compose -f docker-compose.dev.yml up -d  # Postgres, Redis, Meilisearch, MailHog
npx prisma migrate dev
npx prisma db seed
pnpm dev
```

Backend runs on `http://localhost:3000`.

OpenAPI docs at `http://localhost:3000/api/docs`.

### Test the install

```bash
curl http://localhost:3000/health
# {"status":"ok"}

curl http://localhost:3000/api/v1/categories
# returns category tree from seed data
```

---

## Admin setup

```bash
cd ~/volu/volu-admin
cp .env.example .env.local
# Set NEXT_PUBLIC_API_URL=http://localhost:3000
pnpm install
pnpm dev
```

Admin runs on `http://localhost:3001`.

Default admin login (from seed): `admin@volu.local` / `volu123!` / TOTP from `seed.ts` console output.

---

## Mobile setup

```bash
cd ~/volu/volu-mobile
dart pub global activate melos
melos bootstrap

# iOS
cd apps/user_app/ios && pod install && cd ../../..

# Run user app on iOS simulator
melos run dev:user:ios

# Run merchant app on Android emulator
melos run dev:merchant:android
```

Build flavors:

- **dev** — points to `http://localhost:3000`
- **staging** — points to `https://api-staging.volu.ae`
- **prod** — points to `https://api.volu.ae`

Default flavor: `dev`.

### iOS-specific

- Sign in to Xcode with your Apple ID for signing.
- Use the Volu development provisioning profile (Team's shared certificate).

### Android-specific

- An emulator with Google Play Services for FCM testing.
- Or use a physical device with USB debugging.

---

## Local services (Docker Compose)

`docker-compose.dev.yml` in `volu-backend`:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: volu_dev
      POSTGRES_USER: volu
      POSTGRES_PASSWORD: dev
    ports:
      - 5432:5432
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - 6379:6379

  meilisearch:
    image: getmeili/meilisearch:v1.6
    environment:
      MEILI_MASTER_KEY: dev-key
    ports:
      - 7700:7700
    volumes:
      - meilisearch_data:/meili_data

  mailhog:
    image: mailhog/mailhog
    ports:
      - 1025:1025 # SMTP
      - 8025:8025 # Web UI
```

`docker compose up -d` starts all services. Web UI: MailHog at `http://localhost:8025` to view emails sent locally.

---

## Environment variables

`.env.example` (committed):

```
# General
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://volu:dev@localhost:5432/volu_dev

# Redis
REDIS_URL=redis://localhost:6379

# Meilisearch
MEILI_HOST=http://localhost:7700
MEILI_MASTER_KEY=dev-key

# JWT (generate with: pnpm gen:jwt-keys)
JWT_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"
JWT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"

# Stripe
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Tabby
TABBY_PUBLIC_KEY=pk_test_...
TABBY_SECRET_KEY=sk_test_...

# Tamara
TAMARA_TOKEN=...

# SMS / OTP
UNIFONIC_APP_SID=...
UNIFONIC_SENDER_NAME=Volu
TWILIO_ACCOUNT_SID=...
TWILIO_AUTH_TOKEN=...

# Email
RESEND_API_KEY=...

# Sentry
SENTRY_DSN=...

# Cloudflare R2
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET_PUBLIC=volu-public-images-dev
R2_BUCKET_PRIVATE=volu-private-dev
```

For local dev, **most external services have mock/test mode**:

- Stripe: test API keys.
- Tabby/Tamara: sandbox keys.
- Unifonic: dev mode prints OTPs to console (no SMS sent).
- Resend: routes through MailHog (no real emails).

For Volu engineers, secrets injected via `direnv` from a 1Password vault entry.

---

## Common dev commands

```bash
# Backend
pnpm dev                          # start dev server
pnpm test                         # run all tests
pnpm test:integration             # integration tests with Testcontainers
pnpm gen:openapi                  # regenerate OpenAPI spec
pnpm db:migrate:dev               # create + apply a new migration
pnpm db:reset                     # reset DB and re-seed
pnpm db:studio                    # Prisma Studio GUI

# Admin
pnpm dev
pnpm test
pnpm gen:api                      # regenerate TS API client from openapi.json

# Mobile
melos run dev:user:ios
melos run analyze
melos run test
melos run gen:api                 # regenerate Dart API client
melos run format                  # apply dart format
```

---

## Troubleshooting

### "Database connection refused"

Docker Postgres not running. Run `docker compose up -d` from `volu-backend/`.

### "Prisma migration failed"

Likely DB is in a weird state. `pnpm db:reset` to wipe and re-seed.

### "Port 3000 already in use"

Another service or stale Node process. `lsof -i :3000` then kill.

### "Flutter build failed: pod install"

On macOS: `cd apps/user_app/ios && pod install --repo-update`. If still failing, `pod deintegrate && pod install`.

### "iOS simulator can't reach localhost:3000"

Use `http://localhost:3000` from simulator (it bridges automatically). For Android emulator, use `http://10.0.2.2:3000` — that's the emulator's name for the host.

---

## VS Code recommended extensions

`.vscode/extensions.json`:

```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "Prisma.prisma",
    "Dart-Code.dart-code",
    "Dart-Code.flutter",
    "bradlc.vscode-tailwindcss",
    "Anthropic.claude-code"
  ]
}
```

`.vscode/settings.json` (committed):

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "[dart]": {
    "editor.defaultFormatter": "Dart-Code.dart-code"
  }
}
```

---

## Productivity scripts

`scripts/setup.sh` — installs everything for a fresh laptop.
`scripts/reset.sh` — reset all local data (DB, Redis, caches).
`scripts/sync.sh` — pull all Volu repos to latest.
`scripts/test-all.sh` — run tests across all repos.

---

## See also

- [Monorepo Structure](./01-monorepo-structure.md)
- [Coding Standards](./02-coding-standards.md)
- [Testing Strategy](./03-testing-strategy.md)
- [Git Workflow](./04-git-workflow.md)
