# 01 · Design Principles

**Status:** 🟢 Approved

The five principles that guide every Volu design decision. When in doubt, return to these.

---

## 1. Trust is the product

Volu's defining value vs Cobone, Groupon, Entertainer is **trust**. Every screen should make the user feel in control and informed. Two specific commitments:

- **No hidden cost, ever.** The total payable is shown upfront, including VAT. Surprise top-ups are never tolerated.
- **No friction at redemption.** The most painful moment for our users on competing platforms is the awkward conversation at the merchant counter. Volu's redemption flow eliminates that.

**Design implications:**

- All-in price always visible.
- Verified Merchant badge prominent.
- Clear refund policy on every coupon.
- Live chat support never more than 2 taps away.
- Cashier-side scanner that gives a clear pass/fail in < 1.5 seconds.

---

## 2. Clarity over cleverness

Volu is used by everyone from a first-time tourist to a third-time-this-week regular. Designs must be immediately understandable without onboarding tutorials.

- Plain words ("redeem" not "claim", "expires" not "valid until").
- Numbers and prices, not percentages alone.
- Full sentences in alerts and modals — never just an icon.
- Real photos, not lifestyle stock imagery.
- Empty states that say what to do next, not just "Nothing here."

If a designer needs a tooltip to explain a control, the control needs redesigning.

---

## 3. Bilingual by default, RTL with parity

Volu ships in English and Arabic on day one. **Arabic is not a translation; it's a parallel design.** This means:

- RTL layout is pixel-equivalent to LTR. No half-mirrored views.
- Native font choice for Arabic (IBM Plex Arabic) — not a Latin font with Arabic glyphs forced in.
- Numerals: Arabic uses Eastern Arabic numerals where culturally appropriate; Latin numerals for prices and codes for clarity.
- Direction-aware icons (chevron, back arrow) flipped automatically.
- Layouts tested in both languages every release; long-string truncation never acceptable.

**Implication:** every component in the design system must be designed and tested in both directions.

---

## 4. Mobile-first, touch-first, thumb-first

The user app and merchant app are mobile-first. Every interaction must be thumb-reachable, glanceable, and forgiving.

- **Thumb zone.** Primary actions sit in the bottom third of the screen.
- **Tap targets.** Minimum 44×44pt iOS / 48×48dp Android. Never a 24px-square button.
- **Forgiving.** Every destructive action confirmable; every confirmable action undoable for at least 3 seconds.
- **Glanceable.** Scan a deal card in 0.5 seconds: image, price, expiry, distance.
- **Reach pattern.** Long lists have a back-to-top FAB after scrolling 5+ screens.

The merchant cashier flow is the extreme case: a user with one hand free, in a hurry, with a queue forming. Optimise for this.

---

## 5. Performance is design

A slow UI is a bad UI. We treat performance the same way we treat colour or typography: a design constraint to respect.

Targets:

- **Cold start:** ≤ 2.5s on iPhone 12 / Galaxy S21.
- **Frame rate:** 60 fps minimum across all flows.
- **API responses:** P95 ≤ 500ms.
- **Image load:** progressive blurhash → low-res → full-res.

Design implications:

- No dense lists without virtualisation.
- No animations that drop below 30 fps on target devices.
- Skeleton screens replace spinners for content-heavy views.
- Optimistic UI updates with smart rollback for low-friction interactions.

---

## How we apply these

Every design review asks five questions:

1. Does this **build trust** or erode it? (Hidden cost? Unclear copy? Cluttered T&Cs?)
2. Is it **clear** to a first-time user without explanation?
3. Does it work in **Arabic, RTL** with the same quality as English?
4. Is every interactive element **thumb-reachable** with adequate tap targets?
5. Will it **render in under 16ms** on a typical mid-range Android phone?

If the answer to any of these is no, the design isn't shipped.

---

## See also

- [Design System](./02-design-system.md)
- [Accessibility](./03-accessibility.md)
- [Bilingual & RTL](./04-bilingual-rtl.md)
- [Critical Flows](./05-critical-flows.md)
