# Web accessibility — WCAG 2.1/2.2 Level AA checks

Organized by the four WCAG principles: Perceivable, Operable, Understandable, Robust (POUR). Skip
this whole file only if the project has no web frontend at all (mark a single row `➖ N/A — no web
frontend, mobile-only project`) — otherwise every check below gets a row.

## Perceivable

**P1 — Alt text on meaningful images.** Grep for `<img` (and `background-image:` used to convey
content rather than decoration) and check each one. A meaningful image needs `alt` text that
conveys its *purpose*, not a filename or "image of...": `<img src="chart.png" alt="Q3 revenue up
12% year over year">` is compliant; `<img src="chart.png">` or `<img src="chart.png"
alt="chart.png">` is not. For images that are also links/buttons (a logo linking home, an icon
button), the `alt`/`aria-label` must describe the *action*, not the picture: `<a href="/"><img
alt="Startseite"></a>`, not `alt="Logo"`.

**P2 — Decorative images marked so assistive tech skips them.** Purely decorative images (spacers,
background flourishes, icons that repeat an adjacent visible text label) should have `alt=""` (an
empty alt, not a missing one) or `role="presentation"`/`aria-hidden="true"`, or be implemented as a
CSS `background-image` in the first place. Grep for icon components rendered next to text labels
that don't set `aria-hidden="true"` on the icon — screen readers will otherwise announce the icon
name and then the redundant text.

**P3 — Color contrast for text.** WCAG 1.4.3: normal text needs ≥4.5:1 contrast against its
background; large text (≥18pt / 24px, or ≥14pt/18.66px bold) needs ≥3:1. Pull the actual computed
foreground/background hex values from the CSS/theme/design-token file (not an assumed brand color —
check for opacity/alpha blending, hover/disabled state overrides, and text over images/gradients,
which often fail even when the base palette passes) and compute the real ratio
((L1+0.05)/(L2+0.05) using relative luminance). Report the actual ratio found, e.g. "grey text
`#8A8A8A` on white `#FFFFFF` = 3.2:1 — fails 4.5:1 for body text."

**P4 — Color contrast for UI components and graphical objects.** WCAG 1.4.11: non-text elements
that convey meaning or state — input borders, button boundaries, icons carrying meaning, focus
indicators, chart data series — need ≥3:1 contrast against their adjacent background. Check
disabled-looking-but-actually-active states and low-contrast "ghost button" outlines specifically;
these are common failures.

**P5 — Captions and transcripts for audio/video.** Grep `<video>`/`<audio>` usage and embedded
players (YouTube/Vimeo iframes). Video needs synchronized captions — `<track kind="captions"
src="...">` or the platform player's caption feature actually enabled with real (not
auto-generated-and-unreviewed) captions; audio-only content needs a text transcript linked nearby.
Flag auto-generated-only captions as `⚠️` (accuracy needs human review), not `✅`.

**P6 — Content not conveyed by color alone.** WCAG 1.4.1: check required-field indicators (a red
asterisk with no text/symbol alternative fails; a red asterisk plus `aria-required="true"` and/or
visible "required" text passes), validation states (a red border alone on an invalid field fails;
red border + icon + text message passes), and status/chart legends (color-only legend fails;
color + pattern/label passes).

**P7 — Text resize and reflow.** Check the viewport meta tag for `user-scalable=no` or
`maximum-scale=1` (both block pinch-zoom — a WCAG 1.4.4 failure) and remove/loosen them if found.
Check that content reflows to a single column without horizontal scrolling when the viewport is
narrowed or zoomed to 400% (WCAG 1.4.10) — look for fixed-width containers (`width: 960px` with no
`max-width: 100%`) that would force horizontal scroll.

**P8 — Meaningful sequence.** WCAG 1.3.2: the visual reading order (as arranged by CSS —
`flex-direction: row-reverse`, `order:`, absolute positioning, CSS Grid `grid-template-areas`)
should match the DOM order that a screen reader announces. Grep for `order:` and `flex-direction:
row-reverse`/`column-reverse` in layout CSS and spot-check that the visual result still makes sense
read linearly.

## Operable

**O1 — Full keyboard operability, no keyboard traps.** Every interactive element (links, buttons,
form controls, and any custom widget — dropdown, tab panel, carousel, date picker, modal) must be
reachable via Tab and operable via Enter/Space/Arrow keys without a mouse. For each custom
interactive component found (typically a `<div>`/`<span>` with a click handler — see R1), check for
a matching `onKeyDown`/`keydown` handler implementing the expected key behavior. Check modal/dialog
implementations specifically for a keyboard trap: focus should move into the dialog on open, stay
within it while open (trapped *intentionally*), and both Escape and a visible close control must
return focus to the trigger element — a modal with no Escape handler and no visible focus-trap
release is a genuine keyboard trap (WCAG 2.1.2 failure).

**O2 — Visible focus indicators not suppressed.** Grep CSS for `outline: none`, `outline: 0`, or
`outline-style: none` applied to focusable elements (`:focus`, `a`, `button`, `input`, or a blanket
`*`/`:focus` reset). If found, verify there is a replacement focus style (a `:focus-visible` rule
with a visible box-shadow/border/outline of its own) — `outline: none` with no replacement is a
WCAG 2.4.7/2.4.11 failure regardless of how the rest of the design looks. A common false negative:
the replacement style exists but only on `:focus` and gets overridden by a later, more specific
selector — check cascade order, not just presence of the rule.

**O3 — Logical tab order.** Grep for `tabindex="` with a positive integer value (`tabindex="1"`,
`"2"`, etc.) — these override natural DOM order and almost always create a confusing tab sequence
once the page has more than a couple of them; `tabindex="0"` (adds to natural order) and
`tabindex="-1"` (programmatic focus only, removed from tab order) are the only values that should
normally appear.

**O4 — Skip-to-content link.** Check that the first focusable element on the page (visually hidden
until focused, or always visible) is a "Skip to main content"/"Zum Inhalt springen" link targeting
the main content region — required for any site with a non-trivial navigation/header a keyboard
user would otherwise have to tab through on every page.

**O5 — No seizure-inducing flashing content.** WCAG 2.3.1: check any animation, auto-playing
video, or flashing UI element (a blinking sale banner, a rapidly-cycling carousel with hard cuts)
for flash rate — anything flashing more than 3 times per second in a way that covers a significant
screen area is a failure.

**O6 — Sufficient touch/pointer target size.** WCAG 2.5.8 (AA, WCAG 2.2): interactive targets need
at least 24×24 CSS px, unless inline in text or with adequate spacing to adjacent targets. Check
icon-only buttons and closely-packed action rows (table row actions, chip "remove" buttons) for
actual rendered size including padding, not just the icon's intrinsic size.

**O7 — Timing adjustable, no unexpected context changes.** Check for auto-advancing carousels/
timed redirects with no pause/extend control, session timeouts with no warning or extension option,
and `onchange` handlers on `<select>` elements that immediately navigate/submit without the user
confirming (WCAG 3.2.2) — a select-driven navigation should require an explicit "Go"/submit action,
or be clearly signposted as immediate.

## Understandable

**U1 — Form inputs have associated labels, not just placeholder text.** Grep `<input`, `<select>`,
`<textarea>` and check each has a real `<label for="id">` matching the input's `id`, or
`aria-label`/`aria-labelledby`. `<input placeholder="Email">` with no `<label>` is a WCAG 1.3.1/4.1.2
failure — placeholder text disappears on input, is typically low-contrast, and many screen readers
don't announce it reliably. Compliant: `<label for="email">Email</label><input id="email"
placeholder="you@example.com">`. Non-compliant: `<input placeholder="Email" type="email">` alone.

**U2 — Error messages are specific and programmatically associated with their field.** Check
validation error rendering: the error text should be linked to its input via
`aria-describedby="error-id"` pointing at the element containing the message, and the input should
get `aria-invalid="true"` while invalid. A red border plus a generic toast ("Please fix the errors
above") with no per-field association is a WCAG 3.3.1/4.1.3 failure — a screen-reader user tabbing
into the field hears nothing about what's wrong with it specifically.

**U3 — Consistent navigation and identification.** WCAG 3.2.3/3.2.4: check that primary navigation
appears in the same relative order across pages/templates, and that a control performing the same
function (a search icon, a "delete" trash icon) uses the same icon/label everywhere in the app
rather than switching between synonyms or icons page to page.

**U4 — Language attribute set.** Check `<html lang="...">` is present and matches the page's actual
language (`lang="de"` for a German-language app, `lang="en"` for English), and that any
substantial passage in a different language (an English pull-quote inside a German page) wraps in
its own `lang="en"` span — WCAG 3.1.1/3.1.2. Missing/wrong `lang` breaks screen-reader
pronunciation entirely.

**U5 — Predictable behavior on focus and input.** WCAG 3.2.1/3.2.2: focusing an element must never
by itself trigger a context change (navigation, form submission, opening a new window); changing a
form field's value should only trigger a context change if the user was told to expect it (e.g. a
labeled "auto-saves as you type" behavior is fine, a silent redirect on `<select>` change without
warning is not — see O7).

## Robust

**R1 — Semantic HTML for interactive elements.** Grep for `<div` or `<span` combined with
`onClick`/`onclick`/`@click` and no `role="button"` + `tabindex="0"` + keydown handler alongside it
— this is the single most common accessibility bug: a click-only fake button that's invisible to
keyboard and screen-reader users. Compliant: `<button onClick={...}>Save</button>` (get keyboard
support, focusability, and the `button` role for free). Acceptable but weaker: `<div role="button"
tabindex="0" onClick={...} onKeyDown={handleEnterSpace}>` — only when a real `<button>`/`<a>`
genuinely cannot be used. Same check applies to fake links (`<span>` styled as a link with a click
handler instead of `<a href>`).

**R2 — Proper heading hierarchy.** Grep all `<h1>`–`<h6>` (and heading-styled components using
`role="heading" aria-level="N"`) on a representative page and list them in document order. Check
for: more than one `<h1>` (usually should be exactly one per page), and skipped levels (an `<h2>`
followed directly by an `<h4>` with no `<h3>` — a WCAG 1.3.1 failure that breaks screen-reader
users' ability to navigate by heading outline).

**R3 — ARIA used correctly, not overused or misused.** The first rule of ARIA is not to use ARIA
when a native HTML element already provides the semantics — grep for `role="button"` on an actual
`<button>` element, or `role="list"` wrapping actual `<ul>`/`<li>` (redundant, sometimes harmful,
depending on browser/AT combination). Where ARIA genuinely is needed (custom widgets), check that
roles come from the actual ARIA spec (no invented role names), that required states are present and
kept in sync with real behavior (`aria-expanded` on a disclosure toggle actually flips
`true`/`false` when it opens/closes — check the code updates it, not just that it's present in the
initial markup), and that `aria-hidden="true"` is never applied to an element that also contains
focusable children (creates "ghost" focusable-but-unannounced controls).

**R4 — Landmark regions present.** Check for `<header>`, `<nav>`, `<main>`, `<footer>` (or their
ARIA-role equivalents `role="banner"/"navigation"/"main"/"contentinfo"`) on the page shell, with
exactly one `<main>` per page — landmarks let screen-reader users jump directly to page regions
instead of reading linearly from the top every time.

**R5 — Name, Role, Value exposed for custom widgets.** WCAG 4.1.2: for any custom-built control
that isn't a native HTML element (a custom checkbox/switch/dropdown/tabs/slider built from styled
`<div>`s), verify all three are present and dynamically correct: an accessible **name** (visible
text, `aria-label`, or `aria-labelledby`), the correct **role** (`checkbox`, `switch`, `tab`,
`slider`, etc. — not a generic `button` role hiding a checkbox's actual behavior), and **value/state**
kept current in the DOM as the user interacts (`aria-checked`, `aria-expanded`, `aria-selected`,
`aria-valuenow` actually updated by the event handler, not just set once at initial render).
