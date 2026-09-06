# GDPR / DSGVO checks

Each check cites the exact article so findings are defensible, not vibes. "Compliant" is a legal
conclusion a lawyer makes — you're gathering the technical/documentary evidence a lawyer (or the
user) needs to make that call. Report evidence, not verdicts phrased as certainty.

## Legal basis & governance

**G1 — Lawful basis per purpose (Art. 6).** Every distinct processing purpose (account creation,
AI coaching, marketing email, analytics, …) needs an identified legal basis — consent, contract
performance, legitimate interest, etc. Check: is there a document/table mapping purposes to bases,
or does the privacy policy at least state one per purpose? A basis "implied" by the code existing
is not documented — flag it.

**G2 — Special category data has an Art. 9 basis.** Health data, biometric data, sex life/
orientation, religious/political/union data, genetic data all fall under Art. 9 — processing is
prohibited by default unless an Art. 9(2) exception applies (explicit consent is the common one for
consumer apps). Check: does the app process any of these (health metrics, workout/nutrition data
tied to a person, pain/injury notes, etc.)? If yes, is there an *explicit, separate* consent for
it — not bundled into a general ToS checkbox? (Explicit ≠ implicit: a single "I accept the Privacy
Policy" checkbox that also covers Art. 9 data is a common and real defect, not a formality.)

**G3 — Records of Processing Activities / VVT (Art. 30).** A document (not code) listing: what's
processed, why, by whom, retention period, recipients, international transfers. Look for it in the
repo's docs or ask if it exists elsewhere. Missing entirely is common pre-launch but is a real gap
once processing begins — most controllers must maintain this regardless of size when processing is
not occasional, or the data is special-category, or there's a risk to rights and freedoms (Art.
30(5) exemption is narrower than most teams assume).

**G4 — DPIA / Datenschutz-Folgenabschätzung (Art. 35).** Required when processing is "likely to
result in a high risk to the rights and freedoms of natural persons" — large-scale special-category
data processing is explicitly called out in Art. 35(3)(b) as a trigger. A health/fitness app
processing Art. 9 data at any real user count is a strong candidate for requiring one. Check: does
one exist? If special-category data is processed and no DPIA is documented, that's a genuine gap,
not a nice-to-have.

**G5 — DPO requirement (Art. 37).** Mandatory when core activities involve large-scale, regular and
systematic monitoring, or large-scale Art. 9 processing. Check: has anyone actually assessed
whether this threshold is met (documented, even if the answer is "not yet at our scale")? An
unexamined assumption of "we're too small" is itself the gap to report, distinct from actually
needing a DPO today.

**G6 — Privacy by design and by default (Art. 25).** Defaults should minimize processing —
opt-in for optional processing, not opt-out; new features default to the least invasive setting.
Check concrete instances: are optional consents (marketing, AI features, analytics) *unchecked* by
default in the UI? Does a new user's data footprint start minimal and grow only with explicit
action?

## Data subject rights (Art. 12–22)

For each right, the check is the same shape: **does a *technical* path exist for a user to exercise
it**, not just a promise in the privacy policy text.

**G7 — Right of access / portability (Art. 15, Art. 20).** A working data-export feature covering
*all* personal data held (not a curated subset) in a portable, machine-readable format.

**G8 — Right to erasure (Art. 17).** Account deletion actually deletes or properly anonymizes —
verify it cascades to related tables (don't take the top-level "user deleted" flag's word for it;
check for orphaned rows referencing the deleted user's ID across the schema). Also check retained
copies: backups, logs, analytics exports, AI provider request logs — Art. 17 doesn't require
instant purging of encrypted backups, but the retention/deletion timeline for those needs to be
documented (ties to G14 below).

**G9 — Right to rectification (Art. 16).** Can the user actually edit the data that's wrong (profile
fields, at minimum)?

**G10 — Right to restrict processing (Art. 18) / Right to object (Art. 21).** Less commonly
implemented technically — often handled via support/manual process. Check whether such a process is
at least documented; a total absence with no fallback process is the finding.

**G11 — No unsafeguarded solely-automated decisions (Art. 22).** If an AI/algorithm makes a decision
with legal or similarly significant effect on the user without human involvement, Art. 22 restricts
this unless an exception + safeguards apply. Check: does AI-generated output (a training plan, a
nutrition target) apply automatically, or does it require the user to review and confirm before it
takes effect? A "preview, then user applies" pattern generally keeps this out of Art. 22's core
prohibition — auto-apply without review is the pattern to flag.

**G12 — Consent quality (Art. 7).** Consent must be freely given, specific, informed, unambiguous,
and withdrawable as easily as it was given. Check the actual consent UI: are unrelated purposes
bundled into one checkbox (not specific)? Is there a working, equally-accessible withdrawal path
(account settings, not "email support to opt out")? Is pre-ticking used anywhere (invalid under
Art. 7 and under the ePrivacy consent standard it borrows — see `eprivacy-tracking.md`)?

**G13 — Response process within 1 month (Art. 12(3)).** A technical capability to fulfill a request
is necessary but not sufficient — is there a documented internal process (who handles an incoming
request, by when) so the 1-month clock is actually met in practice, not just possible in theory?

## Retention & deletion

**G14 — Storage limitation (Art. 5(1)(e)) + documented retention/deletion concept.** Every category
of stored data needs a defined retention period tied to its purpose, not indefinite retention by
default. Check for a written policy covering: primary DB, backups, logs (including third-party
provider logs — AI request logs, crash reports), and whether deletion in production actually
enforces those periods (a cron job / scheduled task / migration policy) or if it's aspirational
text with nothing executing it.

## International transfers (Art. 44–49)

**G15 — Transfer mechanism for every third-country processor.** Any processor outside the
EU/EEA (or a non-adequacy-decision country) needs a valid transfer mechanism — Standard
Contractual Clauses (SCCs) are the common one for US-based SaaS/API providers. Check: for every
US-based (or other third-country) service found in code (AI provider, hosting, email, analytics),
is a transfer mechanism referenced anywhere in the project's compliance docs?

**G16 — Transfer Impact Assessment (post-Schrems II expectation).** Beyond having SCCs on paper,
regulators expect an assessment of whether the destination country's laws (e.g. US surveillance
law) undermine the SCCs' protections in practice, and supplementary measures where needed
(encryption, minimization). Check whether this has been done for high-risk transfers (Art. 9 data
leaving the EU is the highest-stakes case here) — this is usually the single most-skipped step
even when SCCs are properly signed.

**G17 — Sub-processor visibility.** Does the primary processor (e.g. the AI vendor, the BaaS
provider) itself use sub-processors, and is that chain covered by the same transfer/contract
guarantees? Usually answered by the vendor's own DPA/sub-processor list, not your code — note this
needs vendor-side documentation, not something the repo can prove.

## Data-processing agreements (Art. 28)

**G18 — DPA/AVV with every processor.** Art. 28(3) requires a contract (or other legal act) with
specific mandatory content (subject matter/duration/nature/purpose of processing, obligations,
sub-processor rules, audit rights, deletion/return of data at contract end) for *every* entity that
processes personal data on the controller's behalf. Build the list from code (every third-party API
call sending user data out — AI provider, hosting, DB/BaaS, email/SMTP, analytics, crash
reporting, payment/subscription validation, CDN) and cross-reference against what's documented as
covered. Every processor found in code without a referenced DPA is a concrete ❌, named
individually — "some processors may lack a DPA" is not an actionable finding, "processor X has no
documented DPA" is.
