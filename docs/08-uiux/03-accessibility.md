# 03 · Accessibility

**Status:** 🟢 Approved

WCAG 2.1 AA is our floor, not our ceiling. Volu is for everyone — including users with low vision, motor impairment, hearing impairment, cognitive disabilities, and users on aging devices.

---

## Standards

- **Target:** WCAG 2.1 Level AA across all consumer surfaces.
- **Stretch:** AAA where reasonable on critical flows (purchase, redemption).
- **Platform standards:** Apple Human Interface Guidelines + Material Design accessibility for native idioms.

---

## Visual

### Contrast

- Body text: ≥ 4.5:1 against background.
- Large text (18pt+): ≥ 3:1.
- Interactive elements + focus indicators: ≥ 3:1 against adjacent colors.
- Verified badges, important indicators: 4.5:1.

Tested in light + dark themes.

### Colour is not the only signal

- Discounts shown with both colour and text ("50% off") — never colour alone.
- Errors shown with icon + colour + text.
- Success states: green + ✓ + word.
- Status indicators always have text.

### Text size

- Default body 15px (mobile) / 16px (web).
- All text scales with OS font size preference (Dynamic Type on iOS, font scale on Android).
- Layout tested at 1.0× through 1.5× without truncation or overlap.
- No text in images for content that must be read.

### Spacing

- Line height ≥ 1.5× font size.
- Paragraph spacing ≥ 2× font size.
- Touch targets ≥ 44×44pt iOS / 48×48dp Android.

---

## Motor

### Tap targets

- 44×44pt minimum on iOS; 48×48dp on Android.
- 8pt+ spacing between adjacent targets to prevent mis-taps.
- Critical actions (Buy, Redeem) get larger targets (≥ 56pt).

### No time-limited inputs

- OTPs valid 5 minutes (announced).
- Cart sessions don't auto-clear.
- Forms don't auto-submit on idle.

### Forgiving destructive actions

- Confirmations for irreversible actions (account deletion, refund).
- Undo affordances where possible (e.g., "Removed from favourites — undo" toast).

### Drag-free design

- No essential interaction requires drag-only gestures.
- All swipe actions have a tappable equivalent.
- Pull-to-refresh has a refresh button alternative.

---

## Screen readers

### Mobile

- iOS VoiceOver tested every release.
- Android TalkBack tested every release.
- Every interactive element has a `Semantics` label.
- Labels describe purpose, not just visual ("Buy Mediterranean Brunch for AED 195" not "Tap me").
- State announced ("Loading," "Loaded," "Selected," "Disabled").
- Headings use `Semantics(header: true)` for navigation.
- Lists use list semantics for item-of-N announcements.

### Web (admin)

- Semantic HTML (`<nav>`, `<main>`, `<button>`, etc.).
- ARIA labels where semantic HTML insufficient.
- Live regions (`aria-live`) for dynamic updates (toasts, KPI tiles refreshing).
- Skip-to-content link at top.

### Specific Volu patterns

- **Coupon card** announces: "Coupon for [deal title], expires [date], redeemable at [merchant]. Tap to open."
- **Deal card** announces: "[Deal title] by [merchant]. Original price [X]. Volu price [Y]. [Z]% saved. [N] km away."
- **QR reveal**: "QR code displayed. Show to cashier."
- **Countdown** announces remaining time on focus, not on every tick (avoid spam).

---

## Hearing

- Sounds are not the only signal:
  - QR scan success: ✓ banner + haptic + sound (sound optional).
  - Notifications: text + icon.
- Live chat support is text-based (Intercom).
- Future video / audio: captions and transcripts mandatory.

---

## Cognitive

### Plain language

- Reading age target: 12 (WCAG AAA target).
- Short sentences. Active voice. Direct verbs.
- No jargon ("idempotent," "tokenised," "synchronously").
- Acronyms expanded on first use ("Buy Now Pay Later (BNPL)").

### Predictable interactions

- Same controls in the same places across screens.
- Navigation patterns consistent.
- Confirmations match user expectations (no "Yes" that means "No").

### Error prevention & recovery

- Validate inline, not just on submit.
- Errors say what went wrong AND how to fix it.
- Suggest corrections where possible (e.g., typo in promo code — show similar codes).
- Don't blame the user.

### Memory aids

- Wallet shows expiry prominently.
- Reminder pushes at 7d / 3d / 24h / 3h.
- Recently viewed deals saved.
- Multi-step flows show progress (e.g., "Step 2 of 4 — Bank details").

---

## Internationalisation accessibility

- Bilingual EN/AR is itself an accessibility feature for Arabic-speaking users.
- RTL is true RTL, not "EN with text reversed."
- Numerals and prices in Latin numerals for unambiguous reading regardless of locale.
- Formatting respects locale (dates, times, currency).

---

## Testing

### Automated

- **axe-core** runs on every admin page in CI.
- Flutter `flutter test --update-goldens` includes accessibility-tree assertions.
- Linter rules forbid `Image` without `semanticLabel` and `IconButton` without `tooltip`.

### Manual

Per release:

- VoiceOver pass on iOS (key flows: signup, purchase, redeem, raffle).
- TalkBack pass on Android.
- Keyboard-only navigation pass on admin web (no mouse).
- Zoom test: at 200% browser zoom, every page remains usable.
- Screen reader user testing: invite at least one VoiceOver / TalkBack user per quarter for usability sessions.

### Devices in the test pool

- iOS: latest + 2 generations back; iPhone SE (small screen).
- Android: Pixel + Samsung; one device aged 3+ years; one budget device with 3GB RAM.
- Both platforms: high-contrast mode, reduce-motion enabled, 1.5× text size.

---

## What we don't ship

- Pages with auto-playing video or audio.
- Animations exceeding 5 seconds without a pause control.
- Flashing content > 3 times per second.
- Text-as-image for content that must be readable.
- Modals that trap focus without a close button.
- Forms without labels.

---

## Common patterns

### Form field with error

```
[Label]
[Input field]
⚠ [Error message — what's wrong + how to fix]
```

The error has `aria-describedby` on the input pointing to the error text. The error appears on blur, not on every keystroke.

### Loading state

Skeleton screens with `aria-busy="true"` on the parent. After load, focus moves to first interactive element if appropriate.

### Toast

Non-blocking, dismissible, auto-removed after 4 seconds. Has `role="status"` for non-critical, `role="alert"` for errors. Doesn't steal focus.

### Modal / bottom sheet

Focus trapped inside. ESC dismisses (web). Down-swipe dismisses (mobile). Focus returns to triggering element on dismiss.

---

## Documentation per component

Every design-system component's documentation states:

- Required accessible name source (label, aria-label, semanticLabel)
- Keyboard interactions (web)
- Screen-reader announcement pattern
- Touch-target compliance
- Contrast verification

---

## Reporting accessibility issues

Users can report accessibility issues via:

- In-app: Settings → Help → Report a problem.
- Email: a11y@volu.ae.

These tickets get priority routing.

---

## See also

- [Design Principles](./01-design-principles.md)
- [Design System](./02-design-system.md)
- [Bilingual & RTL](./04-bilingual-rtl.md)
