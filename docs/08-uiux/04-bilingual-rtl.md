# 04 · Bilingual & RTL

**Status:** 🟢 Approved

How Volu does English + Arabic right.

---

## Principles

1. **Arabic is a parallel design, not a translation.** Every screen designed in EN is also designed in AR — checked together at review.
2. **RTL is full RTL.** Layouts mirror; icons mirror; gestures mirror; metaphors mirror.
3. **Native fonts only.** Arabic uses an Arabic-typographic-system font, not a Latin font with Arabic glyphs.
4. **Test in both languages every release.** Long Arabic strings are common; layouts must accommodate them.

---

## Translation workflow

### Source of truth

`packages/volu_localization/lib/l10n/`:

- `intl_en.arb`
- `intl_ar.arb`

Strings are keyed; values are the localised text. Designers + product write the EN; native Arabic reviewers translate the AR.

### Adding a new string

1. Engineer adds key to `intl_en.arb` with EN value and a description (context for translator).
2. PR auto-flags Arabic translation needed (CI fails if `intl_ar.arb` missing the key).
3. Translator adds AR value.
4. CI re-runs; PR can merge.

### Pluralisation

Arabic has six plural forms. Use the ICU MessageFormat:

```
"deal_count": "{count, plural, =0 {No deals} =1 {1 deal} few {{count} deals} many {{count} deals} other {{count} deals}}"
```

The Flutter `intl` package handles this natively.

### Gender

Some strings benefit from gendered forms in Arabic. We use neutral phrasing where possible. Where unavoidable, use ICU `select`:

```
"welcome": "{gender, select, female {مرحبًا بك} other {مرحبًا}}"
```

### Variable interpolation

`"price_label": "AED {amount}"` — never embed a translatable phrase inside a variable.

---

## Layout rules

### Direction-aware spacing

Use `EdgeInsetsDirectional` (Flutter) / logical properties (CSS):

```dart
// ✅ correct — mirrors automatically
padding: EdgeInsetsDirectional.only(start: 16, end: 8)

// ❌ wrong — never mirrors
padding: EdgeInsets.only(left: 16, right: 8)
```

Same in CSS:

```css
/* ✅ correct */
margin-inline-start: 16px;

/* ❌ wrong */
margin-left: 16px;
```

### Direction-aware alignment

```dart
Alignment.centerStart  // ✅ flips with direction
Alignment.centerLeft   // ❌ doesn't flip
```

### Direction-aware icons

Icons that imply direction (chevron, arrow) flip in RTL:

```dart
Icon(Directionality.of(context) == TextDirection.rtl ? Icons.chevron_left : Icons.chevron_right)
```

Or use icons that are direction-agnostic (×, ⋮, …).

### Layout testing

Every screen has a golden test in both EN and AR. Drift is caught immediately.

---

## Typography

### Latin: Inter

Sizes optimised for Latin glyphs: 13/15/17/22/28/36.

### Arabic: IBM Plex Sans Arabic

Arabic glyphs visually smaller; sizes shifted up by ~10%:

```
Latin body:    15px / 22 line height
Arabic body:   16px / 26 line height
```

Both font families are checked into the apps (or loaded from a CDN with subset).

### Mixed content

A line containing both Latin and Arabic characters (e.g., "AED 150 خصم 50%") renders correctly with Flutter's bidirectional text engine. Numbers stay LTR, Arabic flows RTL, mixed renders without manual intervention.

### Numerals

Default to **Latin numerals** for prices, codes, distances. Reasons:

- Unambiguous to all UAE residents (Latin numerals are universal in the region).
- Consistent across locales (price reads the same in both apps).
- Simpler typesetting.

Eastern Arabic numerals (٠١٢٣٤٥٦٧٨٩) optionally rendered for narrative body text in AR mode if a native Arabic-speaker review prefers them. Toggleable per locale via a simple flag.

---

## Common pitfalls (avoided)

| Pitfall                                | Why bad                                 | Fix                             |
| -------------------------------------- | --------------------------------------- | ------------------------------- |
| English-only "left/right" instructions | Confusing in RTL                        | Use "next/previous"             |
| Hardcoded text in image assets         | Can't translate                         | Always render text in code      |
| Using Latin font for Arabic glyphs     | Looks bad / illegible                   | Use Arabic-specific font        |
| Mixed-script forced to one direction   | Bidi engine handles it; manual is wrong | Trust the platform              |
| Date formatted as MM/DD/YYYY           | Ambiguous globally                      | Use `intl` to format per locale |
| Padding using LTR-only sides           | Doesn't mirror                          | EdgeInsetsDirectional           |
| Forms with labels not flipping         | Confusing                               | Label sides flip with direction |

---

## Specific UI patterns

### Forms

In RTL, labels are on the right; inputs flow right-to-left. Validation messages appear in the same direction as the input.

### Lists & scrolls

Vertical scroll: same in both directions.
Horizontal scroll (e.g., flash deal carousel): flows RTL in Arabic. The "next" item is to the **left** in RTL, vs **right** in LTR.

### Navigation

Bottom nav: order flips. The home tab stays at the trailing position relative to start. We test both layouts.

Back button:

- iOS Arabic: back chevron points right.
- Android Arabic: back arrow points right.

### Forms — phone field

Phone numbers always render LTR (numbers don't have Arabic vs English direction). The label is direction-appropriate; the input itself uses LTR for the digits.

### Receipts and invoices

Invoices generated in either EN, AR, or both. Bilingual invoices have EN on left column, AR on right column.

### Notifications

Push notifications, emails, SMS sent in user's preferred locale. Test for proper rendering of Arabic in iOS push (uses LTR by default — must include Bidi marks where needed).

---

## Locale management

User's locale stored in:

- `users.locale` (server-side preference)
- `shared_preferences` `locale_override` (client-side override)
- iOS: `Locale.preferredLanguages.first` (system)
- Android: `Locale.getDefault()` (system)

Resolution order: client override → server preference → system → English fallback.

User can change in Settings; reflects immediately.

---

## QA per release

- [ ] Switch to AR; navigate every primary flow.
- [ ] Switch back to EN; verify nothing is stuck in AR.
- [ ] Long Arabic strings: confirm no truncation in titles, buttons, list rows.
- [ ] Mixed content: deal title with Arabic + brand name with Latin renders correctly.
- [ ] Numbers in form fields stay LTR.
- [ ] Push notifications received in current locale.
- [ ] Email/SMS in current locale.
- [ ] Wallet pass localised.
- [ ] Receipts localised.

---

## Native Arabic reviewer

Volu retains at least one Khaleeji-Arabic native speaker who reviews:

- Translations before each release.
- Tone and idiom (Arabic varies regionally; UAE Arabic isn't Egyptian Arabic).
- Marketing copy.

---

## Specific Volu phrases (locked translations)

| EN                          | AR                      | Notes                                        |
| --------------------------- | ----------------------- | -------------------------------------------- |
| Volu                        | فولو                    | Brand — same pronunciation in both languages |
| Unbeatable. Exclusive. Now. | لا تُضاهى. حصرية. الآن. | Tagline — locked                             |
| Verified Merchant           | تاجر موثَّق             |                                              |
| Today's Volu drops          | عروض فولو اليوم         |                                              |
| Add to wallet               | أضف إلى المحفظة         |                                              |
| Redeem                      | استبدل                  |                                              |
| Coupon                      | كوبون                   |                                              |
| Raffle                      | السحب                   |                                              |
| Refund                      | استرداد                 |                                              |
| Buy now                     | اشتر الآن               |                                              |
| Confirm                     | تأكيد                   |                                              |

Maintained in `/packages/volu_localization/lib/l10n/locked_phrases.md`.

---

## See also

- [Design Principles](./01-design-principles.md)
- [Design System](./02-design-system.md)
- [Mobile Architecture](../03-architecture/04-mobile-architecture.md)
