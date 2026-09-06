# AI-SKILLHUB

A curated, open-source collection of practical [Claude Skills](https://docs.claude.com/en/docs/claude-code/skills) — reusable instruction sets that give Claude structured, repeatable ways to handle real tasks across development, security, data, writing, DevOps, and more.

Each skill lives in its own folder and follows the standard Skill format:

```
skill-name/
  SKILL.md        # frontmatter (name + description) + instructions
  references/     # optional supporting material
```

Claude discovers a skill by its `description` and invokes it automatically when a request matches, or on demand (e.g. `/skill-name`).

## Available skills

| Skill                                                 | Description                                                                                                         |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| [cybersecurity-check](cybersecurity-check/SKILL.md)   | EU-focused security and GDPR/DSGVO compliance check across backend, frontend, and deployment config                 |
| [code-review](code-review/SKILL.md)                   | Structured code review for correctness, error handling, readability/maintainability, API design, and test coverage  |
| [architecture-review](architecture-review/SKILL.md)   | System/service architecture review: module boundaries, coupling, data ownership, resilience, scalability, ADRs      |
| [api-design-review](api-design-review/SKILL.md)       | API contract design & consistency review across REST, GraphQL, and gRPC — naming, errors, pagination, versioning    |
| [database-schema-review](database-schema-review/SKILL.md) | Database schema design and migration-safety review: data modeling, constraints, referential integrity, migrations |
| [ui-consistency-check](ui-consistency-check/SKILL.md) | Visual/UI design-system consistency check: design tokens, component reuse, typography, theming, cross-platform drift |
| [performance-audit](performance-audit/SKILL.md)       | Performance and resource-usage audit across backend, frontend, database, and infrastructure/deployment              |
| [test-strategy-audit](test-strategy-audit/SKILL.md)   | Project-wide test strategy audit: test pyramid, coverage, flakiness, test data management, and CI gating            |
| [test-plan-generator](test-plan-generator/SKILL.md)   | Generates a concrete, prioritized test plan (test cases) for a specific feature, change, or requirements doc        |
| [dependency-audit](dependency-audit/SKILL.md)         | Third-party dependency health check: vulnerabilities, outdated packages, license compliance, supply-chain risk      |
| [ci-cd-pipeline-review](ci-cd-pipeline-review/SKILL.md) | CI/CD pipeline reliability & safety review: build reproducibility, test gating, deployment strategy, rollback     |
| [observability-audit](observability-audit/SKILL.md)  | Logging, metrics, tracing, alerting, SLOs, and on-call/runbook readiness audit                                      |
| [accessibility-check](accessibility-check/SKILL.md)   | WCAG/a11y conformance check for web and mobile, plus European Accessibility Act (EAA)/BFSG applicability            |
| [i18n-check](i18n-check/SKILL.md)                     | Internationalization/localization readiness: translation completeness, pluralization, RTL, locale-aware formatting |

More skills are being added over time — contributions welcome.

## Contributing

Got a skill that's useful in your own workflow? Open a PR. A good skill is:

- **Specific** — solves one clear task well, not a vague catch-all
- **Evidence-based** — produces verifiable output, not guesses
- **Self-contained** — works from its own SKILL.md without hidden dependencies

## License

See [LICENSE](LICENSE).
