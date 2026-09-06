# Mobile-specific and other EU regulation applicability checks

## Mobile app security (skip if the project isn't a mobile app)

**M1 — Secure local storage.** Tokens/session credentials stored in Keychain (iOS) / Keystore or
`EncryptedSharedPreferences` (Android) — not plain `SharedPreferences`/`UserDefaults`/plaintext
files. Check the actual storage mechanism used for auth tokens specifically.

**M2 — Deep link / intent filter validation.** Beyond the general open-redirect check
(`security.md` S23): does the app's deep-link handler validate the incoming link's structure before
acting on it (navigating, pre-filling a form, triggering an action), or does it trust any
parameter in a `myapp://` link unconditionally? Android: is `autoVerify`/App Links used where
appropriate so only verified domains can trigger the link, rather than any app claiming the same
scheme?

**M3 — Permission minimization.** Does the app request only the platform permissions it actually
uses, at the point of use (runtime request with rationale) rather than everything upfront? Overly
broad permission requests are both a security surface and, for health-adjacent apps, a Google
Play/App Store review risk.

**M4 — App Store / Play Store data-safety accuracy.** The store listing's privacy
label/data-safety form is a public, binding-ish declaration of what data is collected and shared
— cross-check its claims (if visible/documented) against what the code actually collects and sends
to third parties. A mismatch here is a store-policy violation risk, independent of GDPR.

**M5 — Sensitive data not exposed via OS-level surfaces.** iOS app-switcher snapshots and Android
recent-apps thumbnails can capture whatever's on screen — check whether screens showing especially
sensitive data (health notes, payment info) blank/obscure themselves when backgrounded.

**M6 — Biometric handling, if used.** Face ID/Touch ID/fingerprint should unlock a local
keychain-stored secret, not transmit raw biometric data anywhere — confirm the implementation uses
the platform's local-auth API as a gate, not a custom biometric capture/storage path.

## EU regulation applicability — assess, don't assume

These regulations have specific scope tests. Most small/consumer apps fall outside several of
them — the useful check is confirming that with a real reason, not silently ignoring them or
assuming without checking. Report each as `➖ N/A (with reason)` or `⚠️ needs assessment` rather
than skipping the row entirely — a documented "doesn't apply because X" is itself a valid,
reportable finding.

**E1 — EU AI Act (Regulation (EU) 2024/1689), Art. 50 transparency.** Applies whenever the app
uses an AI/LLM system that interacts with users — requires clearly informing users they're
interacting with an AI system. Applicable from 2 August 2026. Check: does the app disclose
AI-generated content as AI-generated (a coach chat, auto-generated plans) somewhere the user
actually sees before or during interaction, not buried only in a ToS paragraph?

**E2 — European Accessibility Act (Directive (EU) 2019/882), obligations from 28 June 2025.**
Covers specific sectors: e-commerce, banking/financial services, e-books, passenger transport,
electronic communications, some audiovisual media, and products/services with a "consumer-facing"
digital element in those sectors. Check applicability first: does the app include e-commerce
(in-app purchases beyond a platform's own IAP flow, which the platform itself handles), banking-
like functionality, or fall under another listed category? If genuinely out of scope, say so with
the reason. If in scope or ambiguous, check basic accessibility: screen-reader labels on
interactive elements, sufficient color contrast, scalable text, no interaction that requires only
a gesture with no alternative.

**E3 — NIS2 Directive (Directive (EU) 2022/2555).** Applies to "essential"/"important" entities in
specific sectors (energy, transport, banking, health infrastructure, digital infrastructure, ICT
service management, public administration, space, plus manufacturers/providers above certain size
thresholds in several other sectors) — generally scoped to medium+ enterprises (≥50 employees or
≥€10M turnover) in those sectors. Check: does the entity operating this app plausibly meet a
sector + size test? For most early-stage consumer apps the honest answer is `➖ N/A`, but state the
threshold reasoning rather than asserting it without checking — this changes as a company scales.

**E4 — Digital Services Act (Regulation (EU) 2022/2065).** Applies to "intermediary services" —
hosting, online platforms, especially ones with user-generated content, marketplaces, or
recommender systems shown to EU users. Check: does the app host/moderate third-party/user-
generated content, or operate any kind of marketplace or content-recommendation feature? If not,
`➖ N/A` with that reasoning.

**E5 — PSD2 / Strong Customer Authentication (Directive (EU) 2015/2366).** Applies only if the app
handles payment initiation or card/bank transactions *directly*, rather than exclusively through
Apple/Google in-app purchase (which the platforms handle compliance for on the merchant's behalf).
Check the actual payment flow: is any payment processor integrated directly (Stripe, a bank API,
etc.) outside of platform IAP? If purchases go solely through App Store/Play Store IAP, `➖ N/A`
with that reasoning.
