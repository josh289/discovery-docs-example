# Architecture Decision Records (ADRs)

Architecture Decision Records capture the significant architectural choices made during the Bugzilla-to-Evergreen modernization audit, each documenting the context, the decision, and its consequences in a stable, reviewable form. The eight ADRs collected here were produced during the architecture phase of the source build pipeline (`output/phase-4-architecture/decisions/`) and are reproduced byte-for-byte from that directory; only the filename prefix has been simplified (from `ADR-adr-NNN-…` to `adr-NNN-…`). They follow the standard `Status / Context / Decision / Consequences` format.

## Index

| ADR | Title | Status | One-line summary |
|---|---|---|---|
| [ADR-001](adr-001-service-boundaries.md) | Service Boundary Decomposition | Accepted (2026-05-06) | Decompose Bugzilla into 7 Banyan CQRS/Event Sourcing microservices, each owning one or more aggregate roots, communicating exclusively through domain events. |
| [ADR-002](adr-002-event-sourcing-activity-replacement.md) | Event Sourcing Replaces `bugs_activity` | Accepted | The `bugs_activity` table will not be migrated; domain events in the Banyan event store replace it entirely as the authoritative audit trail. |
| [ADR-003](adr-003-global-workflow-with-product-flags.md) | Global Status Workflow with Product-Level Flags | Accepted | Adopt a single global status workflow consumed by `service-bug`, with per-product customization limited to the `allows_unconfirmed` boolean flag and group-based access control on transitions applied at the policy/authorization layer. |
| [ADR-004](adr-004-elasticsearch-search-engine.md) | Elasticsearch for Bug Search | Accepted | Use Elasticsearch as the search engine for `service-search`, deployed as a cluster node in the Docker stack. |
| [ADR-005](adr-005-flag-ownership.md) | All Flag Logic Resides in service-attachment | Accepted | All flag logic — both bug-level and attachment-level — resides in `service-attachment`, with `target_type` modeled as a field on the flag entity rather than a service boundary. |
| [ADR-006](adr-006-group-permission-three-way-split.md) | Group Permission Model — Three-Way Split | Accepted | Adopt a three-way ownership split for group permissions across `service-user` (membership), `service-product` (group-control configuration), and `service-bug` (per-bug enforcement). |
| [ADR-007](adr-007-event-payload-design.md) | Event Payload Design — Full Diffs vs Query-Back | Accepted | Domain events carry full diff payloads including changed fields, old/new values, and sufficient context for downstream consumers (especially `service-notification`) to render complete notifications without synchronous query-back to source services. |
| [ADR-008](adr-008-user-identity-surrogate-id.md) | Surrogate userId as Aggregate ID | Accepted | Use a surrogate UUID as the `UserAggregate` ID (`userId`); the `login_name`/email is a mutable field on the aggregate with a case-insensitive unique constraint enforced at the command handler level. |

## Known limitation: missing ADRs

Two further ADRs are referenced elsewhere in this audit (notably in `risk-register.md` and `sow.md`) but were **not materialized as standalone files** in the source build pipeline:

- **ADR-010** — bug-flag ownership (further refinement of the boundary split established in ADR-005).
- **ADR-012** — event payload versioning (operational evolution of the design captured in ADR-007).

Readers should treat the inline references to ADR-010 and ADR-012 as design notes rather than authoritative ADR documents. If those decisions need to be formally ratified, they should be authored as standalone ADRs in this directory using the same `Status / Context / Decision / Consequences` template.
