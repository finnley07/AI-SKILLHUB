# Infrastructure, resource sizing & cost checks

Focused on the deployment/runtime layer: is the service provisioned to actually handle its load
without either falling over (under-provisioned) or wasting money (over-provisioned)? Mark
individual checks `➖ N/A` for deployment targets that don't apply (e.g. all Kubernetes/HPA checks
if the project isn't deployed to Kubernetes at all) rather than skipping the whole file — most
projects will have *some* applicable subset (container limits, DB sizing, cost items) even without
Kubernetes specifically.

## Container/pod resource configuration

**I1 — Resource requests & limits are set at all.** For any container/Kubernetes deployment, check
the actual manifest (`deployment.yaml`, Helm `values.yaml`, `docker-compose.yml` resource
constraints) for `resources.requests`/`resources.limits` (CPU and memory) on every container. No
limits set means a single misbehaving pod can starve other workloads on the same node (noisy
neighbor) for CPU, or consume unbounded memory until the node itself is affected; no *requests* set
means the scheduler can't make good placement decisions and the pod has no guaranteed minimum.
Absence of both is a common real-world finding, not a rare one — check for it explicitly rather
than assuming a Helm chart's defaults cover it.

**I2 — Limits are sane relative to actual usage, not guessed.** Where limits exist, check whether
they were set from an observed usage baseline (a comment/doc referencing actual metrics, or values
that look deliberately chosen) versus copy-pasted defaults that are either so low the app gets
OOM-killed under normal load or so high that the cluster is provisioned for far more headroom than
is ever used (direct wasted cost — a pod requesting 4 CPU / 8 GiB that a metrics dashboard would
show using a fraction of that is real, checkable waste if metrics are available). If a metrics/APM
setup exists (from the workflow's surface-mapping step), this is the one check worth asking the
user for actual observed CPU/memory usage numbers to compare against the configured limits, rather
than guessing from the manifest alone.

**I3 — Memory limit vs. OOM-kill risk for the runtime in use.** For runtimes with their own internal
memory management (JVM heap, Node.js `--max-old-space-size`, .NET GC), check whether the
runtime-level memory setting is coordinated with the container memory limit — a JVM heap sized
larger than (or unconstrained relative to) the container's memory limit will get OOM-killed by the
container runtime rather than the JVM handling memory pressure gracefully itself. This is a common,
specific misconfiguration worth checking by name for any JVM/Node/.NET service in a container.

## Autoscaling

**I4 — Horizontal Pod Autoscaler (or equivalent) configured, with sane thresholds.** Check for an
`HPA` resource (or cloud-equivalent: AWS ECS/App Runner autoscaling, Azure App Service autoscale
rules, GCP Cloud Run concurrency/min-max instances) and read its actual target
metric/threshold and min/max replica bounds — not just that autoscaling exists in principle. A
target CPU threshold set unrealistically high (e.g. 90%) means the service is already saturated and
degrading latency before it scales up; a `minReplicas: 1` on a customer-facing service means a
scale-to-zero-then-cold-start gap or a single point of failure during a deploy/restart.

**I5 — Scale-up/scale-down responsiveness (cooldowns/stabilization windows).** Check the
configured scale-up delay/cooldown versus scale-down — an HPA with an overly long scale-up
stabilization window responds too slowly to a real traffic spike; conversely a scale-down window
that's too short causes flapping (scale down, immediately need to scale back up), adding cold-start
latency repeatedly. Report the actual configured values found, not a general statement that
"autoscaling exists."

**I6 — No autoscaling / fixed replica count on a variable-load service.** If the service handles
meaningfully variable traffic (has daily/weekly patterns, marketing-driven spikes, batch-triggered
load) but runs a fixed replica/instance count with no autoscaling at all, flag it — either as an
availability risk (fixed count sized for average load falls over at peak) or a cost issue (fixed
count sized for peak wastes money at all other times).

## Database sizing & connections

**I7 — Database instance size vs. actual/expected load.** Check the provisioned database tier/
instance size (CPU, memory, IOPS class) against any available evidence of actual load (connection
count, query volume, CPU utilization from a managed-DB dashboard or metrics export if accessible).
Without metrics access, this is inherently a `⚠️` — state plainly that confirming right-sizing
needs the actual utilization numbers from the database provider's monitoring, and note only
whether the *tier chosen looks proportionate* to the scale suggested by the rest of the codebase
(e.g. a single-digit-connection hobby-tier instance backing a service configured for a large
connection pool is an obvious mismatch worth flagging even without dashboard access).

**I8 — Application connection-pool limit vs. database's max-connections.** This is a concrete,
checkable mismatch: read the application's configured DB pool size (`Max Pool Size` in a connection
string, SQLAlchemy `pool_size`+`max_overflow`, a `pg.Pool` `max` option, HikariCP `maximumPoolSize`)
and multiply by the number of running application instances/replicas (from I4's replica count),
then compare the total against the database's actual `max_connections` setting (Postgres) or
equivalent. If `replicas × pool_size` can exceed `max_connections`, the database will start
rejecting connections under full scale-out — a real, specific, and commonly-missed capacity bug,
not a theoretical one. Show the actual numbers/arithmetic in the evidence column.

## CDN & static asset delivery

**I9 — CDN/edge caching used for static assets.** Check whether static assets (built JS/CSS
bundles, images, fonts) are served through a CDN or edge cache (CloudFront, Cloudflare, Fastly, a
cloud provider's static-site/CDN offering, or at minimum appropriate `Cache-Control`/`immutable`
headers on hashed-filename assets from the origin) rather than every request hitting the
application origin server directly for unchanging files. Check actual response headers on a static
asset (`curl -I` against a real deployed URL) if one is reachable.

**I10 — Cache-Control correctness for dynamic vs. static responses.** Where caching headers exist,
check they're not misapplied — a `Cache-Control: no-store` on assets that should be cached long-
term (wasted CDN benefit), or the inverse and more dangerous case, aggressive caching applied to a
response that varies per-user/contains sensitive or frequently-changing data (stale/wrong data
served to users, or a data-leak-adjacent caching bug if a shared cache serves one user's response to
another — flag that specific pattern strongly if found, even though it borders on a security
issue, since the root cause is a caching/performance misconfiguration).

## Serverless cold starts

**I11 — Package/deployment bundle size for the function.** For any serverless/FaaS function
(AWS Lambda, Azure Functions, Cloud Functions/Cloud Run), check the deployed package size and
dependency count — a large bundle (an entire unused SDK, dev dependencies accidentally included,
an unnecessarily heavy framework for a single function) directly increases cold-start init time.
Check the actual build/package output size where discoverable, or the `package.json`/
`requirements.txt` for obviously-oversized dependencies for what the function does.

**I12 — Expensive initialization at module load time.** Check what code runs at module/file scope
(outside the handler function) versus inside the handler — a database connection setup, a large
config/schema parse, a heavy SDK client construction done at module load time is *good* (it's
reused across warm invocations) only if it's actually reusable/idempotent; but if a cold start is
frequent (low-traffic function, or a runtime that doesn't keep instances warm), that same
initialization cost is paid on every cold invocation, so its cost should be minimized specifically
(lazy-init non-critical clients, defer anything not needed for every invocation).

**I13 — Provisioned concurrency / warm-up strategy for latency-sensitive functions.** For a
function on a user-facing latency-sensitive path (not a background/batch job), check whether
provisioned concurrency, a scheduled warm-up ping, or a minimum-instance setting (Cloud Run
`min-instances`, Lambda provisioned concurrency) is configured — its absence means every scale-up
event or period of low traffic pays a full cold start on the next request. If the function is
genuinely background/latency-insensitive, mark `➖ N/A` with that reasoning.

## Background jobs & queues

**I14 — Backpressure handling on queue consumers.** Check whether a queue/job consumer has a
concurrency limit (a worker pool size, a `prefetch`/visibility-timeout configuration) matched to
what downstream resources (the database, a rate-limited external API) can actually sustain, versus
consuming as fast as messages arrive with no limit — an unbounded consumer can overwhelm a
downstream dependency exactly when the queue is most backed up (which is usually exactly when
downstream is already stressed).

**I15 — Dead-letter/retry handling doesn't cause resource-wasting retry storms.** Check retry
configuration (max attempts, backoff strategy) on job/message processing — no backoff (immediate
retry in a tight loop) or no max-attempt cap on a permanently-failing message wastes compute
retrying something that will never succeed, and in the worst case fully occupies the consumer pool
with retries of doomed messages while healthy messages queue up behind them.

## Cost-relevant misconfiguration

**I16 — Always-on resources that could scale to zero or be scheduled.** Check for infrastructure
provisioned to run continuously at full size when its actual usage pattern is intermittent or
predictable (a dev/staging environment identical in size to production and running 24/7, a batch-
processing resource sized for its peak but kept running between batch windows, a service with a
platform-supported scale-to-zero option — Cloud Run, some serverless container platforms — left
configured with a nonzero minimum for a low-traffic/non-critical environment). This is squarely a
cost check, not a performance-degradation risk, and worth calling out as such in the recommendation.

**I17 — Over-provisioned "just in case" sizing without evidence.** More generally than I2/I7: look
for any resource sized notably above what the rest of the codebase/config implies is needed (a
worker pool count, a DB tier, a VM size) with no comment, ticket reference, or metrics-based
justification findable — flag these as cost-review candidates even when you can't prove the exact
right size, since the finding here is "unjustified," not necessarily "wrong."
