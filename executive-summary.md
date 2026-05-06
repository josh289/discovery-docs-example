# Executive Summary

> **Recommendation: Proceed.** Four-phase strangler-fig, 349 story-points (~3.5-month engagement at experienced-team velocity). **Risk: Medium-High** — driven primarily by the zero-test-coverage baseline (R-009) and the one-shot data migration cutover (R-010). Phases 1–3 are fully reversible; only Phase 4 step 4c (parallel-run sign-off) is irreversible. The legacy Bugzilla instance is retained read-only for 90 days post-cutover as a fallback.

## Headline numbers

| Metric | Value |
|---|---|
| Bounded contexts identified | 7 |
| Source size | ~111 kLOC Perl |
| Decision rules catalogued | 472 |
| Active risks | 19 (1 resolved on 2026-05-06) |
| ADRs ratified | 8 |
| Engagement | 349 story-points (~3.5 months at ~100 pts/month) |

*Sources: cluster-inventory.md, decision-rules.md, risk-register.md, adrs/, sow.md.*

## What the audit revealed beyond scoping

- **Authorization decisions race against eventual consistency.** Bugzilla cached group memberships, group controls, and status workflow in memcached with synchronous invalidation, so a permission change took effect on the next request. Evergreen has no shared cache: each service projects its own read models from events, opening a window between the authoritative state and the in-service projection where group-visibility and transition-permission decisions can be wrong. [source: audit-output/risk-register.md#hidden-behaviors]

- **Hidden cross-aggregate side effects are invisible from the data model.** Adding a local-bug URL as a `see_also` reference automatically inserts the reverse row on the target bug — a behaviour the schema does not reveal (rows are unidirectional). The same shape repeats: comment privacy toggles synchronously re-index the parent bug's fulltext, and product mandatory-group flips retroactively cascade across every existing bug in the product as a non-transactional batch. An extraction that treats each command as a single-aggregate operation will silently lose these. [source: audit-output/risk-register.md#hidden-behaviors]

- **The migration baseline is invisible: zero git history, 6/7 services without tests.** `discovery/bugzilla/` was imported as a single bulk snapshot, so every file reports `(unversioned)` author and zero churn. Only `service-search` has a sibling test file (`xt/search.t`); the other six services have no test signal at all. The migration team has no churn signal, no author signal, and no test-coverage signal to triage decomposition risk — the source code itself is the sole knowledge artifact. [source: audit-output/cluster-inventory.md#stability-ratings]

## Recommended path forward

Strangler-fig in four phases: stand up the zero-inbound-dependency foundation services (user, product) first, then the core bug/comment/attachment triad, then the search and notification leaves, then a freeze-cut-transform-load data migration behind a REST compatibility shim that preserves the legacy endpoints. The first three phases land entirely alongside the live monolith with no production data migrated, so each is independently reversible by reverting the gateway routes. Only Phase 4 step 4c (parallel-run sign-off) crosses an irreversible boundary; the legacy Bugzilla instance is held read-only for 90 days post-cutover as the rollback path of last resort.

## Before Phase 1 starts (customer pre-flight)

- Identify domain experts who can confirm the three hidden-coupling behaviours: `see_also` reverse-linking (R-008), mandatory-group cascade (R-006), and comment-privacy fulltext re-index (R-013) — the source code alone cannot certify these are fully captured.
- Provision read-only access to a production-snapshot Bugzilla database for a full migration dry-run, including pre-scan for orphan version/milestone references (R-010, R-012).
- Confirm with compliance that 90-day read-only retention of the legacy Bugzilla instance is acceptable as the cutover rollback path (sow.md rollback-boundaries).
- Stakeholder ratification of the four event-name canonicalisations the audit applied on the design's behalf: `BugProductChanged`, `CommentTagAdded`, `EmailPreferencesUpdated`, and the target-specific `Flag*` events (R-004 — resolved).
- Sign-off that scheduled-reports / Whine ownership now lives in `service-notification` only; the search-side duplicate has been removed (decision-rules.md duplicates-and-conflicts).
- Confirm an acceptable secret-rotation mechanism in the Evergreen platform — Bugzilla's `localconfig` is read once at startup and has no equivalent runtime story (R-020).

## Alternative considered

A big-bang rewrite of the monolith in a single release was rejected: with all seven clusters at `unknown` stability and no test-coverage signal, a single irreversible cutover would compound the data-migration risk (R-010: non-resumable FETL) across every service simultaneously and eliminate the per-phase rollback checkpoints the strangler-fig preserves at every boundary. [source: audit-output/sow.md#rollback-boundaries]
