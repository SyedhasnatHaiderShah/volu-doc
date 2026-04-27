# 02 · Networking

**Status:** 🟢 Approved

VPC layout, security groups, and network controls.

---

## VPC layout

```
VPC: volu-prod-vpc (10.0.0.0/16)
├── AZ-1a (10.0.0.0 – 10.0.31.255)
│   ├── Public subnet         10.0.0.0/24   → ALB, NAT GW
│   ├── Private app subnet    10.0.10.0/24  → ECS tasks
│   └── Private data subnet   10.0.20.0/24  → RDS, ElastiCache
├── AZ-1b (10.0.32.0 – 10.0.63.255)
│   ├── Public subnet         10.0.32.0/24  → ALB, NAT GW
│   ├── Private app subnet    10.0.42.0/24  → ECS tasks
│   └── Private data subnet   10.0.52.0/24  → RDS, ElastiCache
└── AZ-1c (10.0.64.0 – 10.0.95.255)
    ├── Public subnet         10.0.64.0/24  → ALB, NAT GW
    ├── Private app subnet    10.0.74.0/24  → ECS tasks
    └── Private data subnet   10.0.84.0/24  → RDS read-replica
```

**Internet access:**

- Public subnets: direct internet via Internet Gateway (only ALB + NAT GW live here).
- Private app subnets: outbound only via NAT Gateway (per AZ for HA).
- Private data subnets: no internet access; egress only via PrivateLink to specific AWS services.

---

## Security groups (defence in depth)

### `sg-alb`

- **Inbound:** 443 from `0.0.0.0/0` (Cloudflare-only via WAF rules at app layer).
- **Outbound:** all to `sg-app-tasks` on app ports.

### `sg-app-tasks` (ECS tasks)

- **Inbound:** from `sg-alb` only on app port (e.g., 3000).
- **Outbound:**
  - to `sg-rds` on 5432
  - to `sg-redis` on 6379
  - to `sg-meilisearch` on 7700
  - to `0.0.0.0/0` on 443 (for external API calls — Stripe, Tabby, Tamara, Unifonic, FCM, etc.)

### `sg-rds`

- **Inbound:** from `sg-app-tasks` on 5432. From `sg-bastion` (JIT) on 5432 when bastion enabled.
- **Outbound:** none.

### `sg-redis`

- **Inbound:** from `sg-app-tasks` on 6379.
- **Outbound:** none.

### `sg-meilisearch`

- **Inbound:** from `sg-app-tasks` on 7700.
- **Outbound:** to S3 endpoint for snapshots.

### `sg-bastion` (rarely used; JIT only)

- **Inbound:** from approved SSO via session manager only (no direct SSH).
- **Outbound:** to `sg-rds`, `sg-redis` for emergency access.

---

## Network ACLs

Default-deny stance:

- **Public subnet NACL:** allow inbound 443, 80 (redirect), ephemeral. Allow outbound 443, 80.
- **Private app NACL:** allow inbound from VPC CIDR only. Allow outbound 443, 5432, 6379, 7700.
- **Private data NACL:** allow inbound 5432, 6379, 7700 from app subnets. Allow outbound only to S3 endpoint (for backups).

NACLs are stateless — be mindful of return traffic on ephemeral ports.

---

## NAT Gateways

One per AZ for HA. Cost-trade-off — consider NAT Instances for staging.

Outbound traffic from private app subnets routes:

- To AWS service endpoints → via PrivateLink (no NAT).
- To external internet (Stripe, Tabby, etc.) → via NAT GW.

---

## VPC endpoints (PrivateLink)

To avoid NAT egress for AWS service calls:

- **Gateway endpoint**: S3 (free).
- **Interface endpoints**:
  - Secrets Manager
  - Parameter Store (SSM)
  - KMS
  - ECR (api + dkr)
  - CloudWatch Logs
  - SQS (if used)
  - STS

---

## Cloudflare integration

Cloudflare proxies all traffic to ALB. Origin protection:

- **Authenticated origin pulls**: ALB only accepts requests with valid Cloudflare client cert.
- **IP allowlist** at ALB: only Cloudflare IP ranges allowed (auto-updated via Lambda fetching Cloudflare IPs).

This prevents direct access to ALB bypassing Cloudflare's WAF.

---

## TLS / certificates

- **Edge (Cloudflare)**: full Strict TLS; auto-renewed.
- **Origin (ALB)**: AWS ACM certificate for `*.volu.ae`; auto-renewed.
- **Backend**: TLS terminated at ALB; ALB-to-ECS over private network unencrypted (within VPC, acceptable).
- **Database**: TLS-only to RDS (`require_secure_transport=ON` enforced).
- **Redis**: in-transit encryption enabled.

Mobile apps pin the Cloudflare-issued public cert (or the issuing intermediate) to prevent MITM.

---

## DNS

- Cloudflare DNS for all `volu.ae` records.
- Internal Route 53 private hosted zone for `internal.volu.ae` (used inside VPC for service discovery — e.g., `db-primary.internal.volu.ae`).

---

## Egress policy

What's allowed to leave the VPC:

| Destination                             | Purpose         | Allowed                    |
| --------------------------------------- | --------------- | -------------------------- |
| Stripe API                              | Payments        | ✅                         |
| Tabby/Tamara API                        | BNPL            | ✅                         |
| Unifonic/Twilio API                     | SMS             | ✅                         |
| WhatsApp/Meta API                       | Messaging + ads | ✅                         |
| TikTok API                              | Ads             | ✅                         |
| FCM/APNs                                | Push            | ✅                         |
| Sentry                                  | Error reporting | ✅                         |
| Cloudflare R2                           | Object storage  | ✅                         |
| AWS services (KMS, Secrets, S3, ECR)    | Internal        | ✅ via PrivateLink         |
| Public package mirrors (npmjs, pub.dev) | Build only      | ✅ in CI subnet            |
| Anything else                           | —               | ❌ blocked at NAT/firewall |

This explicit allowlist sits behind a future egress-firewall (AWS Network Firewall when traffic justifies the cost).

---

## DDoS protection

Layered:

1. **Cloudflare** — automatic DDoS mitigation at edge (free with Pro plan).
2. **AWS Shield Standard** — enabled by default for ALB + Route 53.
3. **App-level rate limiting** — Redis sliding window on critical endpoints.

If Volu becomes a frequent target, upgrade to AWS Shield Advanced + Cloudflare Enterprise.

---

## See also

- [Cloud Topology](./01-cloud-topology.md)
- [Threat Model](../06-security/01-threat-model.md)
