---
name: ci-cd-pipeline-review
description: Reviews a project's CI/CD pipeline mechanics and deployment practice for reliability and safety — not the security content of the application itself. Covers build reproducibility (pinned tool/runtime versions, pinned third-party CI actions/plugins vs. mutable tags), test gating (does a test failure actually block merge/deploy, are required checks enforced at branch-protection level), secrets handling within the pipeline configuration (platform secret store vs. hardcoded, log-printing risk, scoping, fork-PR exposure), CI job permissions and production deployment approval gates, deployment strategy and safety (blue-green/canary/rolling vs. hard cutover, rollback path, post-deploy health checks/smoke tests, feature flags), environment parity and promotion (build-once-promote-many vs. rebuild-per-environment, config/artifact separation), artifact provenance and integrity (signing/checksums, commit-to-deploy traceability, pinned minimal base images), pipeline observability (failure notifications, run-time/failure-rate tracking), and pipeline cost/efficiency (caching, redundant jobs, overly broad triggers) across whichever platform(s) are in use (GitHub Actions, GitLab CI, Jenkins, Azure DevOps, CircleCI, Bitbucket Pipelines, Travis CI). Use this whenever the user asks for a "CI/CD pipeline review", "CI/CD audit", "pipeline review", "pipeline audit", "pipeline check", "build pipeline check", "deployment pipeline review", "deployment process review", "release process audit", "release pipeline check", "DevOps pipeline review", "pipeline reliability check", "pipeline safety check", "deployment readiness check", "rollback readiness", "ci pipeline check", "cd pipeline check", "github actions review", "github actions audit", "gitlab ci review", "jenkins pipeline review", "azure devops pipeline review", "circleci review", "pipeline-audit", "pipeline-check", "deployment-prozess prüfen", "deployment prüfen", "bereitstellungsprozess prüfen", "build-pipeline check", "build-pipeline prüfen", "ci/cd prüfen", "ci/cd check", "pipeline-prüfung", "release-prozess prüfen", "rollout prüfen", "auslieferungsprozess prüfen", "deployment-sicherheit", "ist unsere pipeline sicher", "ist unser deployment sicher", "können wir sicher deployen", "rollback-fähigkeit prüfen", or asks about any specific item this covers (pinned actions, mutable action tags, allow_failure, continue-on-error, required status checks, branch protection, secrets in build logs, fork PR secrets, least-privilege CI token, manual approval gate, canary deployment, blue-green deployment, build-once-promote-many, artifact signing, SBOM for a build artifact, image provenance, pipeline caching, flaky pipeline triggers) — even if they only name one or two of these and not "pipeline review" explicitly.
---

# CI/CD Pipeline Review

A structured, evidence-based review of a project's CI/CD pipeline **mechanics and deployment
safety** — not a code audit and not an application-security review. It investigates the actual
pipeline configuration files across whatever platform(s) are in use, and reports one table the
user can act on.

## Ground rules

- **Evidence or it didn't happen.** Every row needs a concrete pointer — a pipeline YAML/config
  `file:line`, an actual job/step definition, a grep match, or (for anything that lives outside
  the repo, like branch-protection settings) an explicit note that it needs a platform/dashboard
  check. Never write "the pipeline looks solid," "deployment seems safe," or "build practices are
  standard" as a standalone claim — quote the actual step, trigger, or condition you found.
- **Scope boundary — say it once, up front, and hold it for every row.** This skill reviews
  pipeline *mechanics and deployment safety*: does the pipeline build reproducibly, does it gate
  correctly, is a deployment recoverable, are credentials handled safely *within the pipeline
  config itself*. It does **not** review:
  - the security content of the application code (injection, auth, access control, headers,
    SSRF, etc.) — that is `cybersecurity-check`'s job;
  - dependency CVEs/license/staleness of the packages the app itself uses — that is
    `dependency-audit`'s job.
  The one place these overlap: secrets *handling inside the pipeline configuration* — a secret
  hardcoded in a workflow YAML, echoed to a build log, or leaked to a fork PR — is squarely a
  pipeline-mechanics problem and stays in scope here even though "secrets" also appears in
  `cybersecurity-check` (which looks at secrets in *application* code/config, S15). If a finding
  belongs to one of the other two skills, say so and point there rather than re-doing or skipping
  it silently.
- **Don't invent scope you can't check, and don't silently drop scope either.** No CD/deployment
  automation exists yet, only CI → every deployment-strategy, rollback, and promotion check
  (CI11–CI16) gets `➖ N/A` with that one-line reason, not a quiet omission. No container images
  built → artifact/image checks (CI17, CI19) are `➖ N/A`. Branch-protection settings can't be
  read from the repo alone → mark `⚠️` and say exactly what would need a dashboard/API check
  (`gh api repos/<org>/<repo>/branches/<branch>/protection` if `gh` is available and authorized,
  otherwise a manual UI check). Every check in this skill gets a row; none are quietly skipped.
- **Read the actual job/stage definitions — never assume from a README or project description.**
  A README claiming "we do blue-green deployments" or "tests are required to merge" is a claim,
  not evidence; find the actual step that implements it (or find that no such step exists).
- **Be genuinely thorough.** A pipeline that "runs and goes green" is not the same as a pipeline
  that's safe to depend on — the checks that take the most digging (fork-PR secret exposure,
  whether a required check is actually enforced at the branch-protection level, whether rollback
  is truly automated vs. a documented manual runbook) are usually exactly the ones worth getting
  right, not the ones to wave through because the YAML "looks fine at a glance."

## Workflow

1. **Identify every CI/CD platform in use and locate every pipeline config file.** Don't assume a
   single platform — a repo can run GitHub Actions for CI and a separate Jenkins/Azure DevOps
   pipeline for deployment. Look for:
   - GitHub Actions: `.github/workflows/*.yml`/`*.yaml`
   - GitLab CI: `.gitlab-ci.yml` plus any `include:`-ed files
   - Jenkins: `Jenkinsfile`, `Jenkinsfile.*`, or a `jenkins/` folder
   - Azure DevOps: `azure-pipelines.yml`, `azure-pipelines/*.yml`, `*.pipeline.yml`
   - CircleCI: `.circleci/config.yml`
   - Bitbucket Pipelines: `bitbucket-pipelines.yml`
   - Travis CI: `.travis.yml`
   - Any deploy-only scripts invoked by the above (`deploy.sh`, Helm charts, Terraform/Ansible
     invoked from a pipeline step, a separate `Dockerfile` used for the build)
2. **Read the actual job/stage/step definitions in each file** rather than inferring behavior from
   file names or a README's description of "how deployment works." Note which pipeline(s) run on
   which trigger (PR, push to main, tag, manual dispatch, schedule) — this matters for several
   checks below (test gating, fork-PR secrets, deploy triggers).
3. **Report the platform(s) and files found as one table** before diving into individual files, so
   the scope of the review is explicit up front.
4. **Work through every check below** (CI1–CI24), grouped by theme, marking each ✅/❌/⚠️/➖ with
   concrete evidence.
5. **Report as one table**, ❌ findings ordered with the riskiest first (an unrecoverable
   deployment or a fork-PR secret leak outranks a missing cache), followed by the prioritized
   punch list and needs-review list.

## Build reproducibility & supply-chain pinning

**CI1 — Build/runtime versions are pinned, not floating.** A step that resolves to `latest` at
run time (a Docker base image tag, a language-runtime setup action's version input, an
unpinned package manager pulled at build time) can pass today and silently break tomorrow when
upstream publishes a new `latest` — or worse, quietly change behavior without any code change on
this side. Check:
- `Dockerfile`/build-stage images: `FROM node:latest`, `FROM python:3`, `FROM ubuntu` (no tag at
  all defaults to `latest`) — should be an exact tag, ideally with a digest (`FROM
  node:20.11.1-bookworm-slim@sha256:...`).
- Language/runtime setup steps: GitHub Actions `actions/setup-node@vX` with `node-version: 'latest'`
  or a bare major (`'20'` floats across minors — decide if that's intentional or should be exact);
  GitLab `image: node:latest`; Jenkins `tool` steps with no version pinned.
- Any `apt-get install`/`pip install`/`npm install -g` of a build-time tool with no version pin in
  the pipeline script itself (separate from the project's own dependency lockfile, which
  `dependency-audit` covers).

**CI2 — Third-party CI actions/plugins are pinned to a commit SHA or exact version, not a mutable
tag or branch.** This is a supply-chain risk distinct from CI1: an action referenced by a mutable
tag (`uses: some/action@v1` or `@main`) can have that tag's target moved by the action's
maintainer (or an attacker who compromises their account) to point at different, potentially
malicious code — the pipeline then runs it automatically on the next trigger with no review.
Check:
```bash
grep -rn "uses:" .github/workflows/           # GitHub Actions — look for @<branch> or @v<major-only>
grep -rn "^\s*- " .gitlab-ci.yml              # GitLab CI includes/templates from external refs
```
- Safe: `uses: actions/checkout@8f4b7f84...` (full 40-char commit SHA) or, as a weaker but common
  middle ground, an exact semver tag from a maintainer with a track record (`@v4.1.7`) — flag
  floating major-only tags (`@v4`, `@main`, `@master`) as the actual finding, since those are
  exactly the mutable references the risk describes.
- Same logic for GitLab CI `include: remote:`/`include: project:` refs pinned to a branch instead
  of a tag/commit, and for Jenkins shared-library `@Library('foo@master')` declarations.
- First-party actions from the platform vendor itself (`actions/checkout`, `actions/setup-node`)
  are lower risk but still worth pinning for reproducibility — note them as lower severity than a
  small, less-established third-party action pinned to a floating tag.

## Test gating

**CI3 — A test failure actually blocks merge/deploy.** A test job that runs but doesn't gate
anything is a false sense of safety — worse than no test job, because it looks like a safeguard.
Check for the failure-tolerant patterns that silently defeat gating:
```bash
grep -rn "continue-on-error" .github/workflows/
grep -rn "allow_failure" .gitlab-ci.yml
grep -rn "// ignoreFailures\|catchError\|unstable(" Jenkinsfile
```
- GitHub Actions: `continue-on-error: true` on the test step/job means the job reports success
  regardless of the test result, and a merge queue or required-check based on that job's overall
  status won't block. Also check the workflow trigger doesn't run tests only on `workflow_dispatch`
  (manual) while `push`/`pull_request` skip them.
- GitLab CI: `allow_failure: true` on the test job has the same effect on pipeline status.
- Jenkins: a `catchError`/`unstable()` wrapper around the test stage that keeps the build "green"
  regardless of test outcome; a post-build step that doesn't fail the build on test-report
  failures.
- Confirm the branch-protection/required-checks list (see CI4) actually *names* the test job — a
  pipeline can have a passing/failing test stage that nothing downstream ever consults.

**CI4 — Required-checks enforcement lives at the branch-protection/repo-settings level, not just
in the pipeline file.** A workflow file defining a job is not the same as that job being *required*
to pass before merge — that's a separate, platform-level setting. Check what's verifiable from the
repo, and mark the rest for a dashboard/API check:
```bash
gh api repos/<org>/<repo>/branches/<branch>/protection --jq '.required_status_checks'
```
- If `gh` isn't available/authorized, or the platform is GitLab/Azure DevOps/Bitbucket (each has
  its own equivalent: GitLab protected-branch "merge checks," Azure DevOps branch policies,
  Bitbucket branch restrictions), mark `⚠️ needs manual/dashboard check` and say exactly what to
  look up, rather than assuming enforcement exists (or doesn't) from the pipeline file alone.
- If the required-checks list is retrievable and it omits the actual test job's name (or lists an
  outdated job name after a rename), that's a concrete `❌` — the protection exists but doesn't
  cover what the user thinks it covers.

## Secrets handling in CI

**CI5 — Secrets come from the platform's secret store, not hardcoded in the pipeline YAML.**
```bash
grep -rniE "(api[_-]?key|password|secret|token|conn(ection)?string)\s*[:=]\s*['\"][A-Za-z0-9+/=_-]{8,}" \
  .github/workflows/ .gitlab-ci.yml Jenkinsfile azure-pipelines.yml .circleci/config.yml
```
Hardcoded values fail outright; references to the platform's own secret mechanism pass —
`${{ secrets.X }}` (GitHub Actions), `$X` sourced from a masked/protected CI/CD variable (GitLab),
`credentials('x')` (Jenkins), `$(X)` backed by a variable group/Key Vault link (Azure DevOps).
Check git history too if asked to go that deep — a secret rotated after being committed is a
different (closed) finding than one still live in the current file.

**CI6 — No risk of a secret being printed to build logs.** Even correctly-sourced secrets can leak
if a step echoes them or runs a command in verbose/debug mode that logs its own arguments:
```bash
grep -rn "echo.*\$\(SECRET\|TOKEN\|PASSWORD\|API_KEY\)" .github/workflows/ .gitlab-ci.yml Jenkinsfile
grep -rniE "curl .*-v |set -x|--verbose|ACTIONS_STEP_DEBUG" .github/workflows/
```
Look for: a secret echoed directly for "debugging"; a shell script run with `set -x` (which prints
every command including any inline secret argument) in a step that also handles a credential; a
secret passed as a bare CLI argument (`--password $DB_PASSWORD`) rather than via stdin/an env var
the tool reads internally, since a CLI argument can also leak via process listings on a shared
runner, not just logs. Most CI platforms auto-mask registered secrets in log output — check
whether the secret is actually registered as a masked secret/variable (not just an env var set
from a plain pipeline variable, which typically isn't masked).

**CI7 — Secrets are scoped to the minimum jobs/environments that need them.** A secret usable by
every job in the pipeline (declared pipeline-wide) is available to steps that don't need it,
widening blast radius if any one of those steps is compromised or misconfigured. Check whether
secret access is scoped per-job/per-environment (GitHub Actions `environment:` protection rules +
environment-scoped secrets; GitLab CI/CD variables scoped to specific protected
environments/branches) rather than declared at the workflow/pipeline-wide level and used
everywhere by default.

**CI8 — Secrets are withheld from pull-request-triggered workflows originating from forks.** This
is a well-documented privilege-escalation vector: a `pull_request` (not `pull_request_target`)
trigger on GitHub Actions from a fork PR normally runs with a read-only token and no repo secrets
by default — but a workflow that uses `pull_request_target` (which *does* get secrets and a
write-scoped token) while also checking out and running the PR's own code is exploitable, since a
malicious fork PR can modify the workflow/build script to exfiltrate those secrets.
```bash
grep -rln "pull_request_target" .github/workflows/
```
For every match, check whether the job also does `actions/checkout` with `ref:
github.event.pull_request.head.sha` (or similar) — checking out and executing the fork's own code
under a secret-bearing trigger is the actual vulnerable pattern; `pull_request_target` used only to
label/comment on the PR without executing its code is lower risk. GitLab's equivalent: check
whether CI/CD variables are marked "protected" (unavailable to non-protected-branch pipelines,
which includes most fork MRs) and whether "Run pipelines for external MRs" combined with variable
exposure has been considered. No fork-based PR workflow at all → `➖ N/A`.

## Permissions & access

**CI9 — CI job tokens/service accounts use least privilege, not broad default write/admin access.**
```bash
grep -rn "permissions:" .github/workflows/
```
GitHub Actions: since late 2023 new repos default the `GITHUB_TOKEN` to read-only unless a workflow
opts into more, but older repos and workflows can still run with broad default write permissions —
check for an explicit top-level `permissions:` block scoped to only what each job needs (e.g.
`contents: read` for a build job, `contents: write` only for a job that actually pushes tags/
releases) rather than every job running with the same broad grant regardless of what it does. For
Jenkins/Azure DevOps/GitLab, check what service-account/credential the pipeline runs as and whether
it's a shared, broadly-privileged account or scoped per pipeline/environment.

**CI10 — A human approval gate exists before production deployment, where that's the intended
process.** Check whether every merge to main auto-deploys to production with no checkpoint, or
whether there's a manual approval step (GitHub Actions `environment:` with required reviewers,
Azure DevOps pre-deployment approvals, GitLab `when: manual` job, Jenkins `input` step) gating the
production deploy job specifically. This is not inherently a failure if full continuous deployment
to prod is the team's deliberate, documented practice — state which case applies rather than
defaulting to "no gate = fail"; mark `⚠️` if it's unclear whether the absence of a gate is
intentional and ask/note it explicitly.

## Deployment strategy & safety

*No CD/deployment automation in this repo, only CI → mark CI11–CI14 `➖ N/A` with that reason
rather than skipping them.*

**CI11 — Deployment strategy is identified.** Determine whether the pipeline performs blue-green,
canary, or rolling deployment (gradual traffic shift, ability to compare old/new before full
cutover) versus a hard cutover (all instances/traffic replaced at once with no intermediate state).
Evidence: the actual deploy step/tool invocation (a Kubernetes rolling-update strategy in a
manifest, a canary-percentage config for a service mesh/load balancer, a blue-green swap script) —
not a claim in documentation.

**CI12 — Rollback path is automated or fast, not a manual multi-step recovery.** If a deployment
fails health checks, can the pipeline (or the deployment platform it drives) revert automatically
or via a single command/trigger, or does recovery require a person to manually reconstruct the
previous state (re-deploy an old artifact by hand, manually edit infrastructure)? A "redeploy the
previous git tag" pipeline job that exists and is tested counts as fast-manual and is a materially
different finding than no documented rollback procedure at all.

**CI13 — Post-deploy health checks/smoke tests run automatically before traffic is fully shifted.**
Check for an automated post-deploy verification step (hitting a health endpoint, running a smoke
test suite) gating full traffic cutover in a canary/blue-green setup, or gating "deployment marked
successful" in a simpler setup — versus a deploy step that's considered done the moment the new
version starts, with no verification it's actually healthy before users see it.

**CI14 — A feature-flag mechanism decouples deployment from release for risky changes.** Not every
project needs this, but check whether one exists (LaunchDarkly, Unleash, a config-driven flag
system, even a simple env-var toggle) for gating new behavior independently of the deploy itself —
absence isn't automatically a failure for a low-risk/low-traffic project, but note it as a gap for
anything doing frequent production deploys of user-facing risk.

## Environment parity & promotion

*No CD/deployment automation, or only a single environment → mark CI15–CI16 `➖ N/A` with that
reason.*

**CI15 — Build-once, promote-many: the same artifact tested in staging is the one deployed to
production.** Check whether the pipeline builds one artifact/image and promotes it unchanged
through environments (tag/hash carried forward, same binary), versus rebuilding from source
separately per environment — the latter risks environment-specific drift where what was tested in
staging isn't bit-for-bit what ships to production (different dependency resolution at build time,
a different compiler/toolchain version picked up between runs, etc.). Evidence: does the
production-deploy job reference the exact artifact/image digest produced by the staging build job,
or does it invoke a fresh build step of its own?

**CI16 — Environment-specific configuration is separated from the build artifact, not baked in
per-environment at build time.** Check whether environment differences (API endpoints, feature
flags, resource limits) are injected at deploy/runtime (env vars, a mounted config, a
platform-native config service) rather than requiring a separate build per environment with
different values compiled/bundled in — the latter is what makes build-once-promote-many (CI15)
impossible even if attempted.

## Artifact provenance & integrity

*No container images or published build artifacts → mark CI17 and CI19 `➖ N/A` with that reason.*

**CI17 — Build artifacts/container images are signed or checksummed.** Check for image signing
(`cosign sign`/`cosign verify` in the pipeline, Docker Content Trust, a signed provenance
attestation via SLSA/`slsa-github-generator`) or at minimum a published checksum/digest that a
deploy step verifies before use, versus an artifact pulled and deployed by a mutable tag alone with
no integrity check.

**CI18 — Traceability from a deployed artifact back to the exact source commit and pipeline run
that produced it.** For incident response, check whether the deployed artifact carries metadata
(an image label, a build-info file, a version string embedded at build time) recording the git
commit SHA and CI run ID/URL that built it — versus an artifact/version number with no link back to
"what code, built by which run, is actually running in production right now."

**CI19 — Container images are built from a pinned, minimal base image.** Distinct from CI1's
"is it pinned at all" — this also checks *what* it's pinned to: a full general-purpose OS image
(`ubuntu:22.04` as a runtime base) carries far more attack surface and larger size than a
minimal/distroless equivalent (`gcr.io/distroless/...`, an `-alpine`/`-slim` variant appropriate to
the runtime). Note the base image and flag an unnecessarily broad one as a lower-severity
improvement, separate from the pinning check itself.

## Pipeline observability

**CI20 — A pipeline failure notifies the right people/channel promptly.** Check for a configured
notification step/integration (Slack/Teams/email webhook on failure, a platform-native
notification setting) versus relying on someone noticing a red X in the platform's UI. Look for
`on: failure`/`if: failure()` steps calling a notification webhook, or a platform-level
notification rule — absence means a broken pipeline (especially a scheduled/off-hours one) can go
unnoticed for a long time.

**CI21 — Pipeline run times and failure rates are tracked over time.** Check for any dashboard,
CI-native insights (GitHub Actions' own workflow-run history/insights, GitLab CI/CD analytics), or
external tooling (a DORA-metrics dashboard, a build-times tracking tool) — versus no visibility
beyond each individual run's own log, which makes a slow degradation (creeping build times, a
flaky test creeping toward "always fails") invisible until someone happens to notice.

## Cost & efficiency of the pipeline itself

**CI22 — Dependency/build-layer caching is configured.** Check for a caching step
(`actions/cache`, GitLab `cache:` key, Docker layer caching / BuildKit cache mounts, a Gradle/Maven
local-repo cache) keyed appropriately (by lockfile hash, not just a static key that never
invalidates or a per-run key that never hits). No caching at all means every run reinstalls/
rebuilds from scratch — a real, measurable cost and time waste, not just a style nit.

**CI23 — No redundant jobs doing the same work multiple times.** Check for duplicate work across
jobs/workflows — e.g. both a CI workflow and a separate release workflow independently rebuilding
and re-testing the same commit with no artifact reuse between them, or multiple jobs each
redundantly re-running a lint/build step that could run once and be shared/cached.

**CI24 — Trigger scope is appropriately narrow.** Check the `on:`/`rules:`/trigger configuration:
does the full pipeline (including expensive integration tests, deploy-adjacent steps) run on every
branch push regardless of relevance, or is it scoped (path filters so a docs-only change doesn't
trigger a full backend test suite, draft-PR exclusion, a lighter job set for non-main branches)?
Flag a pipeline with no path/branch filtering at all where the repo clearly has independent
areas (e.g. a monorepo) that don't need to trigger each other's full suite.

## Output format

Start with 3-4 sentences: which CI/CD platform(s) and pipeline config files were found (list them —
this is the scope statement), which checks could be fully verified from the repo alone vs. which
need a platform/dashboard check this investigation couldn't reach (branch protection settings,
environment-approval configuration, external notification/analytics tooling), and the scope
boundary from Ground rules (pipeline mechanics and deployment safety only — not application
security or dependency CVEs, which belong to `cybersecurity-check`/`dependency-audit`
respectively).

Then ALWAYS use this exact table — one row per check, none omitted:

| # | Check | Bereich/Area | Status | Befund/Evidence | Empfehlung/Recommendation |
|---|---|---|---|---|---|
| 1 | ... | Build/Test Gating/Secrets/Permissions/Deployment/Environment Parity/Provenance/Observability/Efficiency | ✅/❌/⚠️/➖ | file:line, actual step/job definition, or command output | only if not ✅ |

(Match the table's actual language to the conversation's language — column names above are
illustrative. Keep "Befund"/"Evidence" concrete: a path+line quoting the actual step, a command and
its output, or "not found — searched X, Y, Z.")

Status legend:
- ✅ Pass — the pipeline mechanism/safeguard was found and verified in the actual config
- ❌ Fail — checked, and the mechanism is missing, misconfigured, or actively defeated (e.g.
  `continue-on-error` on a test job that's supposed to gate merges)
- ⚠️ Needs manual/platform review — code can't answer this from the repo alone (branch-protection
  enforcement, environment-approval settings, whether a deploy gate's absence is intentional)
- ➖ N/A — this check's surface doesn't exist here (no CD/deployment automation yet, no container
  images built, single-environment setup) — state why in one clause

End with a **prioritized punch list**: every ❌, ordered by deployment-risk severity (an
unrecoverable/ungated production deploy or a fork-PR secret exposure outranks a missing cache),
each with the one-line fix. Follow it with a **needs-review list**: every ⚠️, since those need a
platform/dashboard check or a human process decision rather than a config change alone.
