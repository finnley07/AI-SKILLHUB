# Backend performance & resource-usage checks

Focused on the request/job path: what makes a single request or background job slow, or what
makes the service use more CPU/memory/DB connections than it needs to under real load. As with
every check in this skill: read the actual call site, don't infer from the framework's or ORM's
defaults — an ORM that *supports* eager loading doesn't mean this codebase uses it on the path you
're looking at.

## Algorithmic complexity & hot loops

**B1 — Nested loops / quadratic-or-worse complexity on request-path data.** Grep for nested
`for`/`foreach`/`.map`/`.filter` loops (or repeated linear scans, e.g. `list.find(...)` inside
another loop) operating on collections whose size scales with user data (all orders, all users, all
rows in a table) rather than a small fixed-size structure. A nested loop over two potentially large
collections (`O(n*m)`) or a lookup-inside-a-loop pattern (`O(n²)` where a `Dictionary`/`Set`/index
would make it `O(n)`) is the single most common "why is this slow at scale but not in dev" cause.
Check: is the inner lookup backed by a hash map/set, or is it a linear scan repeated per outer
iteration? Flag it even if current data volumes make it unnoticeable — note that explicitly as
"not yet a problem at current scale, will be at N records."

**B2 — Unnecessary work inside a loop.** Look for expensive operations (regex compilation, date
parsing, JSON serialization, a new HTTP/DB client construction — see B4) happening once per loop
iteration when they could be hoisted outside the loop and done once. Small individually, but a
common source of surprising per-request cost at scale.

## Database query patterns (N+1 and friends)

**B3 — N+1 query pattern.** The most common backend performance bug: fetching a list of parent
records, then issuing one additional query per parent to fetch related child data inside a
loop/map, instead of one batched/joined query. Look for:
  - A loop over a query result where the loop body calls another query/ORM-navigation-property
    access (e.g. `order.Customer.Name` triggering lazy-load per iteration in EF Core, `user.posts`
    triggering a fetch per user in an ORM without eager loading, `.get_related()`/lazy-attribute
    access in Django/SQLAlchemy inside a `for` loop over a queryset).
  - Confirm whether the ORM call uses eager-loading (`.Include()` in EF Core, `select_related`/
    `prefetch_related` in Django, `.populate()` in Mongoose, `joinedload`/`selectinload` in
    SQLAlchemy, `with:` in Rails/Laravel) or whether it's left to lazy-load per row.
  - If an ORM/query-logging tool is available, the definitive check is to actually run the code
    path (or a test that exercises it) with SQL query logging on and count queries — report the
    actual count if you can get it; otherwise report the pattern found in code as the evidence.

**B4 — Missing database indexes.** For columns used in a `WHERE`, `JOIN ON`, `ORDER BY`, or foreign
key columns queried directly, check whether an index exists:
  - Grep migration files / schema definitions / ORM model attributes for `[Index]`, `@Index`,
    `db_index=True`, `CREATE INDEX`, or the ORM's equivalent, and cross-reference against the
    columns actually filtered/sorted/joined on in query code.
  - If you have access to a non-production database, `EXPLAIN`/`EXPLAIN ANALYZE` the actual query
    is the definitive check — a sequential scan (`Seq Scan` in Postgres, `type: ALL` in MySQL) on a
    table that isn't tiny is the smoking gun. Run it if you can; otherwise mark the row `⚠️` and
    say an `EXPLAIN` run against the real data volume would confirm it.
  - A missing index on a foreign key used for joins is a very common miss — many ORMs create the
    FK constraint but not an index on it automatically.

## Connection pooling & resource reuse

**B5 — Database connections are pooled, not created per-request.** Check the actual client
construction: is the DB connection/client instantiated once at startup (or via the framework's
built-in pooling — connection strings with `Pooling=true`/`Max Pool Size`, a `Pool` object in
`pg`/`mysql2`/`psycopg2.pool`, a singleton `DbContext`/`DataSource`) and reused, or is a brand-new
connection opened inside the request handler (`new SqlConnection(...)` per request, `psycopg2.
connect()` called inside a route function)? Creating a raw connection per request exhausts the
database's max-connections limit under concurrent load and adds connection-setup latency to every
request.

**B6 — HTTP/outbound clients are reused, not created per-call.** Same pattern for outbound HTTP
clients to other services: `new HttpClient()` constructed inside a method body (a well-known .NET
socket-exhaustion trap — should come from `IHttpClientFactory` or be a long-lived singleton),
`requests.Session()` vs. a bare `requests.get()` per call, `axios.create()` reused vs. a fresh
instance per function. A new client per call means no connection reuse (new TCP+TLS handshake every
time) and, in some runtimes, actual socket/handle leakage under load.

## Caching

**B7 — Missing cache on expensive/repeated work.** Look for calls to expensive operations
(external API calls, heavy computation, large aggregation queries) that are re-executed on every
request/call with identical or near-identical inputs, where a cache (in-memory `MemoryCache`,
Redis, `@lru_cache`/`@cached`, HTTP response caching headers) is plausible but absent. Not every
repeated call needs a cache — flag the ones where the same input is genuinely likely to recur
across requests within a reasonable TTL (e.g. a lookup of rarely-changing reference/config data
being re-fetched from the DB on every single request).

**B8 — Cache without eviction/TTL (unbounded growth).** The inverse problem: a cache exists but
has no size bound, no TTL, and no eviction policy — an in-memory `Dictionary`/`Map` used as an
ad-hoc cache that only grows, never expires or evicts entries. This is a slow memory leak, not a
performance win, and eventually causes an OOM under long-running-process conditions. Check:
does the cache have a max size / LRU eviction (`MemoryCache` with `SizeLimit`, a proper LRU library,
Redis with a `maxmemory-policy`), or an explicit TTL on entries? A cache keyed by something
unbounded (a user ID, a request ID, free-text input) with no eviction is a near-certain leak over
time.

## Async/blocking I/O

**B9 — Blocking I/O on a request thread in an async framework.** In an async-first stack (ASP.NET
Core, Node.js, async Python/FastAPI), grep for synchronous-blocking calls used where an async
equivalent exists and should be used on a hot path: `.Result`/`.Wait()` on a `Task` in C# (a classic
deadlock/thread-pool-starvation risk), synchronous file/DB/HTTP calls in Node (`fs.readFileSync`,
a sync DB driver call) inside a request handler, `requests.get()` (sync) called from inside an
`async def` route in Python instead of `httpx.AsyncClient`/`aiohttp`. Under load, blocking calls on
a limited thread/event-loop pool directly reduce request throughput far more than the raw latency
of the blocking call itself would suggest.

**B10 — Missing timeouts on synchronous calls to external services.** Any outbound call to another
service, API, or the database that doesn't specify a timeout can hang the calling thread/request
indefinitely if the downstream is slow or unresponsive, which under load exhausts the caller's
thread pool/connection pool and takes down the caller too (cascading failure). Check the actual
client configuration (`HttpClient.Timeout`, `axios` `timeout` option, `requests` `timeout=`
parameter, DB command/connection timeout) is set explicitly and to a sane value — a client left at
library defaults (sometimes "no timeout at all") on a call that matters is a real finding.

## API/data-shape efficiency

**B11 — Unbounded pagination on list endpoints.** Any endpoint returning a list (`GET /orders`,
`GET /users`) should enforce a page size, either via required pagination parameters or a hard
server-side cap on the default/maximum page size. Check: can a client omit page-size parameters
entirely and get the full unbounded table back? A list endpoint with no cap is both a performance
risk (one query returns and serializes an ever-growing result set as the table grows) and, at
scale, a way for one request to degrade the whole service.

**B12 — Serialization/deserialization overhead on hot paths.** For very high-throughput endpoints,
check whether the response payload includes large nested object graphs, unused fields, or
unnecessarily verbose formats (full entity graphs serialized instead of a lean DTO/projection) —
serializing more data than the client needs costs CPU on every request and increases payload size/
transfer time. Also check for accidental over-fetching from the DB feeding that serialization (a
`SELECT *`/full-entity-load feeding a response DTO that only needs three fields).

## Logging

**B13 — Excessive or synchronous logging on hot paths.** Look for verbose logging (full request/
response body dumps, per-iteration debug logs inside a loop) left enabled on a path that runs on
every request, and check whether the logging sink is synchronous/blocking (writing directly to a
slow disk or a remote endpoint on the calling thread) rather than buffered/async. Both add latency
proportional to request volume; verbose logging additionally adds real I/O and storage cost at
scale. Check the actual configured log level for production, not just that log statements exist in
code with a level check that would filter them out.
