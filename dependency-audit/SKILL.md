---
name: dependency-audit
description: Runs a comprehensive third-party dependency health check across every package manager and manifest in a project (npm/yarn/pnpm, pip/poetry/Pipenv, NuGet, Maven/Gradle, Cargo, Go modules, Composer, RubyGems, CocoaPods/SwiftPM, and container base images where relevant), then reports the results as one table (check, area, status, evidence, recommendation). Covers known vulnerabilities/CVEs (native audit tooling per ecosystem), outdated packages (patch/minor vs. major, migration risk), license compliance (copyleft/GPL/AGPL/LGPL conflicts with closed-source distribution, missing/unknown licenses), unused and dead dependencies, lockfile integrity and manifest/lockfile drift, reproducibility of installs, and supply-chain risk signals (unmaintained packages, single-maintainer risk, suspicious/low-download additions, install/postinstall scripts as an attack vector, typosquatting-risk names), plus whether automated update tooling (Dependabot/Renovate or equivalent) is configured. Use this whenever the user asks for a "dependency audit", "dependency check", "dependency health check", "package audit", "abhängigkeitscheck", "paketaudit", "npm audit", "yarn audit", "pnpm audit", "pip-audit", "cargo audit", "govulncheck", "supply chain check", "supply-chain-check", "lieferkettencheck", "lizenzcheck", "lizenzprüfung", "license compliance", "license audit", "copyleft check", "sind unsere pakete aktuell", "sind unsere abhängigkeiten sicher", "sind unsere dependencies sicher", "veraltete abhängigkeiten", "outdated packages", "outdated dependencies", "unused dependencies", "ungenutzte pakete", "tote abhängigkeiten", "dead dependencies", "lockfile check", "lockfile drift", "lockfile integrity", "SBOM", "software bill of materials", "dependabot check", "renovate check", "vulnerable packages", "anfällige pakete", "CVE check for dependencies", or asks about any specific item this covers (unmaintained packages, single maintainer risk, postinstall scripts, typosquatting, manifest/lockfile out of sync) — even if they only name one or two of these and not "dependency audit" explicitly.
---

# Dependency Audit

A structured, evidence-based check of a project's third-party dependencies — vulnerabilities,
staleness, license risk, dead weight, lockfile integrity, and supply-chain hygiene — across every
package manager present in the repo. This is a deeper, broader complement to `cybersecurity-check`
(which covers dependency vulnerabilities as a single line item, S16); this skill is the one to
reach for whenever dependencies themselves — not the rest of the application surface — are the
actual subject of the question.

## Ground rules

- **Evidence or it didn't happen.** Every row needs a concrete pointer: an actual command and its
  actual output, an actual version number, an actual CVE/advisory ID, an actual file:line. Never
  write "dependencies look fine," "packages appear up to date," or "no vulnerabilities found" as a
  standalone claim — report what tool ran, against what manifest, and what it printed.
- **A ✅ means "the audit tool ran and reported clean" — not "there are definitely no
  vulnerabilities."** Audit tools have real false-negative rates, especially for vulnerabilities
  disclosed in the last few days (the advisory database hasn't caught up yet) and for
  transitive/indirect dependencies some tools don't fully walk. Say this caveat once, up front, so
  the user doesn't over-trust a clean run. Phrase findings as "`npm audit` reported 0
  vulnerabilities as of <date>," not "this project has no vulnerable dependencies."
- **Don't invent scope you can't check, and don't silently drop scope either.** If an ecosystem's
  native tooling isn't installed, isn't available in this environment, or genuinely has no audit
  equivalent (e.g. no CVE database integration for a given package manager), mark that row
  `➖ N/A` with a one-line reason — "no lockfile present," "cargo not installed in this
  environment," "Dart/Flutter has no built-in CVE audit — cross-checked outdated packages
  manually instead." Every check gets a row; none are quietly skipped, and no ecosystem present in
  the repo is skipped because its tool is less familiar than npm/pip.
- **Be genuinely thorough in a monorepo.** Check every manifest and lockfile in the project, not
  just the root — a `packages/*/package.json`, a `services/*/requirements.txt`, a nested
  `pom.xml`, a mobile app's separate `Podfile`/`pubspec.yaml` sitting next to a backend's
  `package.json`. A vulnerability or copyleft license three folders down is exactly as reportable
  as one at the root, and it's also exactly the one that's easy to miss by only running the audit
  tool once at the top level.
- **Distinguish "audited" from "not applicable" from "couldn't check."** A package manager with
  zero dependencies declared, a manifest with no lockfile at all, and a tool that isn't installed
  in this environment are three different findings — don't collapse them into one vague status.

## Workflow

1. **Identify every package manager and manifest in the project.** Walk the whole tree (not just
   the root) for: `package.json`/`package-lock.json`/`yarn.lock`/`pnpm-lock.yaml`,
   `requirements*.txt`/`Pipfile`/`Pipfile.lock`/`pyproject.toml`/`poetry.lock`,
   `*.csproj`/`packages.config`/`packages.lock.json`, `pom.xml`/`build.gradle`/`build.gradle.kts`
   (+ `gradle.lockfile`), `Cargo.toml`/`Cargo.lock`, `go.mod`/`go.sum`,
   `composer.json`/`composer.lock`, `Gemfile`/`Gemfile.lock`, `Podfile`/`Podfile.lock`,
   `Package.swift`/`Package.resolved`, `pubspec.yaml`/`pubspec.lock`, and any `Dockerfile` pinning
   OS/base-image packages. A monorepo can legitimately have several of these side by side — list
   them all before running anything.
2. **Run each ecosystem's native audit and outdated-listing tooling** against every manifest found
   (see the checks below for the actual commands per ecosystem). Capture real output; don't
   summarize from memory of what a tool "usually" reports.
3. **Cross-reference license metadata** for direct dependencies (and transitive ones where the
   tooling makes it easy) against the project's actual distribution model — ask or infer whether
   the project is closed-source/proprietary, since that's what determines whether a copyleft
   license is actually a problem.
4. **Check for unused dependencies, lockfile drift, and supply-chain risk signals** per the checks
   below.
5. **Report as one table**, ❌ findings ordered with critical/high-severity CVEs first, followed by
   the prioritized punch list and needs-review list.

## Known vulnerabilities

**D1 — Run the correct native audit tool per detected ecosystem; report actual findings.** Never
silently skip an ecosystem that's present in the repo, and never substitute one ecosystem's result
for another's. For each finding, capture: package name, currently-installed version, CVE/advisory
ID, severity, and the fixed-in version if one exists.

- **npm/yarn/pnpm:** `npm audit --json` (or `yarn audit`, `pnpm audit`) — run once per lockfile in
  a monorepo with independent workspaces; a single root run can miss a workspace with its own
  lockfile.
- **Python:** `pip-audit` (preferred, uses OSV) or `safety check` against
  `requirements.txt`/`poetry.lock`/`Pipfile.lock`. `pip list --outdated` alone is not a
  vulnerability check — don't conflate the two.
- **.NET:** `dotnet list package --vulnerable --include-transitive`.
- **Ruby:** `bundle audit check --update`.
- **Rust:** `cargo audit`.
- **Go:** `govulncheck ./...` (preferred — call-graph aware, fewer false positives than a bare
  manifest scan) or at minimum `go list -m all | nancy sleuth`.
- **PHP:** `composer audit`.
- **Java (Maven):** `mvn org.owasp:dependency-check-maven:check`.
- **Java (Gradle):** the OWASP `dependencyCheckAnalyze` task (requires the
  `org.owasp.dependencycheck` plugin — check `build.gradle` for it; if absent, note that no native
  vulnerability scanning is configured rather than assuming it ran).
- **Swift/CocoaPods:** no widely-adopted native CVE audit tool exists for these ecosystems as of
  this writing — mark `➖ N/A` with that reason and note that a manual check against GitHub
  Security Advisories for high-risk direct dependencies is the fallback.
- **Dart/Flutter:** `flutter pub outdated` has no built-in CVE audit either — cross-check flagged
  outdated packages against a CVE database (OSV, GitHub Advisories) manually and say so.
- **Container/base images:** if a `Dockerfile` is present, `trivy image <tag>` or `docker scout cves
  <tag>` against the built or pulled image, covering OS packages the language-level tools don't
  see.

## Outdated packages

**D2 — Patch/minor drift.** For each ecosystem's outdated-listing command (`npm outdated`/`yarn
outdated`/`pnpm outdated`, `pip list --outdated`, `dotnet list package --outdated`, `cargo
outdated`, `go list -m -u all`, `composer outdated`, `bundle outdated`, `mvn
versions:display-dependency-updates`, `gradle dependencyUpdates`), report how many direct
dependencies are behind by patch or minor version only. These are usually low-risk to update
(no breaking API changes expected under semver) — call that out explicitly so the user can
batch-update them with less scrutiny than majors.

**D3 — Major-version drift.** Separately list every direct dependency that is one or more major
versions behind. For each, note whether a migration guide/changelog exists and flag it as needing
a scoped migration check before updating — don't recommend a blanket "just update everything,"
since a major bump can be a breaking change even when the audit tool has no vulnerability to
report against the old version.

## License compliance

**D4 — Direct (and where feasible transitive) dependency licenses.** Enumerate the license of
every direct dependency using the ecosystem's metadata rather than guessing from a package's
reputation:

- npm/yarn/pnpm: `npx license-checker --summary` (direct + transitive).
- Python: `pip-licenses`.
- .NET: `dotnet-project-licenses` or inspect `.nuspec` license metadata.
- Ruby: `license_finder`.
- Rust: `cargo license`.
- Go: `go-licenses report ./...`.
- Java: the `license-maven-plugin` report or Gradle `licenseReport` task.

**D5 — Flag copyleft and missing licenses.** Cross-reference the license list against the
project's actual distribution model (ask if unclear, or infer from context — a proprietary SaaS
backend vs. an OSS library vs. distributed desktop/mobile binaries have different exposure).
Flag: any GPL/AGPL/LGPL-licensed dependency (AGPL is the highest-risk for a SaaS product
specifically, since its copyleft trigger includes network use, not just distribution — call this
distinction out rather than treating all copyleft licenses as equally risky); any package with no
discoverable license field or `UNKNOWN`/`UNLICENSED` in the audit output, which is a legal
question mark even without being explicitly copyleft. A finding here is `⚠️` when it needs a
human/legal judgment call about actual usage (e.g. "is this LGPL library dynamically linked or
statically bundled") rather than a clear-cut `❌`.

## Unused dependencies

**D6 — Declared but never imported.** Packages listed in the manifest that no source file actually
imports/requires are dead weight and unreviewed attack surface (every declared dependency is code
that runs at install time and potentially at runtime, whether or not the project uses it).

- JS/TS: `npx depcheck` (also flags devDependencies used only in scripts vs. genuinely unused).
- Python: `deptry` or compare `pip list` against actual `import` statements with a tool like
  `pip-check-reqs`.
- Rust: `cargo +nightly udeps` (or `cargo machete` for a stable-toolchain alternative).
- Go: `go mod tidy -diff` shows what would be removed without actually modifying `go.mod`.
- PHP: `composer-unused`.
- .NET/Java: no single dominant tool — grep for the namespace/package's actual usage across the
  source tree and note this was done manually if no static tool is set up; don't skip the check
  just because it's more manual here.

Report each candidate with a caveat: a dependency imported only via reflection, dynamic
`require()`, a build plugin, or a peerDependency satisfying another package's requirement can look
"unused" to a static tool while still being necessary — verify with a source grep before listing
something as a confirmed removal candidate rather than trusting the tool's output blindly.

## Lockfile integrity & reproducibility

**D7 — Lockfile committed and present per manifest.** Every manifest that supports a lockfile
should have one committed (`package-lock.json`/`yarn.lock`/`pnpm-lock.yaml`, `poetry.lock`,
`Pipfile.lock`, `packages.lock.json`, `Cargo.lock`, `go.sum`, `composer.lock`, `Gemfile.lock`,
`Podfile.lock`/`Package.resolved`). No lockfile at all means every install can silently resolve
different transitive versions — a real reproducibility gap, not a style preference. `Cargo.lock`
for a library crate is conventionally *not* committed (consumers resolve their own) — don't flag
that as missing; do flag it for a binary/application crate.

**D8 — Manifest/lockfile drift.** Confirm the lockfile is actually in sync with the manifest, not
stale from a hand-edited `package.json`/`requirements.txt` that was never followed by a fresh
install. Run the ecosystem's strict/CI-mode install and check it doesn't silently rewrite the
lockfile: `npm ci` (fails on drift, unlike `npm install`), `pnpm install --frozen-lockfile`, `yarn
install --immutable`, `poetry check --lock`, `bundle check`. A tool that "just updates the
lockfile to match" when run is a sign nothing enforces sync today.

**D9 — Pinning strategy and CI enforcement.** Note whether the manifest uses exact versions or
ranges (`^`/`~`/`*`) and whether that looks intentional (a library commonly uses ranges
deliberately; an application typically benefits from exact pins plus a lockfile). Check CI config
(`.github/workflows/*`, `azure-pipelines.yml`, etc.) for a step that runs the frozen/strict install
mode above — if CI only runs a plain `install`, lockfile drift can merge silently and this is worth
flagging even absent any current drift.

## Supply-chain risk signals

**D10 — Unmaintained packages.** For security-sensitive or heavily-relied-upon direct
dependencies, check last-publish date and repository activity: `npm view <pkg> time` (or the
registry page), the GitHub repo's last commit/release date, and number of listed maintainers
(`npm view <pkg> maintainers`). A single-maintainer package with no commits in 1-2+ years isn't
automatically disqualifying, but it's a real risk signal worth surfacing, especially if it sits on
a critical path (auth, crypto, payment).

**D11 — Suspicious or low-adoption recent additions.** For dependencies added recently (check
lockfile diff history / git blame on the manifest if available) that perform trivial
functionality (e.g. a one-function left-pad-style utility), check download counts and package age
on the registry — a brand-new, near-zero-download package pulled in for something the project
could trivially implement in a few lines is exactly the profile of both an unnecessary dependency
(D6-adjacent) and a supply-chain risk (an attacker-controlled or soon-to-be-abandoned package).

**D12 — Install/postinstall scripts in dependencies.** Lifecycle scripts
(`preinstall`/`install`/`postinstall`) in a dependency's own `package.json` run arbitrary code on
every install and are a well-documented real-world supply-chain attack vector (credential
exfiltration, cryptominers, backdoors shipped via a compromised maintainer account or a malicious
transitive dependency). Enumerate which installed packages declare one:

```bash
npm query ":attr(scripts, [install]), :attr(scripts, [postinstall]), :attr(scripts, [preinstall])"
# or, without npm query support:
find node_modules -maxdepth 2 -name package.json -exec grep -l '"postinstall"\|"preinstall"' {} \;
```

Report the list — don't assess each script's intent yourself beyond a quick read; the deliverable
is "these N packages run install-time scripts, here they are, a human should look at what they
actually do," not a definitive verdict per script. Equivalent for other ecosystems: pip has no
universal install-hook mechanism for pure wheels but does for source builds with `setup.py`
(`python_requires`/build_ext code) — check any dependency still requiring a source build; Ruby
gems can define `extconf.rb` native-extension build steps similarly.

**D13 — Typosquatting-risk names.** For dependencies with names close to a much more popular
package (`reqeust` vs `request`, `crossenv` vs `cross-env`, an unscoped name mimicking a normally
`@scope/`-namespaced package), flag for manual double-check even without concrete evidence of
malice — this is a "worth a human glance" flag, not a definitive finding, and should be marked
`⚠️` rather than `❌` unless something concrete (mismatched publisher, near-zero downloads despite
an old-looking version number) is also found.

## Update process

**D14 — Automated update tooling.** Check for `.github/dependabot.yml`, `renovate.json`/
`renovate.json5`/a `renovate` key in `package.json`, or an equivalent scheduled-update bot/CI job.
Note its actual configured scope (which ecosystems/paths it covers — a Dependabot config scoped
only to `/` in a monorepo with dependencies in `/services/api` isn't covering those) and update
frequency. No automation found → report that dependency updates are fully manual/ad hoc, which is
itself a finding worth a recommendation, not a silent gap.

## Output format

Start with 2-4 sentences: which package managers/manifests were found in the project (list them —
this is the scope statement), which native audit/outdated/license/unused-dependency tools were
actually run against each, anything that couldn't be checked (tool not installed, no network
access to query a registry, no lockfile to diff), and the caveat from Ground rules that a clean
audit-tool run means "reported clean," not "provably free of vulnerabilities."

Then ALWAYS use this exact table — one row per check, per ecosystem/manifest where a check applies
to more than one (e.g. D1 gets one row per package manager found), none omitted:

| # | Check | Bereich/Area | Status | Befund/Evidence | Empfehlung/Recommendation |
|---|---|---|---|---|---|
| 1 | ... | Vulnerabilities/Outdated/License/Unused/Lockfile/Supply Chain/Process | ✅/❌/⚠️/➖ | actual command + output, actual version/CVE, or file:line | only if not ✅ |

(Match the table's actual language to the conversation's language — column names above are
illustrative. Keep "Befund"/"Evidence" concrete: the command you ran and what it printed, an
actual CVE ID and severity, an actual version number, or "not found — searched X, Y, Z.")

Status legend:
- ✅ Pass — audit/outdated/license/unused-dependency tool ran and reported clean/expected for this
  check
- ❌ Fail — checked, and a concrete problem was found (a CVE, a copyleft conflict, confirmed
  lockfile drift, a confirmed-unused dependency, a missing lockfile)
- ⚠️ Needs manual/human review — a license-usage judgment call, a typosquatting-risk name worth a
  second look, an install-script list a human should read, an "unmaintained" signal that isn't
  automatically disqualifying
- ➖ N/A — this check's scope doesn't apply here (no such ecosystem present, no native tool exists
  for this ecosystem, no lockfile-supporting manifest, library crate where a lockfile isn't
  conventionally committed) — state why in one clause

End with a **prioritized punch list**: every ❌, ordered with critical/high-severity CVEs first,
then other confirmed failures (missing lockfile, confirmed license conflict, confirmed drift),
each with the one-line fix (upgrade to version X, remove the unused dependency, commit the
lockfile). Follow it with a **needs-review list**: every ⚠️, since those need a human decision
(often a legal one for licenses) rather than a code change.
