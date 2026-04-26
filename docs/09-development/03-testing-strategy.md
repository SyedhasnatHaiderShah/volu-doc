# 03 · Testing Strategy

**Status:** 🟢 Approved

How Volu tests its software. Honest about trade-offs.

---

## Test pyramid

```
                E2E (few, slow, brittle, high-confidence)
                ────
          Integration (many, medium speed, real components)
          ──────────────
       Unit (many, fast, isolated, low-confidence)
       ──────────────────────
```

Volu's distribution:

| Layer | Backend | Mobile | Admin |
|---|---|---|---|
| Unit | 60% of tests | 50% | 50% |
| Integration | 35% | 30% (widget tests) | 30% |
| E2E | 5% | 20% (golden + Patrol) | 20% (Playwright) |

---

## Backend testing

### Unit tests (Jest)

Test pure logic in `domain/` and `application/`:

```ts
describe('Coupon entity', () => {
  it('rejects redemption when expired', () => {
    const coupon = makeCoupon({ expiresAt: yesterday });
    expect(() => coupon.assertEligibleForRedemption(ctx)).toThrow(CouponExpiredError);
  });

  it('decrements uses_remaining on multi-use redemption', () => {
    const coupon = makeCoupon({ usageType: 'multi_use', usesTotal: 5, usesRemaining: 5 });
    coupon.markRedeemed(ctx);
    expect(coupon.usesRemaining).toBe(4);
    expect(coupon.status).toBe(CouponStatus.Active);
  });

  it('marks single-use coupon redeemed after one use', () => {
    const coupon = makeCoupon({ usageType: 'single_use', usesTotal: 1, usesRemaining: 1 });
    coupon.markRedeemed(ctx);
    expect(coupon.usesRemaining).toBe(0);
    expect(coupon.status).toBe(CouponStatus.Redeemed);
  });
});
```

Rules:
- One assertion per test where possible.
- Test factories (`makeCoupon`) consolidate setup.
- Mock external dependencies; never real DB calls.
- Test names read as full sentences.

### Integration tests (Jest + Testcontainers)

Real Postgres, real Redis, mocked external HTTP. Tests at the service layer:

```ts
describe('RedemptionService', () => {
  let pg: StartedPostgreSqlContainer;
  let app: INestApplication;

  beforeAll(async () => {
    pg = await new PostgreSqlContainer('postgres:16').start();
    app = await createTestApp({ databaseUrl: pg.getConnectionUri() });
  });

  afterAll(async () => {
    await app.close();
    await pg.stop();
  });

  it('confirms redemption end-to-end', async () => {
    const { coupon, cashier } = await seedActiveCoupon();
    const result = await app.get(RedemptionService).confirmRedemption({
      qrToken: signedToken(coupon),
      pin: coupon.pin,
      branchId: coupon.eligibleBranchIds[0],
      cashierId: cashier.id,
    });
    expect(result.status).toBe('redeemed');

    const ledgerEntries = await app.get(WalletRepository).findByMerchant(coupon.merchantId);
    expect(ledgerEntries).toContainEqual(expect.objectContaining({ type: 'pending_release' }));
  });
});
```

Real DB catches integration bugs that mocks miss (missing FKs, constraint violations, transaction edge cases).

### Contract tests

For each external dependency (Stripe, Tabby, Tamara, Unifonic), we maintain contract tests that hit a sandbox or recorded fixture. Run nightly to catch upstream changes.

### E2E tests (supertest)

Full HTTP through the app, real DB:

```ts
describe('POST /api/v1/orders (E2E)', () => {
  it('creates an order with idempotency', async () => {
    const idemKey = uuidv4();

    const r1 = await request(app.getHttpServer())
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${userToken}`)
      .set('X-Idempotency-Key', idemKey)
      .send({ items: [{ dealId: deal.id, quantity: 1 }] });

    const r2 = await request(app.getHttpServer())
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${userToken}`)
      .set('X-Idempotency-Key', idemKey)
      .send({ items: [{ dealId: deal.id, quantity: 1 }] });

    expect(r1.status).toBe(201);
    expect(r2.status).toBe(201);
    expect(r1.body.data.id).toBe(r2.body.data.id);
  });
});
```

---

## Mobile testing

### Unit tests (`flutter_test`)

Pure logic in `domain/` and `application/`:

```dart
test('Coupon expires when valid_until is in the past', () {
  final coupon = makeCoupon(expiresAt: yesterday());
  expect(coupon.isExpired(now: now()), isTrue);
});
```

### Widget tests

Single widgets in isolation:

```dart
testWidgets('VoluPriceTag shows discount %', (tester) async {
  await tester.pumpWidget(MaterialApp(
    home: VoluPriceTag(originalFils: 40000, dealFils: 19500, currency: 'AED'),
  ));
  expect(find.text('AED 195'), findsOneWidget);
  expect(find.text('51% off'), findsOneWidget);
});
```

### Golden tests

Visual regression for design-system components and key screens:

```dart
testGoldens('VoluCouponCard light EN', (tester) async {
  await loadAppFonts();
  await tester.pumpWidgetBuilder(
    Center(child: VoluCouponCard(coupon: sampleCoupon)),
    surfaceSize: const Size(400, 300),
  );
  await screenMatchesGolden(tester, 'coupon_card_light_en');
});

testGoldens('VoluCouponCard dark AR', (tester) async {
  await loadAppFonts();
  await tester.pumpWidgetBuilder(
    Center(child: VoluCouponCard(coupon: sampleCoupon)),
    surfaceSize: const Size(400, 300),
    wrapper: (child) => Directionality(textDirection: TextDirection.rtl, child: child),
    themeData: voluDarkTheme,
  );
  await screenMatchesGolden(tester, 'coupon_card_dark_ar');
});
```

Goldens caught visual regressions during refactors.

### Integration tests (`flutter_test` `IntegrationTestWidgetsFlutterBinding`)

Full app boot, mocked backend:

```dart
testWidgets('full login flow', (tester) async {
  app.main();
  await tester.pumpAndSettle();
  await tester.tap(find.text('Continue with phone'));
  // ...
});
```

### E2E (Patrol or Maestro)

Real device / emulator, against a staging backend:

```yaml
# maestro flow
- launchApp
- tapOn: "Continue with phone"
- inputText: "+971501234567"
- tapOn: "Continue"
- inputText: "111111"  # test OTP
- assertVisible: "Today's Volu drops"
```

Run on CI nightly + before release.

---

## Admin testing

### Unit + component tests (Jest + React Testing Library)

```tsx
test('refund dialog disables submit while pending', () => {
  render(<RefundDialog order={mockOrder} />);
  fireEvent.click(screen.getByText('Refund'));
  expect(screen.getByRole('button', { name: 'Submitting…' })).toBeDisabled();
});
```

### E2E (Playwright)

Critical admin flows:

```ts
test('approve a KYC submission', async ({ page }) => {
  await login(page, opsAdmin);
  await page.goto('/merchants/kyc-queue');
  await page.click(`text=${merchant.name}`);
  await page.click('text=Approve KYC');
  await page.fill('textarea[name=notes]', 'All docs verified');
  await page.click('button:has-text("Confirm")');
  await expect(page.locator('text=Approved')).toBeVisible();
});
```

### Visual regression (Chromatic)

Storybook + Chromatic catch unexpected UI changes during refactors.

---

## Coverage targets

| Layer | Target |
|---|---|
| Backend domain logic | ≥ 90% |
| Backend services | ≥ 80% |
| Backend controllers | covered by E2E |
| Mobile domain | ≥ 90% |
| Mobile widgets | ≥ 60% (golden tests count) |
| Admin components | ≥ 70% |

Coverage is a guide, not a goal. 100% coverage on trivial code is wasted effort; high coverage on money-handling logic is essential.

---

## Property-based testing

For pure logic with many edge cases (e.g., raffle rule evaluator, price math, ledger reconciliation), use property-based tests with `fast-check` (TS) or `dart_test` properties:

```ts
import * as fc from 'fast-check';

it('vat is always 5% of subtotal-after-promo', () => {
  fc.assert(fc.property(
    fc.integer({ min: 1000, max: 1_000_000 }),
    fc.integer({ min: 0, max: 500_000 }),
    (subtotal, promo) => {
      const vat = computeVat({ subtotal, promo });
      expect(vat).toBe(Math.round((subtotal - Math.min(subtotal, promo)) * 0.05));
    },
  ));
});
```

---

## Mutation testing (optional, post-launch)

`Stryker` mutates the code and re-runs tests. If mutated code passes, you have a coverage gap. Useful for critical modules (Wallet, Payouts).

---

## Performance testing

### k6 for API load tests

```js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '1m', target: 0 },
  ],
};

export default function () {
  const res = http.get('https://api-staging.volu.ae/api/v1/deals?limit=10');
  check(res, {
    'status 200': (r) => r.status === 200,
    'p95 < 500ms': (r) => r.timings.duration < 500,
  });
}
```

Run before major releases and after architectural changes.

### Mobile performance profiling

Per release: profile on iPhone 12 + Galaxy S21 in profile mode. Track:

- Cold start time
- Frame rates on home feed scroll
- Memory usage on long sessions
- Network data per session

Compare to previous release; regressions are bugs.

---

## Test data

- **Fixtures** in `test/fixtures/` — realistic but anonymous data.
- **Factories** (`makeUser`, `makeMerchant`, `makeDeal`, `makeCoupon`) — programmatic generators.
- **Seeds** (`prisma/seed.ts`) — populate dev DB with realistic state.

Tests never use production data.

---

## Flaky tests

Flaky tests are bugs. When discovered:
1. Quarantine immediately (`.skip` with reason + ticket).
2. Investigate root cause: time, async ordering, network mock, shared state.
3. Fix the test or the code; never just retry.

Goal: zero quarantined tests in `main` for more than a week.

---

## Test reporting

- Coverage reports posted as PR comments.
- Test results in CI summary.
- Slow-test report (top 10) reviewed monthly.

---

## See also

- [Coding Standards](./02-coding-standards.md)
- [CI/CD](../07-infrastructure/04-ci-cd.md)
- [Non-Functional Requirements](../02-requirements/02-non-functional-requirements.md)
