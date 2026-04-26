# 05 · Admin Console Architecture

**Status:** 🟢 Approved

The Volu admin console — internal-only browser-based tool for the Volu team. Built with Next.js 14.

---

## Layout

```
volu-admin/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── layout.tsx                # root layout (auth gate, theme)
│   │   ├── page.tsx                  # → redirect to /dashboard
│   │   ├── (auth)/                   # auth-free routes
│   │   │   ├── login/page.tsx
│   │   │   └── 2fa/page.tsx
│   │   ├── (app)/                    # auth-required routes
│   │   │   ├── layout.tsx            # main shell
│   │   │   ├── dashboard/
│   │   │   ├── merchants/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── kyc-queue/
│   │   │   │   └── [id]/
│   │   │   ├── deals/
│   │   │   │   ├── moderation/
│   │   │   │   └── [id]/
│   │   │   ├── coupons/
│   │   │   ├── orders/
│   │   │   ├── refunds/
│   │   │   ├── payouts/
│   │   │   ├── invoicing/
│   │   │   ├── raffles/
│   │   │   ├── marketing/
│   │   │   ├── ads/
│   │   │   ├── content/
│   │   │   ├── users/
│   │   │   ├── audit-log/
│   │   │   └── settings/
│   │   └── api/                      # BFF endpoints (proxy to backend)
│   │
│   ├── components/
│   │   ├── ui/                       # shadcn/ui components
│   │   ├── tables/                   # TanStack Table wrappers
│   │   ├── charts/                   # Tremor / Recharts
│   │   ├── forms/                    # form patterns
│   │   ├── shell/                    # nav, sidebar, breadcrumbs
│   │   └── feature-specific/
│   │
│   ├── lib/
│   │   ├── api/                      # generated typed API client
│   │   ├── auth/                     # NextAuth config + custom logic
│   │   ├── utils/
│   │   └── constants/
│   │
│   ├── hooks/
│   ├── styles/
│   └── types/
│
├── public/
├── tailwind.config.ts
├── next.config.mjs
└── package.json
```

---

## Server vs Client components

We use the App Router model deliberately:

| Pattern | When to use |
|---|---|
| Server Component (default) | Pages with data tables, lists, dashboards. SSR for fast first paint; reduced JS shipped. |
| Client Component (`'use client'`) | Interactive forms, real-time widgets, anything with state. |
| Hybrid | Server Component shell + Client Component interactivity inside. |

**Rule:** Default to Server Component. Add `'use client'` only when you need state, effects, or browser APIs.

---

## Authentication

NextAuth with custom credentials provider that calls Volu Identity API:

```ts
// app/api/auth/[...nextauth]/route.ts
const handler = NextAuth({
  providers: [
    CredentialsProvider({
      async authorize(credentials) {
        const res = await fetch(`${env.API_URL}/api/v1/admin/auth/login`, {
          method: 'POST',
          body: JSON.stringify(credentials),
        });
        if (!res.ok) return null;
        return await res.json();
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.accessToken = user.accessToken;
        token.refreshToken = user.refreshToken;
        token.role = user.role;
        token.requires2fa = user.requires2fa;
      }
      // Refresh access token if near expiry
      if (Date.now() > token.accessExpiry - 60_000) {
        return await refreshAccessToken(token);
      }
      return token;
    },
    async session({ session, token }) {
      session.accessToken = token.accessToken;
      session.role = token.role;
      return session;
    },
  },
  pages: { signIn: '/login' },
});
```

After password verification, the user is redirected to `/2fa` to enter their TOTP. Only after successful 2FA does the session become fully active.

---

## API access pattern

Admin doesn't talk to the public API directly. It talks to **Next.js Server Actions** or **API routes** which forward to the backend with the admin's access token. This:
- Keeps the backend API URL private (server-side only).
- Allows server-side auth header injection.
- Lets us add admin-specific transformations.

```ts
// app/api/merchants/route.ts (or use Server Action)
export async function GET(req: Request) {
  const session = await auth();
  if (!session) return new Response('Unauthorized', { status: 401 });

  const { searchParams } = new URL(req.url);
  const res = await fetch(`${env.API_URL}/api/v1/admin/merchants?${searchParams}`, {
    headers: { Authorization: `Bearer ${session.accessToken}` },
  });
  return new Response(await res.text(), {
    status: res.status,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

---

## Data fetching

| Pattern | Use case |
|---|---|
| Server Component fetch | Initial page load (dashboard, list view) |
| `useQuery` (TanStack) | Polling (live KPI strip), revalidation |
| `useMutation` (TanStack) | Mutations (approve KYC, refund order) |
| Server Action | Form submissions, simple mutations |

All TanStack Query keys follow the pattern `[resource, scope, params]`:
```ts
useQuery({
  queryKey: ['merchants', 'pending-kyc', { page, search }],
  queryFn: () => api.merchants.list({ status: 'under_review', page, search }),
});
```

---

## Tables (TanStack Table)

Data-heavy admin = lots of tables. We standardise:

- Server-side pagination (default 25 rows/page).
- Server-side sorting (one column at a time; default by created_at desc).
- Server-side filtering with debounced inputs.
- Column toggling persisted to localStorage per table.
- Row selection for bulk actions where applicable.
- Sticky header on long pages.
- Empty states and loading skeletons.

A `<DataTable />` component wraps TanStack Table with our defaults.

---

## Charts (Tremor)

Tremor provides React chart components with sensible Tailwind-friendly defaults. Use cases:

- Line charts for time-series KPIs (GMV, orders, redemptions).
- Bar charts for comparisons (per-category GMV).
- Funnels for conversion (install → purchase → redeem).
- Heatmaps for cohort retention.
- Sparklines for KPI tiles.

Custom charts (raffle entry distribution, ROAS curves) use Recharts directly.

---

## Forms

Pattern: `react-hook-form` + `zod` for schema-validated forms.

```tsx
const refundSchema = z.object({
  amount: z.number().positive().max(orderTotal),
  reason: z.enum(['merchant_failure', 'misrepresentation', 'goodwill', 'other']),
  note: z.string().max(500).optional(),
});

type RefundForm = z.infer<typeof refundSchema>;

function RefundDialog({ order }: { order: Order }) {
  const form = useForm<RefundForm>({
    resolver: zodResolver(refundSchema),
    defaultValues: { amount: order.total, reason: 'goodwill' },
  });

  const mutation = useMutation({
    mutationFn: (data: RefundForm) => api.refunds.create({ orderId: order.id, ...data }),
    onSuccess: () => toast.success('Refund processed'),
    onError: (e) => toast.error(formatError(e)),
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((data) => mutation.mutate(data))}>
        {/* ... */}
      </form>
    </Form>
  );
}
```

---

## RBAC enforcement (UI side)

The backend enforces permissions; the UI hides what the user can't do:

```tsx
function MerchantDetailPage({ merchant }: Props) {
  const session = await auth();
  const can = canFor(session.role);

  return (
    <>
      {can('merchant.suspend') && (
        <Button onClick={suspend}>Suspend</Button>
      )}
      {can('payout.approve') && (
        <Link href={`/payouts?merchant=${merchant.id}`}>View payouts</Link>
      )}
    </>
  );
}
```

`canFor()` returns a function from a static role-permission matrix. The matrix is the same one enforced by the backend's RBAC guard — generated from a single source-of-truth YAML committed to the repo.

---

## Dashboards: real-time vs cached

| KPI tile | Refresh strategy |
|---|---|
| Today's GMV | Polling every 30s |
| Pending payouts count | Polling every 60s |
| Live deals count | Polling every 5 min |
| Time-series charts | Cached 5 min, manual refresh button |
| Cohort heatmap | Daily-computed, instant load |
| Anomaly alerts banner | Server-pushed via SSE (Server-Sent Events) |

---

## Audit-log surface

Every privileged action (approve/reject KYC, suspend merchant, issue refund, manual coupon, adjust commission, etc.) writes an audit log entry server-side. Admin pages display:

- "Last edited by Omar at 14:23 on 15 Apr."
- "View change history" → opens audit-log drawer with diff per change.

This builds trust within the team and makes incident review easy.

---

## Styling

- Tailwind for utilities.
- shadcn/ui for the component base (we own the source).
- CSS variables for theming (light + dark mode).
- Bilingual: admin is **English-only** (operated by Volu staff).

---

## Performance targets

- TTFB ≤ 200ms (UAE → infra).
- LCP ≤ 1.5s on a typical office 4G.
- Bundle: route-level code splitting via App Router.
- Images: optimised via `next/image`.
- Heavy admin tables: virtualised after 100 rows.

---

## Security

- HSTS, CSP, X-Frame-Options: DENY, X-Content-Type-Options: nosniff.
- HttpOnly + Secure + SameSite=Strict cookies for session.
- 2FA mandatory.
- IP allowlist option for Super Admin.
- Audit log of every action.
- Session timeout after 30 min idle; max session 8h.
- `next.config.mjs` security headers configured.

---

## Testing

- Component tests with React Testing Library.
- E2E with Playwright covering critical admin flows (KYC approval, refund, payout, raffle creation).
- Visual regression with Chromatic on key screens.

---

## See also

- [Tech Stack](./02-tech-stack.md)
- [Backend Architecture](./03-backend-architecture.md)
- [Security Controls](../06-security/02-security-controls.md)
