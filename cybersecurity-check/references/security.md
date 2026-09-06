# Application & infrastructure security checks

Loosely organized around the OWASP Top 10 (web) and OWASP Mobile Top 10, plus the infra items a
real "backend + frontend + deployment" review needs. As with every check in this skill: read the
actual code path, don't infer from the framework's defaults — a framework being secure by default
doesn't mean this project didn't override that default.

## Injection & input handling

**S1 — SQL/NoSQL/command injection.** Check that database access goes through an ORM/parameterized
queries, never raw string-concatenated SQL built from request input. Same for any shell command
construction (`Process.Start`, `exec()`, `subprocess`) — user input must never reach a command
string unescaped/unparameterized.

**S2 — Input validation & output encoding.** Are request bodies validated against a schema
(required fields, types, ranges) server-side — not just client-side, which an attacker bypasses
trivially by calling the API directly? Is output that gets rendered as HTML/markup properly
encoded to prevent XSS (relevant if there's any webview, server-rendered page, or markdown/HTML
rendering of user content)?

**S3 — File upload restrictions.** If the app accepts file uploads (images, documents — progress
photos, avatars): is file type validated by actual content inspection (magic bytes), not just the
extension or client-supplied MIME type? Is there a size limit? Are uploaded files stored outside
the web root or served with a content-type that prevents execution (e.g. `Content-Disposition:
attachment` for anything that isn't meant to render inline)?

**S4 — XXE.** If XML parsing exists anywhere, confirm external entity resolution is disabled —
otherwise a crafted XML payload can read local files or trigger SSRF via the parser itself.

**S5 — Insecure deserialization.** If the app deserializes untrusted data into typed
objects/dynamic types (not plain JSON-to-DTO with a fixed schema), check whether the deserializer
is restricted to expected types — polymorphic/type-name-embedded deserialization from untrusted
input is a common RCE vector.

## Authentication & session management

**S6 — Password/credential policy.** Minimum length/complexity enforced server-side (not just in
the frontend form), and checked against known-breached-password lists if the auth provider
supports it (e.g. HaveIBeenPwned integration, common in modern auth-as-a-service providers).

**S7 — Brute-force / credential-stuffing protection.** Rate limiting specifically on
login/password-reset/registration endpoints (tighter than the general API rate limit), and ideally
account lockout or backoff after repeated failures. Check the actual rate-limit config, not just
that a generic rate limiter exists somewhere in the stack.

**S8 — Token/session validation is complete.** JWT or session validation checks signature,
issuer, audience, and expiry — not just "a token is present and parses." A validator that skips
issuer/audience checks accepts tokens meant for a different service/environment.

**S9 — Sessions/tokens are actually revocable.** Does "log out" invalidate server-side state (or
at minimum shorten token lifetime meaningfully), or does a stolen token remain valid until natural
expiry regardless of logout? For stateless JWT setups, check whether there's any revocation
mechanism at all (short expiry + refresh rotation is the common mitigation) — no mechanism is a
real gap, not an inherent limitation to shrug off.

**S10 — MFA availability.** Not always required, but check whether it's offered at all for
accounts, especially any with elevated privileges (admin/staff accounts).

## Access control

**S11 — Authorization is checked per-endpoint, not just authentication.** Authenticated ≠
authorized. Spot-check: does a regular user's valid token let them hit admin-only or
other-user's-data endpoints if they guess the URL/ID? This is the single most common real-world
finding — test at least one "fetch resource by ID" endpoint for an object-level check (IDOR): does
the handler verify the resource belongs to the requesting user, or does it trust the ID alone?

**S12 — Privilege escalation paths.** Can a regular user grant themselves elevated
roles/entitlements (admin, paid features) through any endpoint that doesn't itself require the
elevated role to call? Check any "grant role" / "grant entitlement" endpoint specifically for who's
allowed to call it.

## Security misconfiguration

**S13 — Debug/verbose modes off in production.** Stack traces, detailed error messages, debug
endpoints (Swagger/OpenAPI UI, GraphQL introspection, admin consoles) should not be exposed
publicly in production — check the actual environment-conditional config, not just that a flag
exists.

**S14 — Default/sample credentials removed.** No leftover `admin/admin`, seeded test accounts, or
example API keys reachable in a production build/deploy.

**S15 — Secrets management.** No API keys/passwords/connection strings hardcoded or committed to
the repo (check git history if asked to go that deep, not just the current HEAD). Secrets come from
environment variables or a secret manager, are excluded via `.gitignore`, and are rotated if one
was ever accidentally exposed (check for evidence of a known-past leak in commit history or prior
incident notes — a leaked-then-rotated secret is a closed finding; leaked-and-never-rotated is not).

**S16 — Dependency vulnerabilities.** Run the stack's native audit tool rather than guessing:
`dotnet list package --vulnerable`, `npm audit`, `flutter pub outdated` (Dart/Flutter has no
built-in CVE audit — cross-check flagged outdated packages against a CVE database manually), `pip-
audit`, `bundle audit`, etc. Report actual findings, not "dependencies were not checked" as a
silent skip.

## Transport & headers

**S17 — TLS/HTTPS enforced.** No cleartext HTTP reachable in production for API or web traffic;
check any cleartext-allow flags a mobile app might ship with (Android `usesCleartextTraffic`, a
custom network security config, iOS `NSAllowsArbitraryLoads`) are scoped to debug builds/local
networking only, not blanket-enabled in release.

**S18 — HSTS and security headers.** `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`,
`X-Frame-Options`/`frame-ancestors` CSP, `Referrer-Policy` — check actual response headers (a
`curl -I` against a real endpoint if one is reachable, or the reverse-proxy/middleware config if
not yet deployed).

**S19 — CORS is not wide open in production.** `Access-Control-Allow-Origin: *` combined with
credentialed requests is a real vulnerability, not just permissive config — check the actual
allowed-origins list is a specific allow-list in production, not empty (which some frameworks
silently treat as "allow anything").

**S20 — CSRF protection where cookie-based sessions exist.** If any part of the system (an admin
panel, a web dashboard) uses cookie-based auth rather than bearer tokens in headers, check for CSRF
tokens or `SameSite` cookie attributes — bearer-token APIs consumed only by a mobile app are
generally not CSRF-exposed, so this is conditional on the actual auth transport used, not a
blanket requirement.

## Rate limiting & abuse prevention

**S21 — API-wide rate limiting, and correctly partitioned.** A global rate limiter exists and
partitions by authenticated user / IP correctly — a limiter keyed only by a header that's identical
for every client behind a shared proxy effectively rate-limits nothing per-actual-user. Sensitive
endpoints (auth, password reset, any AI/LLM-backed endpoint that costs money per call) should have
tighter limits than general CRUD.

**S22 — SSRF via user-controlled server-side fetches** — see the dedicated method in this same
file's companion checks below (kept here since it's squarely an OWASP-category app-security item).

## SSRF, open redirect, webhook replay (detailed methods)

**S22 — SSRF.** The risk: any backend code path where the server itself makes an outbound
HTTP(S) request to a URL a user (directly or indirectly) influences — webhook URL fields,
"import from URL," avatar/image-from-URL, PDF-from-URL, URL preview/unfurling, proxy endpoints,
OAuth callback verification, a "test this webhook" feature. Without a check, an attacker points
the server at `http://169.254.169.254/latest/meta-data/` (cloud metadata), an internal-only
service, or `127.0.0.1:<port>` and the server fetches it on their behalf.
  1. Grep outbound HTTP client usage (`HttpClient`, `fetch(`, `axios`, `requests.get/post`, etc.)
     and trace which call sites take a target URL/host derived from user input.
  2. For those, confirm there's a check *before the request executes* that resolves the hostname
     and rejects private/reserved ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`,
     `127.0.0.0/8`, `169.254.0.0/16`, IPv6 loopback/link-local) — and that it checks the
     *resolved* IP, not just the literal string (a naive string check misses DNS rebinding,
     decimal/octal/hex IP encodings like `http://2130706433/`, and IPv6-mapped IPv4).
  3. No such user-influenced outbound fetch anywhere → `➖ N/A`, state what you searched for.

**S23 — Open redirect after login.** The risk: a login/OAuth flow accepting a
`redirect_to`/`returnUrl`/`next` param and redirecting there post-success without validating it's
same-origin/allow-listed — used for phishing (victim logs in on the real site, gets bounced to an
attacker page right after, so nothing looked wrong at the point of credential entry).
  1. Find every redirect target read from client-controlled input: query params, POST body, a
     deep-link/URI-scheme handler, an OAuth `state`/`redirect_uri` echoed back.
  2. Confirm same-origin check or allow-list validation — and that it's an actual URL parse, not a
     prefix-string check (`startsWith('/')` alone still allows `//evil.example.com` and
     `/\evil.example.com` in many routers/browsers).
  3. Mobile: check the deep-link intent filter / URL scheme handler doesn't blindly navigate to an
     arbitrary URL embedded in the incoming link.

**S24 — Webhook replay protection.** The risk: a webhook receiver (payment provider, App
Store/Play Store server notifications, Stripe, GitHub, etc.) that checks the signature but not
freshness lets an attacker replay a previously-valid signed payload indefinitely to re-trigger its
effect (re-grant an entitlement, re-fire a paid event).
  1. Find every inbound webhook endpoint.
  2. Confirm the handler rejects requests outside a timestamp tolerance window (not just
     "signature valid, done") — stronger: also dedupes by an already-processed request/event ID.
  3. No webhook receivers at all → `➖ N/A`.

## Email authentication (anti-spoofing/phishing)

**S25 — SPF/DKIM/DMARC.** Not a code check — DNS TXT records on the domain(s) the app sends mail
from. Find the domain from config/docs, then actually query DNS:

```bash
dig TXT <domain> +short                         # SPF: look for "v=spf1 ..."
dig TXT _dmarc.<domain> +short                   # DMARC: look for "v=DMARC1; ..."
dig TXT <selector>._domainkey.<domain> +short    # DKIM — selector is provider-specific
```

(`nslookup -type=TXT <domain>` or PowerShell `Resolve-DnsName -Type TXT` if `dig` isn't
available.) Report exactly what each query returned, including "domain does not resolve at all" if
that's the case — don't silently skip a domain that isn't live yet, report it as the actual
blocker it is. A DMARC record present but `p=none` is a real finding worth a note (monitoring-only,
not enforcing) even though it counts as "present."

## Logging, monitoring & incident response

**S26 — Audit logging for sensitive actions.** Health-data access, granting admin/paid
entitlements, account deletion, and auth events (login, failed login, password change) should be
logged with enough context (who, what, when) to reconstruct an incident — check this actually
exists, not just general request logging.

**S27 — No sensitive data in logs.** Check log statements for accidentally-logged passwords,
tokens, full request/response bodies containing PII or Art. 9 data, or API keys — a very common
and easily-missed leak vector (logs often have weaker access control than the primary DB).

**S28 — Monitoring/alerting exists for anomalies.** Failed-login spikes, 5xx error spikes, unusual
data-export volume — is there any alerting, or would an active incident go unnoticed until a user
complains? Report what exists, even if minimal (health checks alone are not incident detection).

**S29 — Documented incident response process.** Even a short internal doc: who's on call, how a
suspected breach gets escalated, how the 72-hour GDPR Art. 33 notification clock gets started in
practice. Absence is a real gap distinct from the DPIA/breach-notification legal requirement
covered in `gdpr.md`.

## Account & environment hygiene

**S32 — Email address verification.** New accounts should be required to confirm their email
before gaining full access — check the auth provider's actual setting (many providers, e.g.
Supabase, expose this as a dashboard/config toggle rather than app code — check the config file if
self-hosted/CLI, and note explicitly if a cloud-hosted equivalent setting can't be verified from
the repo and needs a dashboard check instead).

**S33 — No real customer data in staging.** A staging/test environment seeded from a production
data dump, or pointed at the same database as production behind an environment flag, exposes real
user data to a lower-security environment (usually broader access, weaker monitoring). Check: does
staging config point at a genuinely separate database/project from production? Is seed data
synthetic/generated rather than a prod dump or CI/CD restore-from-prod step? No staging environment
in this project at all → `➖ N/A`, but note the recommendation for whenever one is created.

## Backups & availability

**S30 — Backups exist and are tested.** A backup configuration existing (managed DB provider
snapshots, etc.) is necessary but not sufficient — has a restore ever actually been tested? An
untested backup is an assumption, not a control.

**S31 — Encryption at rest.** For the primary database and for any particularly sensitive columns
(health notes, progress photos) specifically — some managed platforms encrypt the whole volume by
default (note this explicitly if so) but column-level encryption for the most sensitive fields is
a separate, stronger control worth checking for or recommending.
