---
name: performance-audit
description: Runs a structured performance and resource-usage audit across a project's backend, frontend, database, and infrastructure/deployment configuration, then reports the results as one table (check, area, status, evidence, recommendation). Covers backend hot-path efficiency (algorithmic complexity, N+1 queries, missing indexes, connection pooling, caching, blocking I/O, missing timeouts, unbounded pagination, serialization overhead, hot-path logging), frontend performance (bundle size/code-splitting, unnecessary re-renders, image optimization, render-blocking resources, duplicate/unparallelized network requests, Core Web Vitals-relevant patterns), and infrastructure/cost (container resource requests & limits, autoscaling/HPA config, database sizing, connection-pool-vs-max-connections mismatches, CDN/caching-layer usage, serverless cold starts, queue backpressure, always-on cost waste). This is a performance/throughput/latency/memory/CPU/cost review, not a security review and not a general code-quality/readability review — use cybersecurity-check or code-review for those instead. Use this whenever the user asks for a "performance check", "performance audit", "performance review", "perf audit", "speed audit", "resource usage check", "resource audit", "ressourcencheck", "performance-check", "leistungscheck", "geschwindigkeitscheck", "skalierbarkeitscheck", or asks "why is this slow", "warum ist das langsam", "wie schnell ist das", "is this fast enough", "will this scale", "skaliert das", "why is this using so much memory/CPU", "warum verbraucht das so viel speicher/CPU", "are we wasting cloud cost", "cloud-kosten check", "is this ready for load/production traffic", or asks about any specific item this covers (N+1 queries, missing database index, connection pooling, cache eviction/TTL, blocking I/O in async code, missing request timeouts, unbounded pagination, bundle size, re-renders, lazy loading, Core Web Vitals, LCP/CLS/INP, container resource limits, autoscaling/HPA, connection pool exhaustion, CDN caching, cold starts, queue backpressure) — even if they only name one or two of these and not "performance audit" explicitly. Also trigger before a launch or a load-bearing traffic event when the user asks "will this hold up under load" or similar readiness questions.
---

# Performance & Resource Usage Audit

A structured, evidence-based check of a project's backend, frontend, database, and
infrastructure/deployment configuration for performance, scalability, and resource-cost issues —
not a benchmark run and not a substitute for an actual profiler, load test, or Lighthouse run. It
investigates the codebase and configuration for the patterns that *cause* slowness, excess
resource use, or wasted cost, and reports one table the user can act on.

Out of scope, by design: application security (SSRF, injection, auth, secrets — use
`cybersecurity-check`) and general code correctness/readability/maintainability (use
`code-review`). A finding belongs here only if its primary consequence is latency, throughput,
memory/CPU usage, or infrastructure cost.

## Ground rules

- **Evidence or it didn't happen.** Every row needs a concrete pointer — a `file:line`, a grep
  match, an actual config value, or a command's real output. Never write "looks fine," "should be
  fast enough," or "probably scales" — either you found the pattern (good or bad) in the
  code/config, or you say plainly that you couldn't check it.
- **Be explicit about what kind of check each row is.** Most of this skill is static investigation:
  reading code and config to spot patterns known to cause performance/resource problems (N+1
  queries, missing indexes, unbounded caches, missing resource limits). That tells you a problem
  is *likely*, not its actual magnitude — code review cannot tell you a page's real LCP in
  milliseconds, a query's real execution time, or a service's real p99 latency under load. Where a
  finding genuinely needs a profiler, an `EXPLAIN ANALYZE` run, a load-testing tool (k6, Locust,
  JMeter), or a Lighthouse/WebPageTest run to confirm magnitude or severity, say so explicitly and
  mark it `⚠️` rather than guessing a number. If you *can* run something read-only and safe
  yourself (e.g. `EXPLAIN` on a query against a non-production database, a bundle-size command,
  `npm run build -- --analyze`), do it and report the actual output instead of leaving it as a
  guess.
- **Don't invent scope you can't check, and don't silently drop scope either.** If the project has
  no frontend, or isn't deployed to any container/serverless platform, say so — mark the row
  `➖ N/A` with a one-line reason. Every check in the applicable reference file(s) gets a row in the
  output; none are quietly skipped.
- **Never run a real load test or benchmark against a live/production system as part of this
  skill.** Read-only investigation of code and config, plus safe, local, non-destructive commands
  (a local build, a static `EXPLAIN`, an `npm run build` bundle report). If a load test is genuinely
  warranted, say so as a recommendation — don't fire one yourself against something you don't
  control.
- **Be genuinely thorough.** Don't stop at the first few checks in each file because the table is
  getting long, and don't skip a whole reference file because the stack "probably doesn't have that
  problem" — verify it, even briefly, rather than assuming.

## Workflow

1. **Map the surface.** Identify: backend language/framework, database(s) and ORM/query layer,
   frontend framework (if any — web, mobile, or none), deployment target (container/Kubernetes,
   serverless/FaaS, VM, PaaS), and whether an APM/profiler/metrics setup already exists (Datadog,
   New Relic, Prometheus/Grafana, `dotnet-trace`, Chrome DevTools performance recordings, existing
   Lighthouse CI). An existing metrics setup changes what you should ask the user for (real numbers)
   versus what you have to infer from code alone. Skim `README.md`/deployment docs first — most
   projects already document the stack and deployment target, which saves a lot of exploration.
2. **Work through the applicable reference file(s)** — every check in each one gets a row in the
   final table, marked ✅/❌/⚠️/➖ as appropriate:
   - `references/backend-performance.md` — algorithmic hot loops, N+1 queries, missing indexes,
     connection pooling, caching correctness, blocking I/O in async code, missing timeouts,
     unbounded pagination, serialization overhead, hot-path logging
   - `references/frontend-performance.md` — bundle size/code-splitting, re-render patterns, image
     optimization, render-blocking resources, network-request efficiency, Core Web Vitals-relevant
     code smells (`➖ N/A` this whole file if there's genuinely no frontend/UI layer)
   - `references/infra-resources.md` — container/pod resource requests & limits, autoscaling
     config, database sizing, connection-pool-vs-max-connections, CDN/caching-layer usage,
     serverless cold starts, queue backpressure, cost-relevant misconfiguration (`➖ N/A` the
     items that don't apply to this deployment target, e.g. HPA checks for a project with no
     Kubernetes deployment)
3. **Investigate, don't assume.** Use Grep/Read/Bash freely: search for the actual query-building
   code, the actual HTTP client instantiation site, the actual resource-limit YAML, rather than
   inferring from framework defaults or a well-organized-looking codebase. An ORM that supports
   eager loading doesn't mean this project uses it everywhere it should; a `Dockerfile` existing
   doesn't mean the k8s manifest that deploys it sets resource limits — check that file too.
4. **Report as one table** in the format below, most severe/impactful failures first within each
   area, followed by a prioritized punch list and a separate needs-review list.

## Output format

Start with 3-4 sentences: what was checked (confirm which reference files applied and which were
marked N/A at the file level, e.g. no frontend), what couldn't be reached (no access to a
staging/production environment to profile, no load-testing tool run, no visibility into real
traffic patterns), and the caveat from Ground rules that this is static investigation of
causative patterns, not a measured benchmark.

Then ALWAYS use this exact table — one row per check across all applicable reference files, none
omitted:

| # | Check | Bereich/Area | Status | Befund/Evidence | Empfehlung/Recommendation |
|---|---|---|---|---|---|
| 1 | ... | Backend/Frontend/DB/Infra | ✅/❌/⚠️/➖ | file:line, command output, or config value | only if not ✅ |

(Match the table's actual language to the conversation's language — the column names above are
illustrative, not a fixed vocabulary. Keep "Befund"/"Evidence" concrete: a path+line, a command and
its actual output, or "not found — searched X, Y, Z". Group rows by area with a subheading or an
"Area" column, whichever reads more clearly for the number of rows involved.)

Status legend:
- ✅ Pass — good pattern found and verified in code/config (e.g. eager loading used, resource
  limits set at a sane multiple of observed usage, cache has an explicit TTL/eviction policy)
- ❌ Fail — checked, and the anti-pattern/missing safeguard is present (e.g. confirmed N+1 query,
  missing index on a filtered column, no resource limits set, unbounded in-memory cache)
- ⚠️ Needs profiler/load-test/measurement to confirm — code review found a plausible risk (a loop
  that looks O(n²), a component that looks like it re-renders often) but confirming actual
  magnitude/impact requires running a profiler, `EXPLAIN ANALYZE`, a load test, or a Lighthouse/
  WebPageTest pass that this skill did not (and should not, per Ground rules) run itself
- ➖ N/A — this check's surface doesn't exist in this project (state why in one clause — e.g. "no
  frontend in this repo", "not deployed to Kubernetes, no HPA to check")

End with a **prioritized punch list**: every ❌, ordered by likely impact on latency/throughput/
resource cost if left unfixed, each with the one-line fix. Follow it with a **needs-review list**:
every ⚠️, naming the specific tool/run (profiler, `EXPLAIN ANALYZE`, k6/Locust load test, Lighthouse)
that would confirm or dismiss it — don't let these get lost at the bottom of a long table.
