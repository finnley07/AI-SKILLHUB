# Mobile accessibility and EU/national law applicability

## iOS (skip with `➖ N/A — no iOS app` if not applicable)

**I1 — accessibilityLabel/Hint/Traits on custom controls.** Native controls (`UIButton`,
`UILabel`) get basic accessibility for free; custom `UIView`/`UIControl` subclasses and any
SwiftUI view used as a tappable control do not. Grep for custom view/button implementations and
check: UIKit needs `isAccessibilityElement = true`, a real `accessibilityLabel` (describing
purpose, e.g. "Add to cart", not "icon_plus"), `accessibilityTraits` set appropriately (`.button`,
`.header`, `.selected` for toggled state), and `accessibilityHint` only where the label alone
doesn't convey the outcome. SwiftUI: `.accessibilityLabel("...")`, `.accessibilityHint("...")`,
`.accessibilityAddTraits(.isButton)` on custom-drawn tappable views (a `ZStack`/`Rectangle` with a
`.onTapGesture` is invisible to VoiceOver without these).

**I2 — Dynamic Type support.** Grep for hard-coded font sizes (`.font(.system(size: 14))`,
`UIFont.systemFont(ofSize: 14)`) versus scalable text styles (`.font(.body)`,
`UIFont.preferredFont(forTextStyle: .body)`). Fixed-size fonts don't grow when the user increases
their preferred text size in Settings — a WCAG-equivalent failure for iOS users who rely on larger
text. Check `UILabel.adjustsFontForContentSizeCategory = true` is set alongside `preferredFont`
usage, and that layout constraints allow text containers to grow rather than truncating/clipping at
larger sizes.

**I3 — VoiceOver focus order.** Where a screen's visual layout doesn't match natural view-tree
order (overlapping views, absolutely-positioned elements, custom layout containers), check whether
`accessibilityElements` is explicitly set on the container to define the intended VoiceOver reading
order, and whether related controls are grouped via `accessibilityElement(children: .combine)` /
`isAccessibilityElement` on a container so VoiceOver doesn't announce every sub-label separately
for what should read as one item (e.g. a product card's image + title + price + "Add" button).

## Android (skip with `➖ N/A — no Android app` if not applicable)

**A1 — contentDescription on custom views and ImageButtons/ImageViews.** Grep layout XML and
Compose code for `ImageButton`, `ImageView`, and `Image`/`Icon` (Compose) with no
`contentDescription` (XML) / no `contentDescription` parameter (Compose). A genuinely decorative
image should explicitly set `android:contentDescription="@null"` (or `contentDescription = null` in
Compose) — that's the intentional "skip me" signal, distinct from simply omitting the attribute,
which TalkBack may announce using the resource filename instead.

**A2 — TalkBack focus order and importantForAccessibility.** Check `android:importantForAccessibility`
usage (`"no"` for decorative wrapper views that shouldn't be a separate stop, `"yes"` forced onto
custom composite views that should be one stop rather than several), and — where visual layout order
diverges from XML/Compose tree order — `accessibilityTraversalBefore`/`accessibilityTraversalAfter`
(XML) to define the intended reading order explicitly rather than leaving it to chance.

**A3 — Font scale / large-text support.** Grep text-size declarations for `sp` versus `dp` units.
Text sizes must use `sp` (scale-independent pixels, which respect the user's system font-size
setting); `dp` on a text size is a common bug that silently ignores the accessibility font-scale
setting. Also check that layouts don't clip/truncate at larger scale factors (test mentally against
200% font scale, per Android's own guidance).

## Cross-platform frameworks (check whichever the project actually uses)

**X1 — React Native.** Grep `Pressable`/`TouchableOpacity`/`TouchableHighlight`/custom tappable
components for `onPress` handlers with no matching `accessible={true}`, `accessibilityLabel`,
`accessibilityRole` (`"button"`, `"link"`, `"header"`, etc.), and — for stateful controls —
`accessibilityState={{ selected, checked, disabled }}` kept in sync with actual component state.
These map to the underlying iOS/Android APIs above, so a missing prop here is a real gap on both
platforms at once, not a cosmetic omission.

**X2 — Flutter.** Grep custom-painted or gesture-driven widgets (`GestureDetector`, `InkWell`
wrapping non-semantic content, `CustomPaint`) for a wrapping `Semantics` widget providing `label`,
`button: true`/appropriate flag, and `value`/`hint` where relevant. Check `Image`/`Icon` widgets for
`semanticLabel`, and confirm `ExcludeSemantics` is used deliberately (for genuinely redundant
decorative content) rather than accidentally hiding real content from TalkBack/VoiceOver.

## Touch targets and contrast (mobile)

**M-T1 — Minimum touch target size per platform.** iOS Human Interface Guidelines recommend at
least 44×44pt; Android Material Design recommends at least 48×48dp. Check actual rendered
tappable area (including padding/hit-slop, not just the visible icon) for icon-only buttons, list
row actions, and closely-spaced controls.

**M-T2 — Color contrast.** Same WCAG thresholds as web (4.5:1 normal text / 3:1 large text and UI
components) apply to native and cross-platform app UI too — pull the actual color constants/theme
tokens used for text and compute the real ratio rather than assuming the design system's palette
was validated.

## System accessibility settings

**M-S1 — Reduced motion respected.** Check whether animations (transitions, auto-playing
parallax/carousel effects) read the platform's reduced-motion setting and simplify/disable
themselves accordingly: iOS `UIAccessibility.isReduceMotionEnabled` (UIKit) /
`accessibilityReduceMotion` environment value (SwiftUI); Android has no single universal flag but
respects `Settings.Global.ANIMATOR_DURATION_SCALE` at the system level — check custom in-app
animations (not system-driven ones) don't ignore it; React Native: `AccessibilityInfo.isReduceMotionEnabled()`.

**M-S2 — Reduced transparency / increased contrast respected.** iOS
`UIAccessibility.isReduceTransparencyEnabled` and `isDarkerSystemColorsEnabled` — check that
blur/translucency-heavy UI (frosted-glass navigation bars, translucent cards) has a solid-background
fallback when these are enabled, rather than becoming a low-contrast mess.

## EAA / BFSG applicability assessment — assess, don't assume

These have specific scope tests, and both a mobile app and a plain website can be in scope — this
section applies whether or not the project has a native mobile app. Most internal/B2B tools and
early-stage consumer apps fall outside part or all of this; the useful check is confirming that with
real reasoning, not silently assuming either way. Report each as `➖ N/A (with reason)` or
`⚠️ needs legal assessment` rather than skipping the row — a documented "doesn't apply because X" is
itself a valid, reportable finding, and the *reasoning* is the actual deliverable, not just the
verdict.

**B1 — European Accessibility Act (EAA, Directive (EU) 2019/882) sector scope.** The EAA covers
specific products (computers and operating systems, ATMs/ticketing/check-in self-service terminals,
smartphones and other consumer computing hardware, TV equipment and services, e-readers) and
specific services (electronic communications services, services providing access to audiovisual
media, air/bus/rail/waterborne passenger transport services including their websites/apps/
e-ticketing, banking and financial services, e-books, e-commerce). Check: does this project's
domain fall into one of the *service* categories (an e-commerce checkout, a banking/fintech app, a
transport-ticketing flow, an e-book reader) — that is the category almost all software projects
would fall under, since the hardware categories rarely apply to a codebase being reviewed here. If
the project is, e.g., an internal admin tool, a B2B SaaS dashboard with no consumer-facing sales
flow, or a product/sector genuinely outside the list, say so with that reasoning and mark `➖ N/A`.
If it's consumer-facing e-commerce, banking, transport ticketing, e-communications, or an
audiovisual-media/e-book service, it is in scope — proceed to B2 and B3.

**B2 — Microenterprise exemption (EAA Art. 4(5), transposed in BFSG §3).** The EAA's *service*
obligations (not product obligations) don't apply to microenterprises — fewer than 10 employees
and either annual turnover or annual balance sheet total below €2 million. Check the operating
company's actual size (headcount, turnover) if known/documented; if genuinely a microenterprise
providing only a service (not manufacturing hardware), that is a real, reportable exemption —
`➖ N/A (microenterprise service exemption)`. If size is unknown or the company has scaled past the
threshold, mark `⚠️ needs assessment` rather than assuming the exemption still holds — this changes
as a company grows and is exactly the kind of fact code review can't confirm on its own.

**B3 — Germany's Barrierefreiheitsstärkungsgesetz (BFSG) — transposition and enforcement.** If the
project is deployed for the German market (or the operating entity is based in Germany), the BFSG
is the national law implementing the EAA, in force for economic operators since 28 June 2025
(existing self-service terminals placed in service before that date may continue in use until the
end of their economic lifetime, capped at 20 years — check whether any relevant hardware/kiosk
predates the cutoff if that's part of the project). Technical requirements are detailed in the
BFSG's implementing regulation (BFSGV), which references the harmonized standard **EN 301 549** —
conformance with EN 301 549 (which in turn incorporates WCAG 2.1 AA for web/software content) is
the practical target, which is exactly what `web-accessibility.md`'s checks assess. Note the
distinction from **BITV 2.0**: BITV applies to German *public-sector* bodies' websites/apps
regardless of the EAA's private-sector sector test — if this project is a public-sector site
(a Behörde, a public university, a state-run service), it is in scope for accessibility
requirements independent of the B1 sector test, and that should be reported as its own `⚠️`/`✅`
row rather than folded into the EAA reasoning.

**B4 — Conformance statement / accessibility declaration in practice.** If B1 concluded the project
is in scope (and B2 didn't exempt it), check what a conformance requirement means practically for a
*service* (as opposed to a hardware product, which would need an EU declaration of conformity and
CE marking — rarely relevant to a software codebase): the service provider must make information
available (typically in general terms and conditions, or a dedicated accessibility statement page)
explaining how the service meets the applicable accessibility requirements, and must provide an
accessible feedback/complaint mechanism for users to report barriers. Check whether such a page/
statement exists in the project (grep for an "Barrierefreiheit"/"Accessibility Statement" page or
route) — if in scope but no such statement exists, that's a `❌`; if out of scope per B1/B2, mark
`➖ N/A` referencing that reasoning instead of evaluating this row independently.
