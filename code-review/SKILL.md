---
name: code-review
description: Runs a structured code review of a diff, pull/merge request, branch, or file set for correctness bugs, error-handling quality, readability/maintainability, API & contract design, test coverage, documentation accuracy, and version-control hygiene — then reports the results as one table (check, area, status, evidence, recommendation). Explicitly does not cover application security (injection, auth, access control, secrets, SSRF — that's the cybersecurity-check skill) or performance/resource efficiency (algorithmic complexity, N+1 queries, memory/CPU usage — that's the performance-audit skill). Use this whenever the user asks for a "code review", "review this PR", "review this diff", "review my code", "review this pull request", "review this merge request", "MR review", "code quality check", "quality audit", "maintainability review", "readability review", "review my changes", "sanity-check this code", "code smell check", "find code smells", "refactor candidates", "test coverage review", "is this well tested", "codeüberprüfung", "code-review", "codequalität prüfen", "qualitätscheck", "pull request prüfen", "merge request prüfen", "diff überprüfen", "wartbarkeit prüfen", "lesbarkeit prüfen", "fehlerbehandlung prüfen", "testabdeckung prüfen", "code smells finden", or "review anfordern" — even if they only name one or two specific concerns (e.g. "check my error handling" or "is this tested enough") rather than asking for a full review.
---

# Code Review

A structured, evidence-based review of a diff, pull/merge request, branch, or file set against
correctness, error-handling, readability/maintainability, API-design, test-coverage,
documentation, and version-control-hygiene best practice. It is a peer to two other skills and
deliberately does not re-cover their ground:

- **cybersecurity-check** owns application/infrastructure security and GDPR — injection, auth,
  access control, secrets, SSRF, headers, rate limiting, and so on. If you spot something that's
  squarely a security issue (e.g. unparameterized SQL), note it exists in one line and point the
  user at `cybersecurity-check`, but don't build it out as a full finding here.
- **performance-audit** owns algorithmic complexity, N+1 queries, memory/CPU/resource efficiency,
  and caching. Same treatment: a one-line pointer, not a full finding.

This skill's lane is: is the code *correct*, is it *maintainable*, does it handle *failure* well,
is its *contract* sane, and is it *tested and documented*.

## Ground rules

- **Evidence or it didn't happen.** Every row needs a concrete pointer — a `file:line`, a quoted
  snippet, a grep match, or (for something code can't answer) an explicit note that it needs human
  judgment. Don't mark something ✅ because it "looks fine" or "follows best practice" — either you
  traced the actual logic/test and it holds up, or it doesn't get a ✅.
- **This is static, read-only investigation, not execution.** You are reading the diff and the
  surrounding code, not running the program or the test suite to confirm behavior at runtime
  (running the *existing* test suite or a linter to gather evidence is fine and encouraged — just
  don't present "I ran it in my head and it should work" as equivalent to a passing test). Say this
  once, up front, so findings aren't over-trusted.
- **Some checks are inherently judgment calls, not pass/fail facts.** "Is this function doing too
  much," "is this abstraction premature," "is this name clear" don't have a single objectively
  correct answer the way "is this catch block empty" does. Ground every judgment call in a
  concrete example anyway (the actual function, the actual duplicated block) so the user can agree
  or disagree with your reasoning instead of just your conclusion — and mark genuinely
  toss-up calls ⚠️ rather than forcing a confident ✅/❌.
- **Stay in your lane, but don't pretend the other lanes don't exist.** If something is clearly a
  security or performance issue, say so briefly and defer to the sibling skill rather than
  building a full row for it here or silently ignoring it. If a finding is genuinely double-natured
  (e.g. a race condition that both corrupts data *and* causes lock contention), review the
  correctness half here and note the performance half exists.
- **Don't invent scope you can't check, and don't silently drop scope either.** No concurrent code
  in the diff → the race-condition check is `➖ N/A`, one line why. No public API touched → the
  backward-compatibility check is `➖ N/A`. Every check gets a row; none are quietly skipped.
- **Diff-scoped reviews: separate new problems from preexisting ones.** When reviewing a diff/PR,
  distinguish issues the diff *introduces* from issues that were already there in code the diff
  merely touches. Report both, but weight newly introduced issues higher in the punch list — that's
  what's actually actionable in this review.
- **Be genuinely thorough.** Don't stop at the first handful of checks because the table is
  getting long, and don't wave through a big diff with a handful of generic comments — read the
  actual changed lines, not just the file names.

## Workflow

1. **Establish the review target.** A diff needs a base to diff against — figure out whether the
   user means uncommitted working-tree changes (`git status` / `git diff`), a diff against a base
   branch (`git diff main...HEAD` or equivalent), a specific PR/MR number (fetch it via `gh pr diff
   <n>` or the platform's CLI if available), a named branch, or an explicit set of files/a whole
   module. If ambiguous, prefer the most recent uncommitted or unmerged changes — that's almost
   always what "review my code" means day-to-day.
2. **Map the surface.** Identify the language(s)/framework(s), the test framework in use, and the
   rough size and shape of the change (how many files, is it one coherent feature, does it touch
   concurrent/async code, does it touch a public API/exported module). Skim any linked
   issue/PR description for the stated intent — you'll need it for the version-control-hygiene
   checks.
3. **Work through the checklist below**, grouped by theme — every check gets a row, marked
   ✅/❌/⚠️/➖. Investigate with actual Grep/Read, not assumptions: open the changed functions, open
   their existing test files, open a sibling function in the same module to compare conventions
   against. A framework or language having a "usual" pattern doesn't mean this code follows it —
   verify.
4. **Cross-reference tests against changed code directly.** For every changed function/branch,
   look for the actual test that exercises it (C7, C22–C25 below) rather than assuming coverage
   because a test file for the module exists.
5. **Report as one table**, most severe/most-confident-❌ findings first within each area, followed
   by the prioritized punch list and needs-review list.

## Correctness & concurrency

**C1 — Boundary & off-by-one conditions.** Check every loop bound, array/string index, slice, and
pagination/offset calculation touched by the diff (`<` vs `<=`, `length` vs `length - 1`, inclusive
vs exclusive date ranges). Don't eyeball it — trace a concrete small case by hand (empty
collection, one element, the last element) for anything that looks index-adjacent.

**C2 — Null/undefined/None handling.** For every new or changed nullable/optional value, find at
least one call site that dereferences it and confirm a null/undefined check actually precedes it —
don't trust the type annotation alone (a TS `?:` or C# nullable-reference-type annotation is a
compile-time hint, not a runtime guarantee if the value crosses a JSON boundary, cast, or `!`
assertion). Check the language's own idiom is actually followed, not just present in a signature.

**C3 — Error propagation correctness.** Grep for empty `catch`/`except`/`rescue` blocks, catches
that log and don't rethrow/return an error, or catches that wrap the original error in a different
type/status without preserving the cause (`throw new Error("failed")` swallowing the original
exception, an HTTP handler that turns every exception into a generic 500 regardless of what it
actually was). A caught error should be handled meaningfully or propagated with context intact —
never silently discarded.

**C4 — Race conditions & shared mutable state.** If the diff touches concurrent code (threads,
async tasks/workers, locks, shared caches, static/singleton state): look for read-modify-write
sequences without a lock or atomic op, check-then-act patterns (`if not exists: create`) vulnerable
to TOCTOU, and mutable state reachable from more than one call path without synchronization. No
concurrent code touched by the diff → `➖ N/A`.

**C5 — Resource leaks.** Every file handle, socket, DB connection, HTTP response body, or lock
acquired in the diff must be released on *all* paths, including exceptions. Grep for
`open(`/`new FileStream`/`.Open()`/unconsumed `fetch` responses and confirm each is wrapped in
`using`/`try-finally`/`with`/`defer`/a context manager — not just closed at the end of the happy
path, which leaks on any exception thrown in between.

**C6 — Async/await usage.** Missing `await` on a call whose result or exception matters (silent
fire-and-forget that swallows a rejection), `async void` in C# (exceptions from it can't be caught
by the caller), blocking calls on an async path (`.Result`/`.Wait()` on a `Task` — a classic
deadlock risk in sync-over-async), and unnecessary `async`/`Task.Run` wrapping of code with no
actual asynchronous work inside it.

**C7 — Edge cases in the diff not covered by tests.** For each branch (if/else, early return, catch
block) added or changed, check whether a test exercises it — specifically the unhappy path: empty
input, zero, negative numbers, max/overflow values, unicode/special characters — not just the
primary success case. Cross-reference against the actual test file in the diff; don't assume
coverage from a large pre-existing suite.

## Error handling

**C8 — Errors handled at the right layer.** A low-level function (repository, HTTP client wrapper)
shouldn't decide user-facing behavior, and a top-level handler shouldn't swallow an error a
specific caller needed to react to individually (e.g. retry only on a timeout, not on a validation
error). Check that the error type/shape carries enough information for the layer that catches it
to make the right call.

**C9 — Silent failures.** Grep for catch blocks that only log at `debug`/`trace` severity or don't
log at all, unhandled `Promise` rejections, and discarded error return values (`_, err :=` in Go
that's never checked, an ignored exit code). Ask concretely: if this fails in production, does
anyone — a log, an alert, the caller, the user — find out?

**C10 — User-facing error message quality.** Distinct from cybersecurity-check's information-
disclosure check (which flags stack-trace/internal leakage as a security misconfiguration) — the
question here is whether the message is *useful*: does it tell the user what happened and what to
do next, or does a specific, actionable failure (e.g. "email already registered") collapse into a
generic "request failed" that the caller can't distinguish from any other failure?

**C11 — Retry/backoff correctness.** For any retry loop around a network/DB call added or changed
in the diff: is there a maximum-attempt cap (an unbounded retry loop is a real bug, not a
theoretical one)? Is backoff actually increasing (exponential/jittered) rather than a fixed delay
hammering a struggling dependency? Is the retried operation idempotent — retrying a non-idempotent
write (a payment charge, an insert without a dedup key) without an idempotency key risks duplicate
side effects. No retry logic in the diff → `➖ N/A`.

## Readability & maintainability

**C12 — Naming clarity.** Flag specific variable/function/class names that don't describe their
purpose: single-letter names outside a tight loop counter, non-domain-standard abbreviations,
booleans without an `is`/`has`/`should` prefix, or a name that no longer matches what the value
holds after this diff's refactor. Name the actual identifier and suggest a concrete replacement —
not a generic "improve naming."

**C13 — Function/class size & single responsibility.** Flag functions/classes that grew
substantially in this diff and now mix unrelated concerns (validation + business logic +
persistence + formatting inline), especially where the diff bolted a new responsibility onto an
already-large function instead of extracting one. Cite actual line/branch counts as evidence, not
a vague "feels long."

**C14 — Real duplication, not premature abstraction.** Flag near-identical logic blocks — not just
similar-looking code — repeated three or more times, or duplicated business-rule logic that will
silently drift if changed in one place and not the other. Per this repo's philosophy: do **not**
flag a single or double occurrence of similar-but-not-identical code as "should be extracted" —
premature abstraction has its own maintenance cost. Only flag duplication that is (a) genuinely
identical or near-identical, and (b) already at three-plus occurrences or about to be (the diff
adds a third call site).

**C15 — Dead code.** Unreachable code after a `return`/`throw`, unused private
functions/variables/imports (check the language's own tooling if available — `tsc
--noUnusedLocals`, compiler warnings, `vulture`/`deadcode`), and feature-flagged branches whose
flag is permanently on or off such that the other branch can never execute.

**C16 — Commented-out code.** Search the diff for blocks of commented-out code left behind rather
than deleted (version control already preserves the history) — flag for removal, and distinguish
this from genuinely explanatory comments that happen to include a short illustrative snippet.

**C17 — Magic numbers/strings.** Numeric or string literals with unexplained business meaning
(`if status == 3`, a bare `30000` timeout, a role check against `"2"`) that should be a named
constant/enum — flag especially hard when the same literal recurs in more than one place, since a
future change now has to remember to update every occurrence.

## API & contract design

**C18 — Consistent parameter/return conventions.** Compare new/changed public functions or
endpoints against their neighbors in the same module: parameter order, sync vs.
`Promise`/`Task`/callback return, error-return vs. throw, pagination shape, casing convention. Cite
the specific sibling function it now diverges from.

**C19 — Backward-compatibility breaks.** For a public API/exported module/library/HTTP endpoint:
does the diff change a parameter's type or meaning, remove/rename a response field, rename a public
method, or change default behavior in a way existing callers depend on — without a version bump or
deprecation path? Grep the repo for other call sites/consumers before flagging, so the finding is
about a real caller, not a hypothetical one.

**C20 — Input validation at boundaries.** Public functions and endpoints should validate their
inputs (type, range, required-ness) at the entry point rather than trusting the caller and failing
confusingly deep inside the call stack. Check validation sits close to the boundary, not scattered
as ad hoc null/range checks discovered by trial and error further in.

**C21 — Sensible defaults.** For a new optional parameter or config value introduced in the diff:
is its default the behavior most callers actually want? (A changed default on an *existing*
parameter is a compatibility break — that's C19, not this check; this one is specifically about
whether a brand-new default was chosen sensibly.)

## Test coverage

**C22 — New/changed paths covered.** For each function touched by the diff, confirm a test in the
diff itself exercises the new/changed behavior — don't infer coverage from "a test file for this
module exists." Open the actual test file changed alongside the source, or note that none was
added.

**C23 — Tests are meaningful.** Flag tests that only assert "did not throw" without checking the
result, snapshot tests whose snapshot was just re-approved without inspection, or assertions that
are trivially always true (checking a mock was called, without checking it was called with the
right arguments/count).

**C24 — Edge cases & error paths tested.** Beyond the happy path (see C7): is there a test for the
validation-rejects case, the not-found/empty case, and the error path, for whichever of these the
changed code actually handles?

**C25 — Test naming & clarity.** Test names should describe the scenario and expected outcome
(`returns_404_when_user_not_found` over `test1`/`testFoo`), and test bodies should read as
arrange-act-assert without unrelated setup noise obscuring what's actually being verified.

## Documentation

**C26 — Doc comments on non-obvious public APIs.** A public function/method whose behavior isn't
inferable from its signature — side effects, thread-safety guarantees, units on a numeric
parameter, an ordering requirement between calls — should have a doc comment explaining it. Flag
the specific absence where behavior is genuinely non-obvious; don't demand blanket doc-comment
coverage on self-explanatory functions.

**C27 — Stale comments.** Check comments adjacent to changed code still describe what the code now
does. A comment left over from before this diff's change that now contradicts the logic beneath it
is worse than no comment — it actively misleads the next reader. Quote the comment and the
conflicting line.

## Version-control hygiene

**C28 — Commit/PR scope & atomicity.** If reviewing a diff/PR: does it do one coherent thing, or
does it bundle unrelated changes (a refactor plus a new feature plus a formatting pass) that would
review and revert more safely as separate changes?

**C29 — Change matches its stated intent.** Compare the diff's actual contents against its
title/description/linked issue. Does everything in the diff serve that stated purpose, or is there
unexplained scope creep worth calling out to the author?

**C30 — Unrelated changes bundled in.** Formatting-only edits to untouched lines, unrelated
dependency bumps, or drive-by renames mixed into a functional change — flag them separately so the
reviewer can tell which lines actually matter to the change under review. No PR/commit
context available (reviewing a raw file set, not a diff) → `➖ N/A` for C28–C30.

## Output format

Start with 2-3 sentences: what was reviewed (the exact diff/PR/branch/paths and how you resolved
that if it was ambiguous), what couldn't be reviewed (no test suite to cross-reference, no access
to the PR's linked issue, etc.), and the ground-rule caveat that this is static/read-only review,
not an execution trace.

Then ALWAYS use this exact table — one row per check, none omitted:

| # | Check | Bereich/Area | Status | Befund/Evidence | Empfehlung/Recommendation |
|---|---|---|---|---|---|
| 1 | ... | Correctness/Error Handling/Readability/API Design/Tests/Docs/VCS | ✅/❌/⚠️/➖ | file:line or quoted snippet | only if not ✅ |

(Match the table's language to the conversation's language — the column names above are
illustrative, not fixed vocabulary. Keep evidence concrete: a path+line and a quoted snippet, or
"not found — checked X, Y, Z.")

Status legend:
- ✅ Pass — checked the actual code/tests and the concern is handled correctly
- ❌ Fail — a concrete bug, gap, or smell was found
- ⚠️ Needs human judgment — a genuine toss-up (is this abstraction premature, is this name
  actually unclear) where you've given your reasoning but a second opinion is warranted, or
  something outside static analysis (whether a linked issue's intent was fully met)
- ➖ N/A — this check's surface doesn't exist in the reviewed change (state why in one clause, e.g.
  "no concurrent code touched")

End with a **prioritized punch list**: every ❌, ordered correctness/data-loss bugs first, then
silent-failure/error-handling gaps, then missing tests, then readability/style — each with the
one-line fix, and noting which are newly introduced by this diff vs. preexisting. Follow it with a
**needs-review list**: every ⚠️, since those need a second set of eyes rather than a mechanical fix.
