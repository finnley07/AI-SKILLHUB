---
name: test-plan-generator
description: Generates a structured, concrete test plan (test cases) for a specific feature, change, user story, requirements document, API contract, or code diff — the generative counterpart to this repo's audit skills. Given real source material (acceptance criteria, a user story, a requirements doc, an API/schema contract, or the actual code diff/implementation), it produces a document with prioritized (P0/P1/P2) test cases covering happy-path, negative, boundary/edge, and state-transition scenarios, plus assumptions, scope, test data needs, non-functional flags, and requirement-to-test-case traceability. It does NOT produce an audit findings table and does NOT review a project's overall test strategy/infrastructure — that is `test-strategy-audit`'s job; this skill is scoped to one feature/change, not the whole project. Use this whenever the user asks to "generate a test plan", "write test cases", "create test cases for this feature", "write a test plan for X", "test case generator", "QA test plan", "acceptance test plan", "manual test script", "regression test checklist", "write a test strategy for this feature" (feature-scoped — not the project-wide meaning covered by `test-strategy-audit`), "testplan erstellen", "testplan schreiben", "testfälle schreiben", "testfälle erstellen", "testfälle für dieses feature", "teststrategie für dieses feature", or asks a bare question implying the same need even without naming the skill — "what should I test here", "what should I test for this", "was muss ich testen", "wie teste ich das", "what test cases am I missing", "help me test this change/PR/story".
---

# Test Plan Generator

A generative skill: given a specific feature, change, user story, requirements document, API
contract, or code diff, it produces a concrete, prioritized test plan document — not a findings
table. Every other skill in this repo audits something that already exists and reports
✅/❌/⚠️/➖ rows; this skill instead writes new test cases that don't exist yet, so its output is a
document deliverable (defined below), not a status table.

**This skill is not `test-strategy-audit`.** `test-strategy-audit` asks whether a *project's*
overall testing setup (pyramid balance, coverage tooling, CI gating, flakiness) is sound — it never
produces concrete test cases for one feature. This skill is the opposite scope: one feature or
change, concrete test cases, no opinion on the project's test infrastructure as a whole. If the
user wants both, run them separately.

## Ground rules

- **Ground every test case in real source material.** Use the actual acceptance criteria, user
  story, requirements doc, API contract/schema, or code diff/implementation being tested. Never
  invent generic filler cases like "test the happy path" or "test error handling" — every case must
  be concrete to the actual feature: specific inputs, specific expected outputs or state changes.
  If you don't have enough source material to make a case concrete, that's a gap to surface (see
  next rule), not a reason to write a vague one anyway.
- **Don't silently guess at ambiguous or incomplete requirements.** When requirements are
  ambiguous, incomplete, or contradictory, write the test case anchored to a stated assumption —
  and list every such assumption explicitly in the "Assumptions & open questions" section so a
  human can confirm or correct it before testing starts. A test plan that quietly resolved five
  ambiguities in the author's favor is a worse deliverable than one that flags them.
- **Functional vs. non-functional — cover the former in depth, flag the latter, don't duplicate
  sibling skills.** This skill's core deliverable is functional test cases. Performance,
  accessibility, security, and localization/i18n each have a dedicated sibling skill in this repo
  (`performance-audit`, `accessibility-check`, `cybersecurity-check`, and an `i18n-check` skill for
  localization). Don't attempt their depth here. Do still name, in the "Non-functional
  considerations" section, which of these angles are actually relevant to *this* feature and why —
  so nothing gets forgotten — then point to the sibling skill instead of writing shallow
  performance/a11y/security/i18n test cases yourself.
- **Every test case must be independently executable.** Specific preconditions, specific input
  data, specific steps, specific expected result — never a vague statement a tester would have to
  interpret. For example:

  - **Bad:** "Test that login works with invalid credentials." (No specific input, no specific
    expected result — a tester has to invent both.)
  - **Good:** "Precondition: user `alice@example.com` exists with password `Correct123!` and is not
    locked out. Steps: (1) go to `/login`, (2) enter email `alice@example.com`, (3) enter password
    `wrongpass`, (4) submit. Expected result: HTTP 401 / login form re-renders with error text
    'Invalid email or password', password field cleared, failed-attempt counter for this account
    increments by 1, no session cookie is set."

- **Prioritize by actual risk/impact, not a flat list.** Every test case gets a P0/P1/P2. P0 =
  blocks release if broken (core functionality, data loss/corruption, security-adjacent
  functional bugs, silent wrong-answer bugs). P1 = significant but has a workaround or narrow
  blast radius. P2 = minor/cosmetic/rare-path. A test plan that doesn't triage is not useful under
  time pressure — don't leave everything at one priority level "to be safe."
- **Don't skip scope categories — call out inapplicable ones explicitly instead of omitting them.**
  Always consider positive/happy-path, negative, boundary/edge, and state-transition cases as
  distinct categories. If a category genuinely doesn't apply (e.g. no state transitions in a pure
  calculation function), say so briefly in that section rather than silently dropping it — a
  missing section reads as an oversight, not a deliberate scoping decision.

## Workflow

1. **Identify the feature/change under test from what's actually available.** Read the real
   requirements/user story/acceptance criteria if the user provided or pointed to one (a ticket,
   a markdown doc, a PR description); read the actual code diff/implementation if that's what's
   given (`git diff`, a PR's changed files, the source files themselves). Don't ask the user to
   restate something already readable — go find and read it first.
2. **Check what test coverage may already exist for this area.** Search for existing test files
   covering the same module/endpoint/component (by name, by import, by route) so the plan can
   focus on genuine gaps rather than re-specifying cases that are already solidly covered. Note
   in the plan, briefly, what existing coverage was found and where.
3. **Identify which test case categories and non-functional angles are actually relevant.**
   Decide whether state transitions, boundary conditions, concurrency, permissions/roles, etc.
   apply to this specific feature, and which non-functional angles (performance, accessibility,
   security, i18n) are relevant enough to flag.
4. **Draft the test plan document** following the exact deliverable format below.
5. **Surface assumptions and open questions explicitly** rather than silently resolving ambiguity
   — this is a checkpoint, not an afterthought; do it as part of drafting, not only if something
   happens to be left over at the end.

## Deliverable format

Produce a single document with these sections, in this order:

1. **Header** — feature/scope name, one-line description of what's being tested, and links or
   references to the source requirements actually used (ticket ID, file path, PR number, diff
   range).
2. **Assumptions & open questions** — bulleted. Only include real assumptions made because the
   source material was ambiguous, incomplete, or contradictory. If the requirements were fully
   unambiguous, write "None — requirements were unambiguous" explicitly; never fabricate
   assumptions just to fill the section.
3. **In scope / Out of scope** — explicit lists so a reader knows exactly what this plan does and
   doesn't cover (e.g. "out of scope: performance under load, see performance-audit").
4. **Test environment & data needs** — what test data, accounts/roles, environment state, feature
   flags, or seeded records are needed before the cases below can be executed.
5. **Test cases table** — columns exactly:

   | ID | Priority (P0/P1/P2) | Category (Happy path / Negative / Edge case / State transition / etc.) | Preconditions | Steps | Test Data | Expected Result |
   |---|---|---|---|---|---|---|

   Give every case a stable ID (e.g. `TC-01`), concrete preconditions/steps/data/expected result
   per the ground rules above.

6. **Edge cases & negative scenarios (called out)** — a dedicated subsection restating just the
   IDs of every edge-case and negative test case from the table above, each with a one-line
   rationale for why it's a real risk (not a restatement of the step). This exists so these cases
   can't get lost among happy-path rows in a long table.
7. **Non-functional considerations** — a short bullet list naming which sibling-skill audits
   (`performance-audit`, `accessibility-check`, `cybersecurity-check`, `i18n-check`) would be worth
   running against this feature, and *why* each is relevant to this specific feature (e.g. "this
   endpoint accepts user-supplied file uploads — worth a `cybersecurity-check` pass on validation
   and storage"). Omit an angle entirely if it's genuinely not relevant, but say so rather than
   silently dropping it if it's a close call.
8. **Traceability** — a short mapping from each acceptance criterion/requirement back to the test
   case ID(s) that cover it, so a reviewer can spot an uncovered requirement (or an untraceable
   test case) at a glance. A requirement with no test case ID next to it is itself a finding — call
   it out rather than leaving a gap unremarked.
9. **Exit/acceptance criteria** — what "done testing" means for this feature (e.g. "all P0 and P1
   cases pass; no open P0 defects; open P2 defects are triaged and explicitly accepted or
   scheduled").
