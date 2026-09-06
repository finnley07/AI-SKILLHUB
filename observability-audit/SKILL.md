---
name: observability-audit
description: Runs a structured, evidence-based audit of a system's observability — logging, metrics, tracing, alerting, dashboards, SLOs/error budgets, runbooks/on-call readiness, health checks, and log/metric retention & cost — then reports the results as one table (check, area, status, evidence, recommendation). Covers structured logging and log-level discipline, correlation/request/trace IDs threaded through a request's lifecycle, golden-signal (RED/USE) metrics coverage for request-driven paths and background/async jobs, distributed trace-context propagation across service and queue boundaries plus sampling strategy, dashboard existence and content (golden signals vs. raw infra graphs, a clear "is the system healthy" entry point), alert quality (symptom-based vs. cause-based, alert fatigue, ownership/runbook links), SLO/error-budget definition and whether it's measured against real production data, on-call runbook coverage and escalation-path documentation, liveness/readiness health-check depth and whether they're actually wired into deploy/routing decisions, and log/metric retention lifecycle plus high-cardinality metric cost risk. This skill covers general engineering observability only — it explicitly does NOT re-check the security-specific logging items already owned by `cybersecurity-check` (audit logging for sensitive actions, no PII/secrets in logs, documented incident-response process); run that skill separately for those, and see the cross-reference note near the end of this file. Use this whenever the user asks for an "observability audit", "observability check", "observability review", "monitoring review", "monitoring check", "monitoring audit", "logging review", "logging audit", "metrics review", "metrics audit", "tracing review", "distributed tracing check", "alerting review", "alert quality check", "alert fatigue check", "on-call readiness", "on-call review", "SLO check", "SLO review", "error budget check", "runbook review", "runbook coverage", "health check review", "dashboard review", "golden signals check", "RED/USE check", "is our monitoring good enough", "can you check our alerting", "review our dashboards", "Observability-Check", "Observability-Audit", "Monitoring-Überprüfung", "Monitoring-Check", "Logging-Überprüfung", "Alarmierung prüfen", "Alerting-Check", "Bereitschaftsdienst-Check", "Bereitschaftsdienst-Bereitschaft", "SLO-Überprüfung", "Fehlerbudget-Check", "Runbook-Überprüfung", "Health-Check-Überprüfung" — even if the user only names one or two specific items (e.g. just "can you check our alerting" or "review our dashboards") rather than the skill's full name.
---

# Observability Audit

A structured, evidence-based check of whether a system's logging, metrics, tracing, alerting, and
on-call tooling actually let engineers tell what the system is doing right now and diagnose it
quickly when something breaks. This is a static/config investigation, not a live chaos-engineering
exercise, and it reports one table the user can act on.

## Ground rules

- **Evidence or it didn't happen.** Every row needs a concrete pointer: a `file:line` for
  instrumentation code, the actual contents of a dashboard/alert-rule config file, a command's
  actual output (e.g. `grep` results, a metrics-endpoint scrape, a query against the logging
  backend), or an explicit note that this needs a human to confirm (e.g. "does the on-call
  rotation doc match who's actually paged" isn't verifiable from a repo alone). Never mark
  something ✅ because "a framework like this usually logs enough" or "they probably have
  dashboards somewhere" — either you found it, or you didn't.
- **General engineering observability only — not the security-logging checks.** This skill answers
  "can we tell what the system is doing, and can we detect/diagnose a problem quickly." It does
  **not** duplicate `cybersecurity-check`'s `references/security.md` checks S26–S29 (audit logging
  for sensitive actions, no PII/secrets in logs, monitoring for security anomalies, documented
  incident-response process). If both skills are run on the same project, this skill's rows and
  those S26–S29 rows are deliberately disjoint — don't re-derive S27 here just because you're
  already reading log statements; point at `cybersecurity-check` instead (see the note near the
  end of this file).
- **Don't invent scope you can't check, and don't silently drop scope either.** No metrics backend
  at all in this project? No tracing library anywhere? No on-call tool integrated? Say so — mark
  the row `➖ N/A` with a one-line reason (what you searched for and didn't find). Every check below
  gets a row in the output table; none are quietly skipped because "this project clearly doesn't do
  that."
- **Inspect actual instrumentation and config — never assume a framework's defaults are enough.**
  A framework logging request lines out of the box doesn't mean application code emits structured,
  correlated logs. A tracing library being a dependency in `package.json`/`.csproj`/`requirements.txt`
  doesn't mean it's wired through every async boundary — check the actual context-propagation code.
- **Be genuinely thorough.** Don't stop at the first few checks because the table is getting long —
  the checks that are tedious to verify (tracing across a queue boundary, whether an alert has ever
  actually fired and been acted on) are usually exactly the ones worth getting right.

## Workflow

1. **Map the observability surface.** Identify: the logging library/format in use (structured
   logger like `pino`/`serilog`/`structlog`/`zap` vs. plain `console.log`/`print`/string
   concatenation), the metrics backend if any (Prometheus, Datadog, CloudWatch, New Relic, Grafana
   Cloud, StatsD, ...), the tracing setup if any (OpenTelemetry, Jaeger, Zipkin, X-Ray, a vendor
   APM agent), the alerting tool (Alertmanager, PagerDuty, Opsgenie, Datadog Monitors, CloudWatch
   Alarms), and where dashboards live (Grafana, Datadog, a cloud console, none). Skim
   `README.md`, `docker-compose.yml`, infra-as-code (Terraform/Helm/CDK), and CI/deploy config
   first — most of this surface is discoverable there before diving into application code.
2. **Work through every check below**, grouped by theme; each gets a row in the final table marked
   ✅/❌/⚠️/➖.
3. **Investigate the actual code and config, don't infer from tooling presence.** A dependency being
   installed is not evidence it's used correctly — find the instrumentation call sites, the actual
   dashboard JSON/provisioning files, the actual alert-rule definitions, and read them.
4. **Report as one table** in the format below, most severe/impactful failures first within each
   area, followed by a prioritized punch list and a needs-review list.

## Structured logging

**OB1 — Structured log format, not free-text concatenation.** Logs should be emitted as JSON or
equivalent key-value structure (fields separate from message), not
`log.Info("User " + userId + " did " + action)` style strings that can't be reliably parsed/queried
at scale in a log backend. Grep for the logging calls actually used
(`logger\.(info|warn|error|debug)`, `console\.log`, `print(`, `System.out.println`) and inspect a
representative sample: do they pass a structured payload/fields object, or an already-interpolated
string? A logging library capable of structured output doesn't count if call sites still build
strings by hand.

**OB2 — Log-level usage is consistent and meaningful.** Check that `debug`/`info`/`warn`/`error`
(or the stack's equivalent) are used with a consistent meaning across the codebase, not everything
logged at `info` (making the level filter useless) and not `error` used for expected/handled
conditions (a validation failure returning 400 to the client is not the same severity as an
unhandled exception — logging both at `error` drowns real failures in noise). Spot-check: grep for
`error(` / `logger.Error` call sites and read a sample — are they genuinely unexpected failures, or
routine "user typed the wrong password" cases that shouldn't page anyone?

**OB3 — Correlation/request/trace IDs on every log line within a request's lifecycle.** A single
incoming request or job execution should carry an identifier (request ID, correlation ID, or trace
ID) that appears on every log line it produces, so the full sequence of events for one request can
be reconstructed from the log backend by filtering on that ID. Check: is the ID generated/extracted
at the entry point (middleware, request handler, job runner)? Is it actually attached to the logger
context for the duration of the request (e.g. via `AsyncLocalStorage`/`contextvars`/thread-local/a
logger child-instance pattern), or only logged once at the start and lost afterward — a common gap
where the ID exists but doesn't propagate into downstream log calls, async callbacks, or calls made
inside a spawned job.

## Metrics coverage

**OB4 — Golden-signal / RED metrics for request-driven paths.** For each externally-facing service
or API, confirm actual instrumentation (not just "the framework could expose this") for rate
(requests/sec), errors (error rate/count by status or type), and duration (latency, ideally as a
histogram/percentiles, not just an average). Grep for the metrics client's instrumentation calls
(`prom-client`, `micrometer`, `statsd`, an OpenTelemetry Meter, a Datadog `dogstatsd` client) at the
actual request-handling layer (middleware is the common, correct place) — a metrics library
installed but only used for one hand-picked custom counter elsewhere doesn't cover this.

**OB5 — Golden-signal / USE metrics for critical background jobs and resource-driven components.**
For queues, workers, cron jobs, and resource-bound components (DB connection pool, cache, message
broker), check for utilization, saturation, and error-rate instrumentation — job success/failure
counts, job duration, queue consumer lag, connection-pool usage. A system with API metrics but zero
visibility into whether its background worker is falling behind or silently failing is a real gap,
not an acceptable omission because "it's not user-facing."

**OB6 — Business-relevant metrics beyond pure infrastructure.** Infrastructure metrics (CPU,
memory, disk, network — usually free from the cloud provider/orchestrator) are necessary but not
sufficient; check whether the metrics setup also captures things a CPU graph can't show: queue
depth, job processing lag, cache hit/miss ratio, per-endpoint business throughput (e.g. orders
placed, payments processed). If the only metrics dashboarded anywhere are infra-level, note this
explicitly as a gap rather than letting infra metrics stand in for "we have metrics."

## Distributed tracing

**OB7 — Trace-context propagation across service and queue boundaries.** If the system spans
multiple services/processes (microservices, a queue-based worker, a serverless function chain),
check that trace context is actually injected into outbound calls and extracted on the receiving
side — not just that a tracing SDK is initialized. Concretely: for HTTP calls between services,
grep the outbound HTTP client setup for propagation headers (`traceparent`/`tracestate` for W3C
Trace Context, or a vendor equivalent) being attached automatically (auto-instrumentation) or
manually; for queue/message-based hops (SQS, RabbitMQ, Kafka, a job scheduler), check whether the
producer writes trace context into message attributes/headers and the consumer reads it back into
its own span — this hop is the one auto-instrumentation most often misses silently, producing
traces that look complete in the UI for HTTP-only paths but quietly break/orphan at every queue
boundary. A tracing library being a dependency proves nothing here; find the actual
injection/extraction call sites.

**OB8 — Sampling is configured deliberately.** Check the actual sampler configuration (head-based
percentage sampler, tail-based/error-biased sampler, always-on for low-traffic services). 100%
sampling on a high-traffic service is a cost/storage decision that should be a deliberate choice,
not a leftover default; conversely a sampling rate low enough that rare but important error traces
are routinely dropped (no error-biased/tail sampling to compensate) is a real diagnostic gap. No
tracing at all → `➖ N/A` for this and OB7, with a one-line note on what was searched for.

## Dashboards

**OB9 — Dashboards exist for key services and show golden-signal metrics.** Find actual dashboard
definitions (Grafana JSON/provisioning-as-code, Datadog dashboard config, a cloud console's saved
dashboards if inspectable) for the system's main services. Check their panels are the golden
signals from OB4/OB5 — not exclusively CPU/memory/disk graphs with no request rate, error rate, or
latency panel anywhere. A project with metrics instrumented (OB4 ✅) but no dashboard surfacing them
is a real gap: the data exists but nobody can see it without writing an ad hoc query during an
incident.

**OB10 — A clear top-level "is the system healthy right now" entry point.** Beyond per-service
dashboards, is there one dashboard (or a small number) an on-call engineer would actually open
first — an overview showing the health of all key services/dependencies at a glance — or would
diagnosing "is anything broken right now" require knowing which of N per-service dashboards to
check and in what order? Absence of this is a real finding, not a nitpick, for any system with more
than a couple of services.

## Alerting quality

**OB11 — Alerts are symptom-based, tied to real impact.** Read the actual alert-rule definitions
(Alertmanager rules, Datadog Monitor config, CloudWatch Alarm definitions, PagerDuty/Opsgenie
service configs). Good alerts fire on symptoms that map to user/business impact: elevated error
rate, latency SLO breach, queue backing up past a threshold that matters. Check for alerts that
instead fire purely on internal causes disconnected from impact — e.g. "CPU > 80%" with no
corresponding latency/error condition, when the service might be running perfectly fine at that
CPU level. Cause-based alerts aren't automatically wrong, but they should be diagnostic aids
attached to a runbook, not paging alerts, unless the causal link to impact is well-established for
that specific system.

**OB12 — No evidence of alert fatigue.** Look at alert volume/frequency if the tool exposes history
(Alertmanager's silence/firing history, PagerDuty/Opsgenie incident counts, a Slack alert channel's
message volume if accessible). Signs of fatigue: a very large number of distinct alert rules for a
system this size, alerts that fire multiple times a day and are routinely acknowledged without
action, or a catch-all noisy channel where real signal would be lost. A curated set of alerts that
rarely fire but mean something when they do is the target state — report what you can actually
observe about firing frequency, or mark `⚠️` if the tooling doesn't expose history you can inspect.

**OB13 — Every alert has a clear owner and a runbook link.** Check alert-rule definitions/routing
config for a target (a specific team/service owner, a PagerDuty service, an on-call schedule) and
for a runbook URL/annotation on the alert itself — not just a description of what the metric means.
An alert firing into a generic channel with no assigned owner and no linked next-step is
functionally "an alert nobody is accountable for," which is worth flagging even if the underlying
metric/threshold is well-chosen.

## SLOs & error budgets

**OB14 — SLOs are defined for key user journeys.** Look for an actual SLO definition — even an
informal one in a doc, dashboard annotation, or SLO-management tool (Nobl9, Google SLO
Monitoring, Datadog SLOs) — for the system's important user-facing flows (e.g. "99.9% of checkout
requests complete under 500ms"), with a corresponding error budget derived from it. If "healthy" is
never defined anywhere beyond an individual's gut feeling, that's a `❌`, not something to infer as
implicitly fine.

**OB15 — SLOs are measured against real production data, not aspirational.** If SLOs exist (OB14),
check whether they're actually wired to a live query/dashboard against production metrics — or
whether the number lives only in a planning doc that nobody revisits. An SLO with no corresponding
live measurement is aspirational, not operational; note this distinction explicitly rather than
letting a documented target count as "done."

## Runbooks & on-call readiness

**OB16 — Runbooks exist for common alerts/failure modes.** For the alerts found in OB11–OB13 and
any documented common failure modes (a known-flaky dependency, a recurring capacity issue), check
for an actual linked runbook (a wiki page, a `runbooks/` directory in the repo, a doc linked from
the alert annotation) with concrete diagnosis and mitigation steps — not just a restatement of what
the alert measures. Responding to paging alerts with no runbook and no linked context means every
incident depends on whoever happens to be on call already knowing the system by heart — flag this
directly when found.

**OB17 — Documented on-call rotation and escalation path exists.** Is there an actual rotation
schedule (in PagerDuty/Opsgenie, a calendar, or a doc) and a documented escalation path for what
happens if the primary on-call doesn't respond? No on-call process at all for a production system
→ `❌`; a rotation exists but with no escalation path defined → `⚠️`/partial, note which half is
missing.

## Health checks

**OB18 — Liveness/readiness endpoints reflect real dependency health.** Find the actual health-check
handler code. A handler that unconditionally returns `200 OK` with no logic says nothing about
whether the service can actually do its job — check whether it verifies real dependency health:
database connectivity, critical downstream service/queue reachability, disk space if relevant.
Distinguish liveness (is the process alive — should stay minimal, mostly process-level) from
readiness (can this instance currently serve traffic — should check the dependencies that matter)
if the platform distinguishes the two; conflating them (a readiness check that's just a liveness
check copy-pasted) is a common, worth-flagging gap.

**OB19 — Health checks are actually wired into deployment/routing decisions.** A correct health
check that the deploy platform never queries is inert. Check the actual orchestrator config
(Kubernetes `livenessProbe`/`readinessProbe`, an ECS/App-Runner health-check path, a load
balancer's health-check target) points at the real endpoint from OB18 and that the failure
threshold/interval is sane (a check so lenient that a genuinely broken instance keeps receiving
traffic for many minutes is close to not having one).

## Log/metric retention & cost

**OB20 — Retention is long enough to investigate, with a bounded lifecycle.** Check the actual
retention configuration on the logging backend and metrics backend (index lifecycle policy,
bucket lifecycle rule, Prometheus/Datadog retention setting). Retention too short to investigate an
incident discovered a few days after the fact is a real gap; conversely, no lifecycle
policy/retention limit at all (indefinite accumulation) is a cost risk worth flagging even if
nothing is on fire today.

**OB21 — No accidental high-cardinality metric labels.** Grep metrics-instrumentation call sites
for labels/tags built from unbounded values — a raw user ID, a full request path with path
parameters un-templated, a raw email or IP as a label. Each distinct label-value combination
becomes its own time series; an unbounded label can silently multiply cardinality and blow up
metrics-backend cost/query performance long after the code was written. Check that path-based
labels use the route template (`/users/:id`) rather than the resolved path (`/users/12345`).

## Cross-reference: sensitive data in logs is out of scope here

Whether logs leak PII/secrets (passwords, tokens, full request bodies with personal data) is
`cybersecurity-check`'s check **S27**, not this skill's. This skill's OB1–OB3 look at log
*structure and correlation*, not log *content sensitivity* — don't re-derive S27 here even though
you'll be reading the same log statements; if the user wants that coverage, point them at
`cybersecurity-check` instead of producing a second, possibly-inconsistent verdict on the same
question.

## Output format

Start with 3–4 sentences: what was checked (confirm you worked through every section above),
what tooling/surface exists vs. doesn't (no metrics backend at all, no tracing, etc.), what
couldn't be reached (no access to the live dashboard/alerting tool, only its config-as-code), and
a reminder that this is a static/config investigation — an alert rule reading correctly doesn't
prove it has ever successfully paged a human, and this skill does not re-check the security-logging
items owned by `cybersecurity-check`.

Then ALWAYS use this exact table — one row per check, none omitted:

| # | Check | Bereich/Area | Status | Befund/Evidence | Empfehlung/Recommendation |
|---|---|---|---|---|---|
| 1 | ... | Logging/Metrics/Tracing/Dashboards/Alerting/SLO/Runbooks/Health Checks/Retention | ✅/❌/⚠️/➖ | file:line, config excerpt, or command output | only if not ✅ |

(Match the table's actual language to the conversation's language — the column names above are
illustrative. Keep evidence concrete: a path+line, a config snippet, a command and its actual
output, or "not found — searched X, Y, Z.")

Status legend:
- ✅ Pass — instrumentation/config found and verified
- ❌ Fail — checked, the capability is missing or broken
- ⚠️ Needs manual/human review — code/config can't fully answer this (whether an on-call rotation
  doc matches who's actually paged, whether alert-firing history shown in a UI you can't query
  represents real fatigue, a judgment call on sampling rate adequacy)
- ➖ N/A — no such surface exists in this project (e.g. no tracing library at all, no background
  jobs) — state what was searched for in one clause

End with a prioritized **punch list**: every ❌, ordered by how much it would hurt during a real
incident if left unfixed, each with a one-line fix. Follow it with a **needs-review list**: every
⚠️, since those need a human (often whoever owns on-call process or the metrics-backend bill) to
close out.
