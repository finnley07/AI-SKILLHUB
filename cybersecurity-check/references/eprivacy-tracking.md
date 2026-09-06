# ePrivacy / cookies & tracking checks

Separate legal basis from GDPR: Directive 2002/58/EC as amended (the "Cookie Directive"), Art.
5(3) — storing or accessing information on a user's device requires **consent**, with a narrow
"strictly necessary" exemption, *regardless* of whether the information counts as personal data
under GDPR. Germany implements this nationally via the TTDSG (§25) — check which national
implementation applies if the target market is known. This is why analytics/crash-reporting SDKs
need consent even for supposedly "anonymous" data: the legal trigger is *accessing the device*, not
what's in the payload.

**T1 — Non-essential trackers only load after consent.** Find every analytics/crash-
reporting/advertising SDK's init call (`Firebase.initializeApp`, `FirebaseAnalytics.instance...`,
a tracking `<script>` tag, an ad SDK's `init()`, Sentry/Crashlytics setup). Confirm each is gated
behind an actual consent check (a stored consent flag read before init, or the SDK only being
added to the build/added to the dependency graph conditionally) — not firing unconditionally at
app/page start. **Crash reporting is the most commonly missed one** — teams gate marketing
analytics carefully and then leave Crashlytics/Sentry running from the first launch "because it's
just crash data." If personal identifiers (user ID, email, device ID) are attached to crash
reports, it's in scope for T1 same as any other tracker.

**T2 — Strictly-necessary exemption isn't over-claimed.** Check what, if anything, loads before
consent is given, and whether each item genuinely qualifies as strictly necessary (session
cookies/tokens needed to keep the user logged in, load-balancing, security features) rather than
being reclassified as "necessary" to dodge the consent gate. A/B testing and performance analytics
are commonly and incorrectly labeled necessary.

**T3 — Consent UI has no dark patterns.** If there's a consent screen/banner, check:
- Reject is as easy/prominent as accept (not hidden behind an extra click, not visually
  de-emphasized while "Accept All" is a bright primary button).
- No pre-ticked boxes for optional purposes.
- No "consent wall" that blocks basic app function entirely if the user declines optional
  tracking (blocking *login itself* behind accepting analytics, for instance, invalidates the
  "freely given" requirement from GDPR Art. 7 too).

**T4 — Purpose granularity.** Consent should be specific per purpose, not one bundled toggle for
"marketing + analytics + AI features + third-party sharing" as a single yes/no. Check whether
distinct purposes get distinct consent controls (the project's own consent screen, if one exists,
is the place to check this directly).

**T5 — Withdrawal is as easy as granting.** A working, discoverable path (account/privacy settings)
to review and withdraw each consent independently — not "email us to opt out," not a one-way
consent that can only be granted, never revoked in-app.

**T6 — Mobile-specific device identifiers.** On mobile, the "device information" ePrivacy targets
includes advertising IDs (IDFA/GAID), and platform rules layer on top (Apple's App Tracking
Transparency requires its own OS-level prompt before IDFA access on iOS). Check whether the app
reads any device/advertising identifier, and if so, whether it's gated the same way as other
tracking — plus, on iOS, whether ATT is actually implemented if IDFA is touched at all.
