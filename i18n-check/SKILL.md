---
name: i18n-check
description: Runs a comprehensive internationalization (i18n) and localization (l10n) readiness check across a project's web frontend, backend, and mobile app, then reports the results as one table (check, area, status, evidence, recommendation). Covers string externalization vs. hardcoded UI/backend text, translation completeness across all supported locales, correct pluralization (ICU MessageFormat/gettext ngettext/framework-native vs. naive English-only concatenation), sentence construction via string concatenation that breaks word order across languages, locale-aware date/time/number/currency/unit formatting vs. hardcoded formats, text-expansion and layout resilience for longer/shorter/differently-scripted translations, RTL (right-to-left) layout support for Arabic/Hebrew/Farsi/Urdu locales, character encoding (UTF-8 end-to-end) and font-fallback coverage for non-Latin scripts, locale detection/switching/persistence and regional-variant fallback chains (e.g. de-CH -> de -> default locale), translation-file/source-key sync and CI extraction/lint coverage (orphaned keys, missing base-locale keys), and locale-specific conventions (measurement units, first-day-of-week, address/phone-number format). Use this whenever the user asks for an "i18n check", "i18n audit", "i18n readiness", "internationalization check", "internationalization audit", "internationalization readiness", "l10n check", "l10n audit", "l10n review", "localization check", "localization audit", "localization readiness", "translation check", "translation audit", "translation completeness check", "multilanguage check", "multi-language check", "multilingual check", "locale check", "locale audit", "pluralization check", "plural forms check", "RTL check", "right-to-left check", "hardcoded strings check", "internationalisierung prüfen", "internationalisierungscheck", "i18n-check", "lokalisierung prüfen", "lokalisierungscheck", "mehrsprachigkeit prüfen", "mehrsprachigkeitscheck", "übersetzungsprüfung", "übersetzung vollständig prüfen", "übersetzungslücken prüfen", "sprachumschaltung prüfen", "lokalisierungsbereitschaft", or asks about any specific item this covers (hardcoded UI strings, missing translation keys, plural forms, date/number/currency formatting, text truncation or layout breakage in translated UI, RTL layout, mirrored icons, tofu/boxes for non-Latin fonts, locale fallback chain, orphaned translation keys, translation sync in CI) — even if they only name one or two items and not "i18n check" explicitly. Also trigger before an international/multi-market launch when the user asks "are we ready for international markets", "ready to localize", or similar readiness questions.
---

# Internationalization (i18n) & Localization (l10n) Check

A structured, evidence-based check of a project's web frontend, backend, and mobile app against
internationalization and localization readiness best practice — not a substitute for an actual
linguistic/QA pass by native speakers of each target locale, and not a substitute for genuine
market-appropriateness review by people familiar with each target culture. It investigates the
actual source code, translation resource files, and configuration, and reports one table the user
can act on.

## Ground rules

- **Evidence or it didn't happen.** Every row needs a concrete pointer — a `file:line` with the
  actual offending string, the actual translation file contents (key counts, a diff between
  locale files, an actually-missing key), or the actual date/number-formatting code. Don't mark
  something ✅ because the project "looks internationalized" or uses a well-known i18n library —
  either you found the translation-key usage / locale-aware call / fallback logic in the code, or
  you didn't. Libraries that support i18n correctly can still be used incorrectly (hardcoded
  strings sitting right next to `t()` calls, dates formatted with `toLocaleDateString()` in one
  place and a hardcoded `MM/DD/YYYY` template literal two files over).
- **This is static source/config review, not a linguistic QA pass and not a rendered-UI visual
  check.** You can check what's checkable from source: whether a string is pulled from a
  translation key or hardcoded, whether all locale files have the same key set, whether a
  pluralization API is used, whether formatting calls are locale-aware, whether RTL-safe CSS
  properties are used, whether a fallback chain exists in code/config. You cannot verify that a
  translation is linguistically correct or natural, that a translated layout doesn't visually
  overflow at runtime, or that a right-to-left layout actually reads correctly to a native
  reader — flag those as needing a human/native-speaker or visual QA pass rather than guessing.
- **Boundary with `accessibility-check`: don't re-derive its territory.** Whether the page/app
  correctly sets the `lang`/`xml:lang` attribute per locale and whether a screen reader announces
  language changes correctly is covered by `accessibility-check`'s
  `references/web-accessibility.md` Understandable section — that's the right skill for it. Here,
  note it as a one-line pointer ("see `accessibility-check` for `lang` attribute correctness") if
  relevant, don't duplicate the check or give it its own ✅/❌ row in this skill's table.
- **Cultural/market-appropriateness is a human judgment call, not a code-reviewable fact.**
  Whether an icon, color, image, or idiom carries an unintended or offensive connotation in a
  specific target market is real i18n/l10n risk, but it cannot be verified by reading source code
  — flag it explicitly as `⚠️ needs local-market review` when the product genuinely targets
  multiple culturally distinct markets, rather than silently omitting it or guessing an answer.
- **Don't invent scope you can't check, and don't silently drop scope either.** If the project has
  no i18n/l10n framework at all and ships hardcoded strings in a single locale by design, say so
  up front and mark the framework-specific checks (translation completeness, pluralization APIs,
  fallback chains, translation-sync tooling) `➖ N/A` with that one-line reason — string
  externalization and layout-resilience checks may still be worth reporting as
  forward-looking recommendations, but say plainly that this is a single-locale product today.
  Likewise, if no RTL locale is in the supported-locale list, mark the RTL checks `➖ N/A` rather
  than inventing a finding. Every check in this skill gets a row in the output; none are quietly
  skipped.
- **Be genuinely thorough.** Don't stop after the easy checks (grep for obvious hardcoded strings)
  and skip the ones that take real digging (tracing a concatenated sentence across three
  variables, diffing every locale file's key set, checking whether a regional-variant fallback is
  actually wired up in code vs. just documented) — those are usually exactly the ones that break
  real usage for real non-English/non-source-locale users.

## Workflow

1. **Identify the i18n/l10n framework in use.** Look for `react-intl`, `i18next`/`react-i18next`,
   FormatJS, `vue-i18n`, `gettext`/`ngettext` (Python/PHP/Ruby), .NET `.resx` resource files,
   Android `strings.xml`/`plurals.xml`, iOS `.strings`/`.stringsdict`/`.xcstrings`, Flutter
   `intl`/`.arb` files, or conclude "none — hardcoded strings only" if no framework or translation
   mechanism is present anywhere in the repo. Check `package.json`/`pubspec.yaml`/`*.csproj`/
   `requirements.txt`/`Gemfile` for the dependency, and confirm it's actually wired into the
   app (imported and called), not just installed.
2. **Locate translation resource files and the supported-locale list.** Find the actual locale
   files (`locales/*.json`, `src/i18n/*/*.json`, `.resx` per culture, `Localizable.strings` per
   `.lproj`, `values-*/strings.xml`, `.arb` per locale, `.po`/`.mo` files) and the list of locales
   the product actually declares as supported (a locale config, a language switcher's option
   list, App Store/Play Store listed languages, `<html lang>` alternates) — this list drives
   which locale-specific checks (RTL, plural-form count, regional fallback) actually apply.
3. **Work through the checklist below**, investigating each with Grep/Read/Bash rather than
   inferring from the framework's reputation — search for the actual string literals, diff the
   actual locale files, read the actual formatting call sites. A framework that supports correct
   pluralization doesn't mean every plural string in this codebase uses it instead of naive
   concatenation.
4. **Report as one table**, most severe/most user-visible failures first within each area,
   followed by a prioritized punch list and a needs-review list.

## String Externalization

**I1 — Hardcoded user-facing strings outside the translation mechanism.** Grep UI code
(components, views, templates) and backend response/error-message code for string literals that
look user-facing — capitalized sentence-like text, strings containing punctuation or spaces,
JSX/template text nodes — that are NOT passed through the project's translation function/key
lookup (`t(...)`, `useTranslation`, `<FormattedMessage>`, `gettext(...)`, `resource lookup`,
`NSLocalizedString`, `getString(R.string....)`). Useful patterns: JSX text between tags not
wrapped in a translation component; `<button>Submit</button>` vs. `<button>{t('submit')}</button>`;
backend `return Error("Invalid email address")` vs. a translated/keyed error response; a
hardcoded string passed to a UI label constructor instead of a resource ID. Distinguish
genuinely-internal strings (log messages, internal admin tool text, code comments) from
user-facing ones — only the latter are findings. Report each with `file:line` and the actual
literal found.

## Translation Completeness

**I2 — Missing keys across supported locales.** For each locale file, compare its key set against
the base/reference locale (usually the source language, e.g. `en.json`) — a key present in the
base locale but absent from another locale's file causes a runtime fallback to the key name itself
or to default-locale text, which is a real user-visible defect, not a cosmetic one. Count keys per
locale file (`jq 'keys | length' locales/*.json`, or equivalent for `.resx`/`.strings`/`.arb`/`.po`)
and flag any locale whose count is meaningfully lower than the base, then actually diff the key
sets to name the specific missing keys (or a representative sample if the gap is large) rather
than reporting only the count delta.

## Pluralization

**I3 — Plural-rule handling.** Check every place a count-dependent message is rendered
(`{count} item(s)`, "You have N new messages") for a proper plural-rule mechanism — ICU
MessageFormat `{count, plural, one {...} other {...}}`, gettext `ngettext(singular, plural,
count)`, Android `plurals.xml` with `<item quantity="...">`, iOS `.stringsdict`, or the
i18n library's native pluralization API — rather than naive concatenation
(`` `${count} item${count !== 1 ? 's' : ''}` ``) or a hardcoded English two-form ternary
(`count === 1 ? 'item' : 'items'`). The naive forms only work for English's two plural forms
(one/other); they silently produce grammatically wrong text for languages with more forms (Polish
has four: one/few/many/other; Arabic has six: zero/one/two/few/many/other) or that don't inflect
on plurals in the same pattern at all (Japanese/Chinese have no grammatical plural, so a plural
mechanism should render the same string for every count while a broken naive implementation may
still needlessly branch). Grep for count interpolation near string literals
(`count`, `.length`, `Count`) combined with a following `s`/`(s)`/ternary as the tell.

## Sentence Construction

**I4 — Full sentences assembled from concatenated translated fragments.** Flag any user-facing
message built by concatenating multiple translated pieces around a variable —
`` t('youHave') + ' ' + count + ' ' + t('newMessages') `` or `"You have " + t('newMessages') +
count` — rather than a single interpolated template per complete sentence (`t('youHaveNewMessages',
{count})` resolving to one full translated string with `{count}` placed correctly for that
language's word order). Word order, required articles/prepositions, and even which word needs to
agree in gender/number with the count differ across languages; assembling a sentence from
independently-translated fragments in source-language order breaks grammatically in most target
languages even when each individual fragment is translated correctly. Grep for
`t(...) + ` / `+ t(...)` / template literals that interleave a translation call with a raw variable
inside a sentence-shaped string.

## Locale-Aware Formatting

**I5 — Dates & times.** Check date/time rendering uses a locale-aware formatter —
`Intl.DateTimeFormat`, a framework's locale-aware date-format helper (`date-fns` with a locale
object, `dayjs.locale(...)`), or a platform-native formatter (`NSDateFormatter` with the current
locale, Android `DateFormat.getDateInstance(...)`, .NET `CultureInfo`-aware `ToString`) — rather
than a hardcoded format string baked into a template (`` `${month}/${day}/${year}` ``, a fixed
`"MM/dd/yyyy"` pattern applied regardless of locale). A hardcoded `MM/DD/YYYY` is read as
day/month by most of the world and is a genuine ambiguity/defect, not just a stylistic
preference.

**I6 — Numbers, currency & units.** Same check for numeric output: thousands/decimal separators
(`1,234.56` vs. `1.234,56`), currency symbols and their position, and unit display should go
through `Intl.NumberFormat`/a locale-aware currency formatter or platform equivalent, not a
hardcoded `$` prefix, hardcoded comma-grouping, or manual string formatting
(`` `$${amount.toFixed(2)}` ``) applied unconditionally regardless of locale.

## Text Expansion & Layout Resilience

**I7 — Fixed-size or truncating UI for translated text.** Check UI containers, buttons, and labels
that hold translated strings for hardcoded fixed pixel widths, `white-space: nowrap` without a
wrap/scroll fallback, or hardcoded character-count truncation — these assume the source-language
string length. Translated text is commonly 30-40% longer in German/Finnish/Russian and other
languages, needs more vertical line-height for CJK even though CJK is narrower per character, and
Arabic/Hebrew reshape and mirror. Grep component styles for fixed `width`/`max-width` on elements
known to contain translated labels, `nowrap` without `text-overflow: ellipsis` and an accessible
way to see the full text, and hardcoded `substring(0, N)`-style truncation of translated strings
in code rather than CSS-level ellipsis. Flag any button/label sized to fit exactly the
English/source string with no accommodation for longer text.

## RTL (Right-to-Left) Support

**I8 — RTL layout readiness.** If Arabic, Hebrew, Farsi, Urdu, or another RTL locale is in the
supported-locale list (per step 2 of the Workflow), check whether layout is actually RTL-aware:
logical CSS properties (`margin-inline-start`/`padding-inline-end`/`inset-inline-start`) instead of
physical, hardcoded-direction properties (`margin-left`, `padding-right`, `float: left`); an
explicit `dir="rtl"` mechanism (HTML `dir` attribute driven by locale, a CSS-in-JS RTL plugin,
platform `layoutDirection` in mobile) that actually flips the layout rather than only flipping
text alignment; and mirrored directional iconography (back/forward chevrons, arrows) where
direction is semantically meaningful. A locale being selectable in a language switcher while the
underlying layout stays physically LTR (text flows right-to-left but buttons, icons, and reading
order don't) is "nominal" RTL support and should be reported as a failure, not a pass — verify by
reading the actual CSS/stylesheet properties used, not by trusting that the RTL locale is merely
listed as supported. No RTL locale in the supported-locale list → `➖ N/A`, state which locales
were checked.

## Character Encoding & Font Coverage

**I9 — UTF-8 consistency end-to-end.** Check that UTF-8 is used consistently: source file
encoding, HTTP `Content-Type` charset and HTML `<meta charset>`, database/column collation
(e.g. MySQL `utf8mb4` vs. legacy `latin1` or `utf8` (3-byte, which truncates some emoji/CJK
extension characters) — `utf8mb4` is the correct modern choice), and API request/response
encoding. A mismatch anywhere in this chain (UTF-8 in the app, `latin1` column collation) causes
mojibake or silent data corruption for non-ASCII text, not just a display glitch.

**I10 — Font fallback coverage for non-Latin scripts.** Check the font stack (CSS `font-family`,
mobile app font configuration) actually includes fallback fonts covering every non-Latin script
present in the supported-locale list — CJK (Chinese/Japanese/Korean), Cyrillic, Arabic, Devanagari,
etc. A font stack that only lists Latin-script webfonts with a generic `sans-serif` fallback may
render unsupported characters as tofu/boxes on systems lacking a suitable installed fallback —
worth flagging even though actual rendering can't be fully confirmed without a real device/browser
per locale; note this as the visual-verification limitation it is.

## Locale Switching & Fallback Chain

**I11 — Locale detection, persistence, and live switching.** Check how the active locale is
determined — explicit user preference (saved setting), browser/OS locale
(`navigator.language`/`Accept-Language` header/platform locale API), or URL/path-based routing
(`/de/...`, `?lang=de`) — and that a user's explicit choice is actually persisted (cookie,
localStorage, account setting, route) and takes effect immediately without requiring an app
rebuild or restart. A locale switcher that only changes the stored preference but doesn't
re-render already-mounted translated content until a full page reload is a real (if minor) defect
worth noting.

**I12 — Regional-variant fallback chain.** Check whether a fallback chain is defined for regional
locale variants — e.g. `de-CH` falling back to `de` and then to the default/base locale — rather
than a missing regional variant silently breaking (falling back to the key name, throwing, or
falling back to an unrelated default without the user realizing it). Verify this in the actual
i18n library configuration (`fallbackLng` in i18next, resource-fallback rules in FormatJS/.NET
`CultureInfo` parent-culture resolution, Android's language-then-region resource qualifier
resolution) rather than assuming the framework does this automatically — some require explicit
configuration.

## Translation Workflow & Sync

**I13 — Source-key/translation-file sync.** Check whether translation keys used in code and keys
present in the translation files actually match in both directions: no keys used in code but
missing from the base/reference locale file (a translation gap that will always fall back), and no
orphaned keys left in translation files after a feature was removed from the code (dead weight
that also risks confusing future translators). Check for an automated extraction/lint step in the
build or CI (`i18next-parser`, `react-intl`'s extract command, a gettext `xgettext`/`msgfmt
--check` step, a custom script diffing used-keys vs. defined-keys) that catches drift
automatically — if sync is fully manual with no verification step anywhere in the repo's scripts
or CI config, report that explicitly as the gap it is, even if no drift happens to exist today.

## Locale-Specific Units & Conventions

**I14 — Measurement units, first-day-of-week, address/phone-number format.** Check whether
measurement units (metric vs. imperial), calendar first-day-of-week (Monday vs. Sunday, relevant
to any date-picker component), and address/phone-number field structure and validation are adapted
per locale where the product's actual target markets require it, rather than one convention
hardcoded as if it were universal (a fixed "MM/DD/YYYY"-shaped address form field order, a
hardcoded phone-number regex that only matches one country's format, weight/distance fields
labeled only in one system with no conversion). If the product's actual target markets all share
one convention (e.g. a DACH-only product), say so and mark this `➖ N/A`/low-priority rather than
inventing a defect that doesn't matter for the real target markets.

## Cultural & Market Appropriateness (needs human review)

**I15 — Imagery, color connotations, icons, and idioms.** If the product genuinely targets
multiple culturally distinct markets, flag that imagery choices, color connotations (e.g. colors
carrying different associations across cultures), icon meanings, and idiomatic phrasing in
marketing/UI copy may not translate culturally even when they translate linguistically. This is
inherently a human/local-market judgment call, not something verifiable by reading code — report
it as `⚠️ needs local-market review` with a note of what to have reviewed (hero imagery, iconography,
color usage in status/branding, any idiom or culturally-specific reference in copy), rather than
attempting to render a verdict or silently omitting the item. If the product targets a single
culturally homogeneous market, mark `➖ N/A` and say so.

## Boundary with accessibility-check

Language-attribute (`lang=`) correctness per locale and screen-reader announcement of language
changes within a page are covered by `accessibility-check` (its `references/web-accessibility.md`
Understandable section). This skill does not re-derive that check — if it comes up during
investigation, note it as a one-line pointer to `accessibility-check` rather than giving it its
own row here, so running both skills together doesn't produce duplicate or conflicting findings.

## Output format

Start with 3-4 sentences: which i18n/l10n framework (or "none") was identified, which locale files
and supported-locale list were found, what couldn't be fully verified (no real device/browser per
locale to visually confirm text expansion or RTL rendering, no native speaker available to judge
translation quality or cultural appropriateness), and the caveats from Ground rules — this is
static source/config review, not a linguistic QA pass, a visual rendering check, or a
market-appropriateness judgment.

Then ALWAYS use this exact table — one row per check, none omitted:

| # | Check | Bereich/Area | Status | Befund/Evidence | Empfehlung/Recommendation |
|---|---|---|---|---|---|
| 1 | ... | Strings/Translation/Formatting/Layout/RTL/Encoding/Locale/Workflow/Culture | ✅/❌/⚠️/➖ | file:line, translation-file diff, or formatting code excerpt | only if not ✅ |

(Match the table's actual language to the conversation's language — the column names above are
illustrative, not a fixed vocabulary. Keep "Befund"/"Evidence" concrete: a path+line with the
actual string/code found, an actual key-count diff between locale files, or "not found — searched
X, Y, Z". Group rows by area/theme — matching the H2 sections above — with a subheading or an
"Area" column, whichever reads more clearly for the number of rows involved.)

Status legend:
- ✅ Pass — the i18n-safe mechanism (translation key, plural API, locale-aware formatter, RTL-safe
  CSS, fallback chain, sync tooling) was found and verified in code/config
- ❌ Fail — checked, the required mechanism is missing or a hardcoded/naive alternative was found
  in its place
- ⚠️ Needs manual/native-speaker/local-market review — code can't fully answer this (translation
  linguistic quality, actual rendered text-expansion overflow, actual RTL reading experience,
  cultural appropriateness of imagery/color/idiom)
- ➖ N/A — this check's surface doesn't apply here (no RTL locale supported, single-locale product
  with no i18n framework, target markets share one convention) — state why in one clause; for the
  cultural-appropriateness check specifically, the *reasoning* for scope is part of the deliverable

End with a **prioritized punch list**: every ❌, ordered so failures most likely to produce visibly
broken or unusable text for real non-source-locale users come first (missing translations, broken
pluralization, hardcoded formats/strings), followed by lower-severity/cosmetic issues, each with
the one-line fix. Follow it with a **needs-review list**: every ⚠️, since those are exactly the
items a human — a native speaker, a visual QA pass on a real device, or someone with local-market
knowledge — needs to close out.
