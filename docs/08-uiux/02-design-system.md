# 02 · Design System

**Status:** 🟢 Approved

The Volu design system — tokens, components, motion. Lives as code in `packages/volu_design_system/` (Flutter) and as Figma library.

---

## Brand foundation

**Brand:** Volu — Unbeatable. Exclusive. Now.

**Personality:** confident, soft, inviting, unmistakably current.

**Voice:** clear, warm, occasionally playful. Not corporate, not "fun-and-quirky." Trusted neighbour.

---

## Colour tokens

### Primary palette

| Token | Hex | Use |
|---|---|---|
| `volu.primary.900` | `#0F172A` | Brand primary — deep slate. Used for major surfaces, headings. |
| `volu.primary.800` | `#1E293B` | Hover/pressed for primary surfaces |
| `volu.primary.700` | `#334155` | |
| `volu.primary.600` | `#475569` | |
| `volu.primary.500` | `#64748B` | Body text on light backgrounds |
| `volu.primary.400` | `#94A3B8` | Muted/disabled text |
| `volu.primary.200` | `#E2E8F0` | Borders, dividers |
| `volu.primary.100` | `#F1F5F9` | Subtle backgrounds |
| `volu.primary.50` | `#F8FAFC` | Page backgrounds |

### Accent — orange

| Token | Hex | Use |
|---|---|---|
| `volu.accent.500` | `#F97316` | Primary CTA, brand accent |
| `volu.accent.600` | `#EA580C` | Hover/pressed |
| `volu.accent.100` | `#FED7AA` | Soft accent backgrounds |
| `volu.accent.50` | `#FFF7ED` | |

### Gold (premium / verified)

| Token | Hex | Use |
|---|---|---|
| `volu.gold.500` | `#C8A04F` | Verified badge, premium tier |
| `volu.gold.100` | `#F5E9C7` | |

### Semantic colours

| Token | Hex | Use |
|---|---|---|
| `success.500` | `#16A34A` | Success states (redeemed, paid) |
| `success.50` | `#DCFCE7` | |
| `warning.500` | `#EAB308` | Warning states (expiring soon) |
| `warning.50` | `#FEF9C3` | |
| `danger.500` | `#DC2626` | Errors, destructive actions |
| `danger.50` | `#FEE2E2` | |
| `info.500` | `#0EA5E9` | Informational |
| `info.50` | `#E0F2FE` | |

### Dark mode

Primary inverted: dark backgrounds, light text. Same accent + semantic palette with adjusted lightness for AA contrast.

---

## Typography

### Latin script (English)

**Font:** Inter (variable). Fallback: system sans-serif.

| Token | Size / Line | Weight | Use |
|---|---|---|---|
| `display` | 36 / 44 | 700 | Hero / brand moments |
| `h1` | 28 / 36 | 700 | Page titles |
| `h2` | 22 / 30 | 600 | Section titles |
| `h3` | 18 / 26 | 600 | Subsection / card titles |
| `body-lg` | 17 / 26 | 400 | Featured body text |
| `body` | 15 / 22 | 400 | Default body |
| `body-sm` | 13 / 20 | 400 | Secondary text |
| `caption` | 12 / 16 | 500 | Labels, metadata |
| `numeric-lg` | 28 / 32 | 600 | Prices in hero positions |
| `numeric` | 18 / 24 | 600 | Inline prices |

### Arabic script

**Font:** IBM Plex Sans Arabic. Fallback: system Arabic.

Same scale, slightly larger sizes (Arabic glyphs visually smaller than Latin):

| Token | Size / Line |
|---|---|
| `display-ar` | 38 / 48 |
| `h1-ar` | 30 / 40 |
| `body-ar` | 16 / 26 |

### Numerals

- Latin numerals for **prices**, **codes** (PIN, short code), **distances**, **times** — for clarity across all users.
- Arabic numerals (Eastern Arabic) optional in body text in `ar` locale; default to Latin for consistency.

---

## Spacing scale (4-base)

| Token | Value |
|---|---|
| `space.0` | 0 |
| `space.1` | 4 |
| `space.2` | 8 |
| `space.3` | 12 |
| `space.4` | 16 |
| `space.5` | 20 |
| `space.6` | 24 |
| `space.8` | 32 |
| `space.10` | 40 |
| `space.12` | 48 |
| `space.16` | 64 |
| `space.20` | 80 |

**Defaults:** card padding `space.4`; section gap `space.6`; screen-edge gap `space.4`.

---

## Radius

| Token | Value | Use |
|---|---|---|
| `radius.sm` | 6 | Chips, small buttons |
| `radius.md` | 12 | Buttons, inputs |
| `radius.lg` | 16 | Cards |
| `radius.xl` | 20 | Bottom sheets, large surfaces |
| `radius.round` | 999 | Avatars, circular buttons |

---

## Elevation / shadows

| Token | Use |
|---|---|
| `shadow.0` | None — flat |
| `shadow.1` | Cards at rest (subtle) |
| `shadow.2` | Cards hover |
| `shadow.3` | Modals, bottom sheets |
| `shadow.4` | Floating action buttons |

Light mode uses soft shadows with low opacity; dark mode uses no shadow + subtle border.

---

## Iconography

- Library: **Lucide Icons** (consistent stroke style).
- Stroke width: 1.5 px or 2 px (consistent across the app).
- Direction-aware icons (chevron, back) flip in RTL.
- Custom icons (Volu mark, raffle wheel, QR scanner) drawn in same style.

---

## Components

### Buttons

| Variant | Use |
|---|---|
| `Primary` | Main CTA. Orange filled. |
| `Secondary` | Secondary CTA. Outline + dark text. |
| `Ghost` | Tertiary action. Just text. |
| `Destructive` | Red filled. Confirmation actions. |
| `Success` | Green filled. Confirm-redeem button on cashier flow. |

Sizes: `sm` (32 high), `md` (44), `lg` (56). Always min 44 high on mobile.

States: rest, hover, pressed, focused, loading, disabled.

### Inputs

`VoluTextField` — label, placeholder, helper text, error state, icon prefix/suffix, password masking, character count for limited fields.

`VoluPhoneInput` — country code dropdown (UAE first, common ME countries next, alphabetical).

`VoluPinInput` — for OTP and coupon PIN entry.

`VoluDatePicker` — Gregorian-only for now; Hijri month names shown alongside in `ar` mode.

### Cards

`VoluCard` — base card.

`VoluDealCard` — used in lists. Hero image (16:9), title, merchant name + verified, original price (struck), Volu price, discount %, expiry countdown, distance.

`VoluCouponCard` — used in wallet. Merchant logo, deal title, expiry countdown, large QR code (revealed inside coupon detail), PIN (biometric-gated reveal).

`VoluMerchantCard` — small merchant pill: logo, brand name, verified badge.

### Badges

`VoluBadge` — text or icon-only. Variants:
- Verified (gold)
- Halal-Certified (green leaf)
- Family-Friendly (teal)
- Pet-Friendly
- Tourist-Friendly
- Trending (orange flame)
- New (orange dot)
- Selling Fast (red flame)

### Chips

`VoluChip` — selectable filter chips. Active and inactive states.

### Countdown

`VoluCountdown` — live ticking countdown for flash deals. Shows days/hours/minutes/seconds; switches to "Ending soon" + animation in last hour. Pure widget that re-renders only the changed unit to save battery.

### Price tag

`VoluPriceTag` — composes original (struck), Volu price, % saved. Used on cards and on detail.

### Bottom sheets, dialogs

`VoluBottomSheet` — primary modal pattern on mobile. Drag handle, content, optional CTA at bottom.

`VoluDialog` — used sparingly for confirmations.

### Loading & errors

`VoluSkeleton` — shimmer placeholders shaped like the eventual content.

`VoluEmptyState` — illustration, helpful copy, primary CTA. Every list view defines its empty state.

`VoluErrorState` — icon, "Something went wrong" copy, retry CTA, support link.

### Forms

`VoluForm` — pattern wrapping `react-hook-form` (admin web) or `formz` (Flutter) + design system controls.

### Toasts

`VoluToast` — non-blocking confirmations (success, info, warning, error). Auto-dismiss 4s; manual close.

### Bilingual layout helpers

`VoluPaddingDirectional` — uses `EdgeInsetsDirectional` instead of `EdgeInsets`. Mandatory for any layout that should mirror in RTL.

---

## Motion

### Duration tokens

| Token | Value | Use |
|---|---|---|
| `motion.fast` | 150 ms | Hover, tap feedback |
| `motion.medium` | 250 ms | Sheet open, navigation |
| `motion.slow` | 400 ms | Hero transitions |

### Easing

`motion.standard`: `cubic-bezier(0.2, 0, 0, 1)` — default for most UI motion.
`motion.emphasized`: `cubic-bezier(0.3, 0, 0, 1)` — for entrance animations of the primary content on a screen.
`motion.decelerate`: `cubic-bezier(0, 0, 0, 1)` — for elements arriving from off-screen.

### Patterns

- Bottom sheet: slide up from bottom + fade in over 250ms.
- Page transition (forward): slide in from end (right in LTR, left in RTL) + fade.
- Page transition (back): reverse.
- Item appearing in a list: fade + slight slide up from below.
- Coupon QR reveal (after biometric): scale up + fade in.

### Respect "Reduce Motion"

If user has Reduce Motion enabled in OS settings, animations cross-fade instead of slide; transitions shortened.

---

## App icon & branding

- **Volu mark:** double-O wordmark with circular emphasis on the "OO."
- **App icon:** dark slate background with orange double-O — works at 1024×1024 down to 60×60.
- **Splash:** Volu wordmark on dark background; brief; no animation.
- **Loading state:** orange dot pulse — minimal, fast.

---

## Accessibility tokens

- Minimum body text contrast: 4.5:1 (WCAG AA).
- Large text contrast: 3:1.
- Interactive elements: minimum 3:1 against adjacent colour.
- Focus indicator: visible 2-px outline in `volu.accent.500`.
- Hit area: 44×44pt minimum.

---

## Implementation rules

- **Code is the source of truth.** Figma library mirrors the code; if there's drift, code wins.
- **No magic numbers.** Every spacing, radius, colour value comes from a token.
- **One way to do a thing.** If two components do similar work, consolidate or document why both exist.
- **Theme-aware.** Every component honours the active theme (light/dark) without per-screen overrides.

---

## See also

- [Design Principles](./01-design-principles.md)
- [Accessibility](./03-accessibility.md)
- [Bilingual & RTL](./04-bilingual-rtl.md)
- [Mobile Architecture](../03-architecture/04-mobile-architecture.md)
