# 04 · Mobile Architecture

**Status:** 🟢 Approved

The detailed design of the Volu mobile apps. Both apps share architecture, design system, and core packages — they differ in features.

---

## Monorepo layout (Melos)

```
volu-mobile/
├── melos.yaml                    # workspace config
├── apps/
│   ├── user_app/                 # consumer app
│   │   ├── lib/
│   │   ├── ios/
│   │   ├── android/
│   │   └── pubspec.yaml
│   └── merchant_app/             # cashier + owner app
│       ├── lib/
│       ├── ios/
│       ├── android/
│       └── pubspec.yaml
│
├── packages/
│   ├── volu_core/                # shared models, utilities
│   ├── volu_design_system/       # tokens, components, themes
│   ├── volu_api_client/          # generated from OpenAPI spec
│   ├── volu_auth/                # auth flows shared between apps
│   ├── volu_payments/            # Stripe, Tabby, Tamara integration
│   ├── volu_localization/        # EN + AR translations
│   ├── volu_analytics/           # PostHog, Mixpanel wrappers
│   └── volu_diagnostics/         # logging, error reporting (Sentry)
│
└── tools/
    ├── codegen/                  # API client codegen scripts
    └── icon_assets/              # app icons + launch images
```

Melos commands:

- `melos bootstrap` — link all packages
- `melos run analyze` — flutter analyze across all packages
- `melos run test` — flutter test across all packages
- `melos run codegen` — regenerate API client + freezed + json_serializable
- `melos run build:user:android` — build user app android release
- (etc.)

---

## App architecture pattern

Each app follows the same pattern — a **feature-first** layout with Riverpod for state and clean separation of presentation from business logic.

```
lib/
├── main.dart                     # bootstrap (env, error handling, providers)
├── app.dart                      # MaterialApp.router with theme + locale
│
├── core/
│   ├── env/                      # build-time env (dev/staging/prod)
│   ├── routing/                  # GoRouter config
│   ├── error/                    # error handling, user-facing copy
│   ├── network/                  # Dio setup, interceptors, retry
│   ├── di/                       # provider overrides for tests
│   └── extensions/
│
├── features/
│   ├── auth/
│   │   ├── data/                 # data sources, repositories impl
│   │   ├── domain/               # entities, repository interfaces
│   │   ├── application/          # state notifiers / use cases
│   │   └── presentation/         # screens, widgets
│   │
│   ├── home/
│   ├── discovery/                # category + search + map
│   ├── deal_detail/
│   ├── cart_checkout/
│   ├── wallet/                   # "My Volus"
│   ├── redemption/               # (merchant app only) scanner
│   ├── raffles/
│   ├── profile/
│   ├── reviews/
│   └── notifications/
│
└── shared/
    ├── widgets/                  # cross-feature widgets
    ├── animations/
    └── modals/
```

---

## State management with Riverpod

Three notifier patterns:

### 1. `Provider` — for simple read-only values

```dart
final apiClientProvider = Provider<VoluApiClient>((ref) {
  return VoluApiClient(baseUrl: ref.read(envProvider).apiBaseUrl);
});
```

### 2. `AsyncNotifier` — for async-loaded state

```dart
@riverpod
class HomeFeed extends _$HomeFeed {
  @override
  Future<HomeFeedData> build() async {
    final api = ref.read(apiClientProvider);
    final location = await ref.watch(currentLocationProvider.future);
    return api.fetchHomeFeed(near: location);
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => build());
  }
}
```

### 3. `Notifier` — for client-only state machines (e.g., cart)

```dart
@riverpod
class Cart extends _$Cart {
  @override
  CartState build() => CartState.empty();

  void addItem(Deal deal, int quantity) { /* ... */ }
  void removeItem(String dealId) { /* ... */ }
  void applyPromoCode(String code) { /* ... */ }
}
```

### Rules

- **State is always immutable** (use `freezed`).
- **Side effects in state notifiers, not widgets.**
- **Widgets are dumb**: they read state and dispatch events. No business logic in widgets.
- **Never use `setState` in feature screens.** Reserved for transient widget-local UI state (e.g., a TextField focus state).

---

## Navigation with GoRouter

All routes declarative in `core/routing/router.dart`:

```dart
final goRouterProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/home',
    redirect: (context, state) {
      final auth = ref.read(authNotifierProvider);
      final isLoggingIn = state.matchedLocation == '/login';
      if (!auth.isAuthenticated && !isLoggingIn) return '/login';
      if (auth.isAuthenticated && isLoggingIn) return '/home';
      return null;
    },
    routes: [
      ShellRoute(
        builder: (_, __, child) => HomeShell(child: child),
        routes: [
          GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
          GoRoute(path: '/discover', builder: (_, __) => const DiscoverScreen()),
          GoRoute(path: '/wallet', builder: (_, __) => const WalletScreen()),
          GoRoute(path: '/raffles', builder: (_, __) => const RafflesScreen()),
          GoRoute(path: '/profile', builder: (_, __) => const ProfileScreen()),
        ],
      ),
      GoRoute(
        path: '/deal/:id',
        builder: (_, state) => DealDetailScreen(dealId: state.pathParameters['id']!),
      ),
      // ...
    ],
  );
});
```

**Deep linking:** every meaningful page has a stable URL. Push notifications and shared links open directly to the right screen.

**Deferred deep linking:** ad clicks attribute through deferred deep linking (Branch.io or App Links + iOS Universal Links).

---

## API client (generated)

The OpenAPI spec from the backend is the source of truth. We use `openapi-generator-cli` to generate a typed Dart client:

```bash
melos run codegen:api
```

Generates `packages/volu_api_client/lib/`:

```dart
final api = ref.read(apiClientProvider);
final result = await api.deals.fetch(dealId: '...');
// result is typed: Deal
```

Manual hand-written code over a generated client is forbidden. Custom logic goes in repositories that wrap the client.

---

## Local storage strategy

| Data                                      | Storage                                | Reason                                                          |
| ----------------------------------------- | -------------------------------------- | --------------------------------------------------------------- |
| Access token (short-lived JWT)            | In-memory only                         | No persistence needed; refresh from refresh-token on cold start |
| Refresh token                             | `flutter_secure_storage` (OS keychain) | Encrypted at rest by OS                                         |
| User profile basics (for offline display) | Hive                                   | Fast read                                                       |
| Cart state                                | Hive                                   | Persists across app restarts                                    |
| Cached deals (offline browse)             | Hive (with TTL)                        | Browsing works in low-connectivity                              |
| Search history                            | `shared_preferences`                   | Non-sensitive                                                   |
| Settings (lang, notifications)            | `shared_preferences`                   | Non-sensitive                                                   |
| Coupon redemption queue (merchant app)    | Hive (encrypted)                       | Critical reliability                                            |
| App PIN (separate from coupon PIN)        | `flutter_secure_storage`               | Sensitive                                                       |

**Never** stored locally: full payment cards, Emirates ID details, full addresses (these stay server-side).

---

## Authentication flow

```
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│  User App    │        │ Identity API │        │  Token Store │
└──────┬───────┘        └──────┬───────┘        └──────┬───────┘
       │                        │                        │
       │  Phone +971xxxx        │                        │
       │───────────────────────>│                        │
       │                        │  Send OTP via Unifonic │
       │                        │                        │
       │  OTP 123456            │                        │
       │───────────────────────>│                        │
       │                        │  Validate              │
       │                        │  Issue access + refresh│
       │  Tokens                │                        │
       │<───────────────────────│                        │
       │                        │                        │
       │  Store refresh in keychain                      │
       │────────────────────────────────────────────────>│
       │  Hold access in memory                          │
       │                        │                        │
       │  Authorized requests with Bearer access         │
       │───────────────────────>│                        │
       │                        │                        │
       │  401 (access expired)  │                        │
       │<───────────────────────│                        │
       │                        │                        │
       │  Refresh with refresh token                     │
       │───────────────────────>│                        │
       │  New access + new refresh                       │
       │<───────────────────────│                        │
       │                        │                        │
       │  Retry original request│                        │
```

`Dio` interceptor handles 401 → refresh → retry transparently. If refresh also fails, the user is signed out gracefully.

---

## Offline support strategy

### User app

- **Browse offline**: home feed cached for 30 minutes; deal detail cached for 5 minutes; user can browse without network until cache expires.
- **Wallet offline**: all active coupons cached locally with QR + PIN; user can show coupon to merchant even if user is offline (merchant app validates).
- **Cart offline**: cart state in Hive; survives app restart.
- **Purchase offline**: blocked. Show "Check connection" banner.

### Merchant app

- **Scanner offline**: critical capability. Up to 4 hours offline.
- Local cryptographic validation of QR signature using public key shipped in app.
- Local cache of redeemed coupons today (so we can detect double-spend on the same device).
- Offline redemptions queued to Hive with cryptographic signature using device key.
- On reconnect: queue drained to server. Server is single source of truth — if server already has a redemption for that coupon (someone else redeemed first while we were offline), the queued redemption is rejected and merchant is shown a clear message.

---

## Bilingual & RTL

### Translation

All user-facing strings live in `packages/volu_localization/`:

```yaml
# en.arb
"home_title": "Today's Volu drops"

# ar.arb
"home_title": "عروض فولو اليوم"
```

Strings are looked up via `BuildContext` extension:

```dart
Text(context.t.home_title)
```

### Locale switching

Locale set in `app.dart`:

```dart
MaterialApp.router(
  supportedLocales: [Locale('en'), Locale('ar')],
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  locale: ref.watch(localeProvider),
  // ...
)
```

User toggles in settings; persisted to shared_preferences.

### RTL

Flutter handles RTL automatically when `Directionality` is correct. We:

- Use `Directionality.of(context)` to read direction.
- Use `Padding(padding: EdgeInsetsDirectional.only(start: 16, end: 8))` instead of `left/right`.
- Mirror icons that have direction (chevron, back arrow): handled by `Transform` + RTL check, or use direction-aware icons.

### Date / number / currency formatting

- `intl` package for localised formats.
- Currency: AED with proper symbol and grouping.
- Dates: Gregorian primary; Hijri shown in AR mode where culturally relevant.

---

## Theme & design system

### Tokens

Defined in `packages/volu_design_system/`:

```dart
class VoluColors {
  static const primary = Color(0xFF0F172A);
  static const accent = Color(0xFFF97316);
  static const success = Color(0xFF16A34A);
  static const danger = Color(0xFFDC2626);
  // ... grayscale ramp, semantic colors
}

class VoluRadius {
  static const sm = 6.0;
  static const md = 12.0;
  static const lg = 20.0;
  static const round = 999.0;
}

class VoluSpacing {
  static const xs = 4.0;
  static const sm = 8.0;
  static const md = 16.0;
  static const lg = 24.0;
  static const xl = 32.0;
}

class VoluTypography {
  // ... text styles using Volu's chosen font (Inter Latin, IBM Plex Arabic for AR)
}
```

### Components

Reusable components in `packages/volu_design_system/lib/components/`:

- `VoluButton` (primary, secondary, ghost, destructive)
- `VoluTextField` (with built-in validation states)
- `VoluCard`
- `VoluBadge` (verified, halal, family-friendly, etc.)
- `VoluChip`
- `VoluCountdown` (live ticking countdown for flash deals)
- `VoluPriceTag` (original + Volu price + % off)
- `VoluCouponCard` (with QR + PIN reveal)
- `VoluEmptyState`
- `VoluErrorState`
- `VoluLoadingSkeleton`
- `VoluBottomSheet`
- `VoluAlertDialog`

Every component:

- Has a Storybook-style preview in `packages/volu_design_system/example/`.
- Has unit tests (golden tests for visual regression).
- Supports both light and dark themes.
- Supports both EN and AR.

---

## Critical screen: the redemption scanner

This is the merchant app's most-used and most-critical screen. Specific design rules:

1. **One-tap entry**: home screen has an enormous "Scan Volu" button (75% of screen).
2. **Camera opens instantly**: pre-warm camera permission on app start; defer initialisation only if denied.
3. **Auto-detect QR**: as soon as a valid QR enters frame, capture; no "tap to scan" needed.
4. **Haptic on detection**: subtle haptic confirms scan.
5. **Voice prompt**: "Ask the customer for their PIN."
6. **Numeric keypad**: large, high-contrast numpad for PIN entry; entered digits visible (cashier needs to confirm).
7. **One-tap confirm**: large green "Redeem" button after PIN entered.
8. **Result is huge**: full-screen success or failure with sound + haptic.
9. **Manual fallback**: prominent "Enter code instead" link if camera fails.
10. **Offline indicator**: small persistent banner when offline, never blocking.

---

## Performance optimisations

### Cold start

- Defer non-critical initialisation (analytics, marketing SDKs) to after first frame.
- Pre-cache only what's needed for the home screen.
- Use `RawImage` + cached decode for known assets.

### Scrolling

- `ListView.builder` for any list > 10 items.
- `cacheExtent` tuned to reduce visible jank.
- Hero images use progressive loading with blurhash placeholders.
- Pagination via `infinite_scroll_pagination` package.

### Image loading

- `cached_network_image` with disk cache.
- Cloudflare Images delivers WebP/AVIF based on Accept header.
- Multiple sizes per image; correct size requested per layout context.

### Build modes

- **Debug**: full debug overlay, hot reload, no obfuscation.
- **Profile**: production-like with profiler attached; for performance testing.
- **Release**: tree-shaken, obfuscated, R8 (Android), bitcode disabled (iOS post-Xcode 14).

### Bundle size targets

- User app: ≤ 35 MB on iOS (App Store IPA), ≤ 25 MB Android (AAB base).
- Merchant app: similar.

### Frame budget

- All screens must render at 60 fps minimum on iPhone 12 / Galaxy S21.
- No animation that drops below 30 fps.
- Profile every release; regressions are bugs.

---

## Testing

### Unit tests

For pure logic in `application/` and `domain/`:

```dart
test('coupon expires when valid_until is in the past', () { ... });
```

### Widget tests

For individual widgets and screens:

```dart
testWidgets('home screen shows flash deals carousel', (tester) async {
  await tester.pumpWidget(...);
  expect(find.byType(VoluCountdown), findsWidgets);
});
```

### Golden tests

For design-system components and key screens (visual regression):

```dart
testGoldens('VoluCouponCard light + AR', (tester) async {
  await tester.pumpWidgetBuilder(...);
  await screenMatchesGolden(tester, 'coupon_card_light_ar');
});
```

### Integration tests

Real backend (test environment) + real Flutter app:

```dart
testWidgets('full purchase flow', (tester) async {
  await loginAs(tester, testUser);
  await tester.tap(find.text('Buy now'));
  // ...
});
```

### E2E

Patrol or Maestro for full device-level flows on CI (iOS sim + Android emulator).

### Coverage targets

- Domain logic: 90%+
- Application layer: 80%+
- Widgets: 60%+ (golden tests count)

---

## Build & release

### Android

- Build: `flutter build appbundle --release --flavor prod`
- Sign: Play App Signing (Google manages key)
- Distribute: Internal testing → Closed beta → Production

### iOS

- Build: `flutter build ipa --release --flavor prod`
- Sign: App Store Connect with team certificate
- Distribute: TestFlight Internal → TestFlight External → App Store

### Versioning

- `pubspec.yaml`: `version: <semver>+<build>` e.g., `1.4.2+47`
- Build number monotonically increasing per platform (set by CI from `GITHUB_RUN_NUMBER`).

### Release cadence

- Production release every 2 weeks (staying within app store review windows).
- Hotfixes via Shorebird OTA only when truly urgent.

---

## Crash reporting & analytics

### Sentry

Initialised in `main.dart`:

```dart
await SentryFlutter.init(
  (options) {
    options.dsn = env.sentryDsn;
    options.tracesSampleRate = 0.1;
    options.environment = env.name;
    options.release = '${packageInfo.version}+${packageInfo.buildNumber}';
  },
  appRunner: () => runApp(...),
);
```

### Analytics events

Event names follow `<noun>_<verb>` pattern: `deal_viewed`, `coupon_purchased`, `coupon_redeemed`, `raffle_entered`.

Properties always include: `app_version`, `build_number`, `locale`, `platform`, `is_authenticated`.

PII is never sent as event properties. User IDs are hashed before going to PostHog/Mixpanel.

---

## Accessibility

- Every interactive element has a `Semantics` label.
- Focus order is correct (tested with VoiceOver / TalkBack).
- Tap targets ≥ 44×44pt iOS / 48×48dp Android.
- Color contrast meets WCAG 2.1 AA (4.5:1 for body, 3:1 for large text).
- Text scales with user's OS font preference.
- Animations can be disabled by user (respects "Reduce Motion").

See [Accessibility](../08-uiux/03-accessibility.md) for full standard.

---

## See also

- [Backend Architecture](./03-backend-architecture.md)
- [Design System](../08-uiux/02-design-system.md)
- [Bilingual & RTL](../08-uiux/04-bilingual-rtl.md)
- [Coding Standards](../09-development/02-coding-standards.md)
