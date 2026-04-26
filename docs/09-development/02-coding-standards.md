# 02 · Coding Standards

**Status:** 🟢 Approved

The standards every Volu engineer follows. Less about taste, more about consistency that lets a team move fast without breaking each other.

---

## Universal principles

1. **Code is read 10× more than it's written.** Optimise for clarity, not cleverness.
2. **Boring is good.** Boring patterns are easy to onboard, easy to debug, easy to refactor.
3. **Make the hard thing the right thing.** Lint rules, types, and CI gates make the right path the easy path.
4. **Reviewability over independence.** Small, focused PRs always.
5. **Tests are documentation.** A reader should learn how a thing works from its tests.

---

## TypeScript (backend + admin)

### Settings

```jsonc
// tsconfig.json — strict mode mandatory
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "skipLibCheck": false
  }
}
```

### Style

- 2-space indent; LF line endings; UTF-8.
- Prettier formats; no manual formatting decisions.
- ESLint with `typescript-eslint/strict` ruleset.
- Named exports preferred over default exports (better refactor + import).
- File names: `kebab-case.ts`. One class/main export per file.
- Function names: `camelCase`. Class names: `PascalCase`. Constants: `SCREAMING_SNAKE_CASE`.
- Acronyms in camelCase: `parseUrl` not `parseURL`; `apiClient` not `APIClient`.

### Types

- **No `any`.** If a type is truly unknown, use `unknown` and narrow.
- **No `as any`** assertions in business logic. Acceptable only at narrow IO boundaries with comments.
- **Prefer narrow types.** `'pending' | 'paid' | 'refunded'` beats `string`.
- **Discriminated unions** for variants:

  ```ts
  type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
  ```

- **Branded types** for IDs to prevent mix-ups:

  ```ts
  type UserId = string & { __brand: 'UserId' };
  type MerchantId = string & { __brand: 'MerchantId' };
  ```

### Async

- `async/await` always; never raw `.then`.
- `Promise.all([...])` for parallel awaits; never sequential awaits when independent.
- Error handling at the boundary, not buried in helpers.

### Imports

- Absolute paths via `tsconfig` paths (`@/modules/...`); not deep relative `../../../..`.
- Group imports: stdlib, third-party, internal — separated by blank lines.
- No circular imports. (ESLint rule.)

### Comments

- Comments explain **why**, not what.
- TODOs include name + ticket: `// TODO(hasnat, VOLU-1234): handle X`.
- No commented-out code committed; remove and rely on git.

### Patterns

- **Prefer composition over inheritance.** NestJS DI makes composition trivial.
- **Pure functions where possible.** Easier to test.
- **Domain entities own their behaviour.** Don't have anemic models with logic in services if it belongs to the entity.
- **Result types over exceptions for expected failures.** Exceptions for truly exceptional, unrecoverable cases.

### Specific to NestJS

- Controllers thin: validate, call service, return DTO.
- Services do business logic, never see HTTP.
- Repositories wrap Prisma; no Prisma calls outside repositories.
- Modules expose a clear public API via `index.ts`.
- Decorators (`@Roles`, `@CurrentUser`, `@Idempotent`) for cross-cutting concerns.

---

## Dart (Flutter mobile)

### Style

- `flutter_lints` + project additions.
- `dart format` enforced.
- 2-space indent.
- File names: `snake_case.dart`. Class: `PascalCase`. Variables: `camelCase`.
- `const` constructors wherever possible (huge perf wins).
- `final` for variables that don't reassign.
- `late` only when truly necessary.

### Types

- Sound null safety always.
- No `dynamic` in business logic.
- Prefer typed records over Map<String, dynamic>.

### Patterns

- **Riverpod** for state; no `setState` in feature screens.
- **Freezed** for data classes and unions.
- **GoRouter** for navigation; no manual `Navigator.push`.
- **Dio** for HTTP via the generated API client; never raw `http`.
- **Pure widgets** wherever possible; logic in providers/notifiers.

### Layout & UI rules

- Always use `EdgeInsetsDirectional` (not `EdgeInsets.fromLTRB`).
- Use `AlignmentDirectional`.
- Use design system components (`VoluButton`, `VoluTextField`) — never raw Material/Cupertino in feature code.
- Translations via `context.t.<key>`; never hardcoded strings.

### Performance

- `ListView.builder` for any list > 10 items.
- `RepaintBoundary` around heavy widgets that update independently.
- `const` widgets to avoid rebuilds.
- Profile in release/profile mode before declaring a perf win.

---

## SQL / Prisma

- Prefer Prisma's typed query API.
- Drop to raw SQL (`$queryRaw`) for: complex aggregations, performance-critical paths, features Prisma doesn't support well (window functions, complex CTEs, specific Postgres extensions).
- Always parameterised; never string concatenation.
- Migrations are SQL — clear, commented, reviewed by schema reviewer.
- Schema fields use `snake_case` at DB level; Prisma maps to `camelCase` in code via `@map` and `@@map`.

---

## Naming conventions across the stack

| Concept | Backend (TS) | Mobile (Dart) | DB |
|---|---|---|---|
| User ID | `userId: UserId` | `userId: UserId` | `user_id UUID` |
| Boolean flag | `isVerified` | `isVerified` | `is_verified` |
| Timestamps | `createdAt: Date` | `createdAt: DateTime` | `created_at TIMESTAMPTZ` |
| Money | `amountFils: number` | `amountFils: int` | `amount_fils INTEGER` |
| Enums | `OrderStatus.Paid` | `OrderStatus.paid` | `order_status` enum value `'paid'` |

Prisma `@@map` and `@map` bridge the snake_case ↔ camelCase divide.

---

## Error handling

### Backend

Three error classes (see [Backend Architecture](../03-architecture/03-backend-architecture.md#error-handling)):

- `DomainError` — business rule violation, 4xx.
- `InfrastructureError` — external dep failure, 5xx, retryable.
- Unknown — global filter, 500, Sentry.

Custom error classes per module:

```ts
export class CouponExpiredError extends DomainError {
  readonly code = 'COUPON_EXPIRED';
  readonly httpStatus = 410;

  constructor(public readonly expiresAt: Date) {
    super('This coupon has expired.');
  }
}
```

### Mobile

`Result<T, AppError>` from API calls. UI consumes:

```dart
state.when(
  data: (data) => SuccessUi(data),
  loading: () => SkeletonUi(),
  error: (e, _) => ErrorUi(e),
);
```

Specific error UI per error type (network, auth, not-found, generic).

---

## Logging

### Backend (Pino)

```ts
this.logger.info({ couponId, userId, branchId }, 'coupon.redeemed');
```

Always structured. Always include relevant IDs. Never log PII (Pino redaction enforces).

### Mobile

Logging only in debug. Sentry breadcrumbs for production diagnostics. Crashes auto-captured by Sentry.

---

## Comments & docs

- Public exports have JSDoc / Dartdoc:

  ```ts
  /**
   * Issue coupons for a paid order.
   *
   * Idempotent: calling twice for the same order returns the existing coupons.
   *
   * @throws {OrderNotPaidError} if the order is not in `paid` state.
   */
  async issueForOrder(orderId: OrderId, tx?: Transaction): Promise<Coupon[]>;
  ```

- Internal helpers don't need docs; clear names suffice.
- Complex logic gets a brief explanation of *why* this approach (with link to ticket / ADR if relevant).

---

## Git commits

- **Conventional Commits** (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `test:`).
- Subject ≤ 72 chars; imperative mood ("Add coupon expiry job", not "Added").
- Body explains the why if non-obvious.
- Ref issue ID: `feat(coupons): add expiry worker [VOLU-1234]`.

---

## Review etiquette

- Reviewer's job: spot bugs, suggest improvements, learn the code.
- Author's job: small focused PR, descriptive title, screenshots/recording for UI.
- Disagreements: ask, don't demand. Trust your reviewer; trust your reviewee.
- Use `nit:` prefix for minor stylistic suggestions; `?` for questions; nothing for required changes.
- Don't merge your own PR without an approval (even with permissions).
- Block merge only for material issues; small things go in follow-up PRs.

---

## Reusable patterns library

Lives in repo READMEs:

- "How to add a new endpoint" — checklist with file changes.
- "How to add a domain event" — producer + subscriber template.
- "How to write a Prisma migration" — common patterns (column add/drop, etc.).
- "How to add a Flutter screen" — feature template.

These reduce mental load and keep newcomers productive on day three.

---

## See also

- [Monorepo Structure](./01-monorepo-structure.md)
- [Testing Strategy](./03-testing-strategy.md)
- [Git Workflow](./04-git-workflow.md)
