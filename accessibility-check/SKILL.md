---
name: accessibility-check
description: Runs a comprehensive digital accessibility conformance check for a project's web frontend and/or mobile app against WCAG 2.1/2.2 (target Level AA), semantic markup and correct ARIA usage, full keyboard operability, screen-reader/assistive-technology compatibility, platform accessibility APIs (iOS VoiceOver, Android TalkBack, React Native, Flutter Semantics), and applicability of accessibility-specific EU/national law — the European Accessibility Act (EAA, Directive (EU) 2019/882) and Germany's Barrierefreiheitsstärkungsgesetz (BFSG) — then reports the results as one table (check, area, status, evidence, recommendation). Use this whenever the user asks for an "accessibility check", "accessibility audit", "a11y check", "a11y audit", "WCAG check", "WCAG conformance", "WCAG audit", "barrierefreiheitsprüfung", "barrierefreiheit check", "barrierefrei prüfen", "ist das barrierefrei", "BITV check", "BFSG check", "BFSG-konform", "BFSG readiness", "European Accessibility Act", "EAA readiness", "EAA conformance", "screenreader test", "screen reader test", "voiceover test", "talkback test", "keyboard navigation check", "tastaturbedienung prüfen", "tastaturbedienbarkeit", "color contrast check", "kontrastprüfung", "farbkontrast prüfen", "alt text audit", "alt-text prüfen", "ARIA audit", "accessibility conformance statement", "Erklärung zur Barrierefreiheit", or asks about any specific item this covers (missing alt text, focus indicators, tab order, form labels, heading hierarchy, landmark regions, touch target size, Dynamic Type, font scaling, reduced motion) — even if they only name one or two items and not "accessibility check" explicitly. Also trigger before a production/store launch or public-sector deployment when the user asks "are we accessible enough to launch" or similar readiness questions.
---

# Accessibility Check

A structured, evidence-based check of a project's web frontend and/or mobile app against WCAG
2.1/2.2 conformance (target Level AA), platform assistive-technology APIs, and EU/national
accessibility law — not a substitute for testing with real assistive technology by real disabled
users, and not a legal conformance audit. It investigates the actual markup, component code, and
styling, and reports one table the user can act on.

## Ground rules

- **Evidence or it didn't happen.** Every row needs a concrete pointer — a `file:line`, the actual
  DOM/JSX/XML markup you found, an actual computed contrast ratio (real hex values run through the
  WCAG contrast formula, not "looks like enough contrast"), or an explicit note that this needs a
  human/AT pass. Don't mark something ✅ because a component library is generally known to be
  accessible, or because a pattern "looks accessible" — either you found the attribute/markup/ratio
  in the code, or you didn't.
- **This is static code/markup review, not a live assistive-technology pass.** You can check what
  is checkable from source: presence and correctness of `alt`/`aria-*`/`accessibilityLabel`
  attributes, semantic element choice, heading structure, computed color values against contrast
  thresholds, keyboard event handlers, focus-management code, and platform API usage. You cannot
  fully verify how a screen reader actually announces a screen, how VoiceOver/TalkBack focus order
  actually feels in practice, or whether a keyboard-only user can genuinely complete a task without
  frustration — a real screen-reader and keyboard-only pass by a human, ideally including disabled
  users, is the gold standard this review cannot replace. Say this once, up front, so the user
  doesn't over-trust the result.
- **The EAA/BFSG applicability and conformance-statement checks are legal/organizational facts, not
  code facts.** Code review can surface evidence (an accessibility statement page, a documented
  conformance target) or its absence — it cannot decide a legal scope question with certainty. Be
  honest about which kind of check you're doing, per row, using the status legend below.
- **Don't invent scope you can't check, and don't silently drop scope either.** If the project has
  no mobile app, skip only the mobile-specific rows and mark them `➖ N/A` with a one-line reason
  (e.g. "no mobile app in this project — web only"). Every check in the applicable reference
  file(s) gets a row in the output; none are quietly skipped.
- **Be genuinely thorough.** Don't stop after the easy checks (alt text, contrast) and skip the
  ones that take real digging (focus order in a custom widget, ARIA state updates, VoiceOver
  grouping) — those are usually exactly the ones that break real usage for people who rely on
  assistive technology.

## Workflow

1. **Identify the frontend stack.** Determine the web framework (React, Vue, plain HTML/CSS,
   server-rendered, a component library like MUI/Bootstrap/Ant), and separately whether there is a
   native or cross-platform mobile app (iOS/Swift, Android/Kotlin-Java, React Native, Flutter). This
   determines which reference file(s) apply — most projects are web-only, some are mobile-only,
   some are both and get rows from both files.
2. **Work through the applicable reference file(s)** — every check in each one gets a row in the
   final table, marked ✅/❌/⚠️/➖ as appropriate:
   - `references/web-accessibility.md` — WCAG 2.1/2.2 AA checks organized by the four POUR
     principles (Perceivable, Operable, Understandable, Robust)
   - `references/mobile-accessibility.md` — iOS/Android/cross-platform accessibility API usage,
     mobile-specific touch-target and system-setting checks, and the EAA/BFSG applicability
     assessment (relevant for web projects too — apply its applicability section regardless of
     whether there's a mobile app)
3. **Investigate, don't assume.** Grep and read the actual markup/component code — search for
   `<img`, `onClick`/`onPress` on non-interactive elements, `outline: none`, `tabindex`,
   `aria-*`, `contentDescription`, `accessibilityLabel`, font-size/color declarations — rather than
   assuming a component library or design system is "probably accessible" out of the box. Pull the
   actual computed hex values for text/background pairs and UI-component colors and run the real
   WCAG contrast ratio formula against them (4.5:1 for normal text, 3:1 for large text and
   graphical/UI-component boundaries) instead of eyeballing a Figma swatch or assuming a brand
   palette is fine. A library being accessible by default doesn't mean this project didn't override
   that default with custom styling or a custom wrapper component.
4. **Report as one table**, most severe failures first within each area, followed by a prioritized
   punch list and a needs-review list.

## Output format

Start with 3-4 sentences: which stack(s) were identified (web/mobile/both), which reference
file(s) were worked through in full, what couldn't be reached (no real device/screen reader to
test with, no access to a design system's full token list, no visibility into store-listing/legal
documents), and the caveat from Ground rules — this is static review, not a substitute for real
assistive-technology testing or a legal conformance audit.

Then ALWAYS use this exact table — one row per check across all applicable reference files, none
omitted:

| # | Check | Bereich/Area | Status | Befund/Evidence | Empfehlung/Recommendation |
|---|---|---|---|---|---|
| 1 | ... | Web/Mobile/EU-Recht | ✅/❌/⚠️/➖ | file:line, markup excerpt, or computed contrast ratio | only if not ✅ |

(Match the table's actual language to the conversation's language — the column names above are
illustrative, not a fixed vocabulary. Keep "Befund"/"Evidence" concrete: a path+line with the
actual markup snippet, a computed color pair and its ratio, or "not found — searched X, Y, Z".
Group rows by area/theme — POUR principle, platform, or EU-law — with a subheading or an
"Area" column, whichever reads more clearly for the number of rows involved.)

Status legend:
- ✅ Pass — the accessible markup/attribute/API usage was found and verified in code
- ❌ Fail — checked, the required markup/attribute/behavior is missing or broken
- ⚠️ Needs manual/AT review — code can't fully answer this (real VoiceOver/TalkBack behavior,
  a subjective usability judgment, a legal scope call, an existing signed conformance statement's
  accuracy)
- ➖ N/A — this check's surface doesn't apply here (no mobile app, no video content, no forms) —
  state why in one clause; for the EAA/BFSG applicability checks specifically, the *reasoning* for
  why something is or isn't in scope is the actual deliverable, not just the verdict

End with a **prioritized punch list**: every ❌, ordered so failures that block screen-reader or
keyboard-only users entirely come first, followed by lower-severity/cosmetic issues, each with the
one-line fix. Follow it with a **needs-review list**: every ⚠️, since those are exactly the items a
human — often someone doing an actual screen-reader/keyboard pass, sometimes a lawyer for the
EAA/BFSG scope calls — needs to close out.
