# Frontend performance checks

Focused on what the browser (or mobile app WebView/runtime) has to download, parse, and render.
Mark this whole file `➖ N/A` if the project genuinely has no frontend/UI layer (a pure backend
API/service). Important framing for every check below: static code review can spot the *patterns*
that cause poor load/render performance, but it cannot measure the actual resulting metric (a real
LCP in milliseconds, an actual re-render count under real user interaction, a real Lighthouse
score) — that requires an actual Lighthouse/WebPageTest/Chrome DevTools Performance run against a
running build. Say so per-check and mark accordingly; don't report a guessed number as if it were
measured.

## Bundle size & code splitting

**F1 — Route/feature-level code splitting vs. one giant bundle.** Check whether the app uses
dynamic `import()`/lazy loading for routes or heavy features (React `lazy()` + `Suspense`, Vue
`defineAsyncComponent`, Angular lazy-loaded modules, Next.js/Nuxt automatic route splitting) or
whether everything ships in a single entry bundle. Run the project's actual build with a bundle
analyzer if available (`npm run build -- --analyze`, `source-map-explorer`, `webpack-bundle-
analyzer`, Vite's `rollup-plugin-visualizer`) and report actual output bundle size(s); if none is
configured, at minimum check the build output directory's file sizes after a real build
(`du -sh dist/**` or equivalent) rather than guessing.

**F2 — Heavy dependencies that could be lighter or lazy-loaded.** Grep `package.json` /
import statements for known-heavy libraries pulled in for a small feature (a full charting library,
moment.js instead of a lighter date lib, a full icon-font/icon-library import instead of
tree-shaken individual icons, a full lodash import `import _ from 'lodash'` instead of
per-function imports). Flag libraries that are imported fully but used for one or two functions, and
check whether a heavy library used on only one page/feature is code-split rather than in the main
bundle.

**F3 — Tree-shaking / dead code not actually eliminated.** If a bundle analyzer run is available,
check for duplicate versions of the same library (multiple versions of React, lodash, a UI kit)
bundled together — a common and easy-to-miss bundle-size bloat source, usually from mismatched
dependency versions or a package that isn't ESM/tree-shakeable.

## Rendering efficiency (React/Vue/etc.)

**F4 — Missing memoization causing avoidable re-renders.** For React: look for expensive
computations or component subtrees re-rendering on every parent render without `useMemo`/
`useCallback`/`React.memo` where the inputs are actually stable — but also check for the *inverse*
anti-pattern, `useMemo`/`useCallback` applied indiscriminately to cheap values (adds overhead
without benefit). The concrete thing to look for: an inline object/array/function literal passed as
a prop to a memoized child component (`<Child config={{...}} onClick={() => ...}` ) — a fresh
reference every render defeats `React.memo` on the child regardless of memoization elsewhere. Vue:
check for missing `computed()` where a derived value is recalculated inline in the template on
every render instead of being cached until its dependencies change.

**F5 — Non-keyed or index-keyed lists.** Grep `.map(` calls that render JSX/template lists — check
each uses a stable, unique `key` (a real ID), not the array index (`key={index}`) when the list can
reorder, filter, or have items inserted/removed. An index key causes React/Vue to misattribute
component state and do unnecessary full re-renders/re-mounts on reorder instead of efficiently
moving/reusing existing DOM nodes.

**F6 — Unstable context/state causing wide re-render trees.** Check whether a React Context
provider's `value` prop is a fresh object literal created on every render of the provider (`<Ctx.
Provider value={{a, b}}>`) — this re-renders every consumer on every provider render regardless of
whether `a`/`b` actually changed, since the whole point of Context is bypassed when the value
reference is never stable. Same idea for global state stores: check whether components subscribe to
the narrowest slice of state they need, or to the entire store (causing a re-render on any
unrelated state change).

## Images & static assets

**F7 — Unoptimized or unresized images.** Check whether images are served at a size appropriate to
their display size (not a 4000px-wide source image displayed at 200px), and whether the project
uses an image-optimization pipeline (a framework's built-in image component — Next.js `<Image>`,
Nuxt `<NuxtImg>` — an image CDN, or a build-time compression step) versus raw unoptimized files
served as-is. Spot-check actual file sizes of a few images in the repo/`public` folder against
their rendered dimensions if both are discoverable.

**F8 — Missing lazy-loading for below-the-fold images.** Check `<img>` tags (or their framework
equivalent) for `loading="lazy"` (or the framework's lazy-loading prop) on images that aren't in
the initial viewport — hero/above-the-fold images should explicitly *not* be lazy-loaded (that
delays LCP), so check both directions: lazy where appropriate, eager/priority-hinted where the
image is likely the LCP element.

**F9 — Modern image formats.** Check whether images are served as WebP/AVIF (directly, or via a
`<picture>` element with fallback, or an image CDN that content-negotiates format) versus only
legacy JPEG/PNG, where file-size savings would be meaningful (photographic content especially).

## Render-blocking resources

**F10 — Render-blocking CSS/JS in `<head>`.** Check for synchronous `<script>` tags (no `defer`/
`async`/`type="module"`) placed in `<head>` that block HTML parsing, and large CSS files loaded
render-blocking with no critical-CSS inlining strategy for above-the-fold content. Third-party
scripts (analytics, chat widgets, ads) are worth checking specifically — these are a common
render-blocking culprit and are easy to load `async`/`defer` or move to load after first paint
without functional loss.

**F11 — Web font loading strategy.** Check for `font-display: swap` (or `optional`/`fallback`) on
`@font-face` declarations, or the framework's font-optimization feature (Next.js `next/font`) —
without it, custom web fonts block text rendering until the font loads (flash of invisible text),
directly hurting perceived load speed.

## Network requests

**F12 — Duplicate or missing request deduplication.** Check whether multiple components
independently fetch the same data on mount (no shared cache/store), causing redundant network
requests for identical data within the same page load. A data-fetching library with built-in
deduplication and caching (React Query/TanStack Query, SWR, Apollo Client's cache, RTK Query) used
consistently is the good pattern; ad-hoc `fetch`/`axios` calls scattered per-component with no
shared cache is the anti-pattern to flag.

**F13 — Waterfall requests that could be parallelized.** Look for sequential `await`ed requests
where the second request doesn't actually depend on the first's result but is written after it
anyway (`const a = await fetchA(); const b = await fetchB();` where `b` doesn't use `a`) — these
should be `Promise.all([fetchA(), fetchB()])`'d. Also check for a client-side fetch that could have
been a single server-side/SSR fetch, avoiding a client-round-trip waterfall entirely (fetch after
mount for data the server already had available at render time).

**F14 — No client-side caching for rarely-changing data.** Reference/config-like data (a list of
categories, feature flags, app configuration) re-fetched from the network on every page navigation
rather than cached client-side (in-memory store, `localStorage`, a data-fetching library's cache
with an appropriate stale-time) — same underlying issue as B7 on the backend, on the client side.

## Core Web Vitals-relevant patterns

These are **code smells that tend to cause** poor Core Web Vitals scores — confirming the actual
LCP/CLS/INP numbers requires running Lighthouse, WebPageTest, or Chrome DevTools' Performance/
Web Vitals panel against a real deployed or locally-served build. Mark these `⚠️` with that
instruction unless such a tool was actually run as part of this audit, in which case report the
real scores.

**F15 — Layout shift (CLS) risk.** Check for images/video/ads/embeds rendered without an explicit
`width`/`height` (or `aspect-ratio`) reserving their space before the asset loads, and for content
(banners, cookie notices, ads) that injects itself above existing content after initial render
without reserved space — both cause visible layout shift.

**F16 — LCP-element loading priority.** Identify the likely largest-contentful-paint element (a
hero image, a large heading/text block) and check it isn't lazy-loaded (see F8), isn't loaded via a
render-blocked/deferred path, and — for an image — isn't unnecessarily large in file size (see F7)
or loaded through an extra client-side-only fetch when it could be present in the initial HTML/SSR
output.

**F17 — INP/interactivity risk from long main-thread tasks.** Look for expensive synchronous work
triggered directly in an event handler (large synchronous loops, JSON parsing of a large payload,
heavy DOM manipulation) that isn't broken up (`requestIdleCallback`, chunking, moving work to a Web
Worker, debouncing/throttling a handler that fires rapidly like `scroll`/`resize`/`input`) — long
synchronous handler execution is the direct cause of poor INP (input responsiveness).
