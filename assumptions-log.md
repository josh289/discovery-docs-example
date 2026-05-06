# Assumptions Log — Bugzilla Modernization to Banyan CQRS/Event Sourcing

**Audit Date**: 2026-05-06
**Sources**: Discovery, Exploration, Clarification, Architecture (ADR-001–ADR-008), SOW

---

## Master Assumptions Table

| Assumption ID | Statement | Source | Category | Invalidation Condition | Owner |
|---|---|---|---|---|---|
| CT-1 | We assume the customer team can provide 3 full-stack TypeScript/CQRS-capable developers for the entire ~5-month engagement. | `audit-output/sow.md:8` | customer-team | If fewer than 3 developers are allocated, or if developers lack CQRS/event-sourcing experience requiring ramp-up training. | Customer PM |
| CT-2 | We assume the customer will sign off on parallel-run comparison before legacy decommission, within the planned 1-week parallel-run window. | `audit-output/sow.md:269` | customer-team | If the customer requires a longer parallel-run period (e.g., 4+ weeks) or refuses to decommission the legacy instance after sign-off criteria are met. | Customer PM |
| TK-1 | We assume a primary Bugzilla domain expert (name TBD) is available throughout the engagement to answer questions about the 864-file Perl codebase that documents alone cannot resolve. | `audit-output/sow.md:223` (Tribal-knowledge holder departure trigger) | tribal-knowledge | If the primary Bugzilla domain expert becomes unavailable or leaves the organization. | Customer PM |
| TK-2 | We assume the decomposition of `Bugzilla::User` god-object correctly assigns `can_see_bug` → service-bug, `can_enter_product` → service-product, and `wants_bug_mail` → service-notification, as no single document exhaustively lists every method's destination. | `output/phase-3-clarification/clarification.md:33` (R5: User God-Object Decomposition Scope) | tribal-knowledge | If implementation reveals additional methods in `User.pm` (103 KB, ~3,400 lines) that don't map cleanly to the three named destinations. | Evergreen Tech Lead |
| TK-3 | We assume the `status_workflow` table has no `product_id` column, confirming the workflow is global — based on the exploration's reading of the Bugzilla schema rather than direct database inspection. | `output/phase-4-architecture/decisions/ADR-adr-003-global-workflow-with-product-flags.md:12` | tribal-knowledge | If a specific Bugzilla installation has patched the `status_workflow` table to include per-product overrides not in the stock schema. | Customer Bugzilla Admin |
| EI-1 | We assume the deployment target provides Docker Compose orchestration capable of running 7 microservices + PostgreSQL + Redis + RabbitMQ + Elasticsearch + Jaeger simultaneously (~8 GB RAM minimum for development). | `output/phase-4-architecture/decisions/ADR-adr-004-elasticsearch-search-engine.md:89` (Elasticsearch 8.x node adds ~2 GB) | environmental-infra | If the deployment environment cannot allocate sufficient memory for Elasticsearch (minimum ~2 GB for a development cluster) or does not permit Docker Compose. | Customer DevOps Lead |
| EI-2 | We assume the Banyan platform (`@banyanai/platform-base-service`, `@banyanai/platform-event-sourcing`) remains stable at its current API surface (v1.0.509) throughout the engagement. | `audit-output/sow.md:343` (Platform dependency version change trigger) | environmental-infra | If the Banyan platform releases a breaking change that requires migration of `@CommandHandlerDecorator`, `@ReadModel`, `@EventHandlerDecorator`, or `BaseService.start` APIs. | Evergreen Tech Lead |
| EI-3 | We assume S3-compatible object storage (S3 or MinIO) is available for binary attachment storage, replacing Bugzilla's DB BLOB + filesystem approach. | `output/phase-3-clarification/clarification.md:149` (R8: Binary Attachment Storage is Out-of-Band) | environmental-infra | If the deployment environment cannot provide S3-compatible storage and attachments must remain in the database, reverting to BLOB storage in the event-sourced aggregate. | Customer DevOps Lead |
| ES-1 | We assume API keys inherit full user permissions (unscoped) for v1, matching Bugzilla's current behavior — no additional permission restriction layer is needed on API keys. | `output/phase-3-clarification/clarification.md:250` (Q15: API Key Scoping) | environmental-security | If a security audit or compliance requirement mandates scoped API keys before launch, requiring a new permission model on top of `resource:action` contracts. | Customer Security Lead |
| ES-2 | We assume the existing LDAP/RADIUS authentication providers can be integrated as gateway-level adapters without modifying `service-user` internals, and that the customer provides LDAP directory and RADIUS server configuration. | `output/phase-2-exploration/exploration.md:228` (Critical migration note for LDAP/RADIUS) | environmental-security | If the customer's LDAP schema uses non-standard attributes or the RADIUS server is behind a firewall unreachable from the API gateway. | Customer Security Lead |
| EC-1 | We assume comment deletion is not required for legal/compliance purposes; if it becomes required, it will be handled as `SuppressComment` (body replacement) rather than stream removal. | `output/phase-3-clarification/clarification.md:262` (Q17: Comment Deletion Policy) | environmental-compliance | If a GDPR right-to-erasure or legal-hold requirement mandates actual deletion of comment content from the event stream, requiring event-stream rewriting infrastructure. | Customer Legal/Compliance Lead |
| EC-2 | We assume the existing `bugs_activity` audit trail data in the legacy Bugzilla database does not need to be preserved as queryable events — it can be either discarded or stored as a one-time migration snapshot. | `output/phase-4-architecture/decisions/ADR-adr-002-event-sourcing-activity-replacement.md:36` | environmental-compliance | If a compliance auditor requires that all historical activity data remain queryable through the new system's API, not just as a flat archive. | Customer Legal/Compliance Lead |
| ED-1 | We assume a development velocity of 8 story points per developer per month (OTR Connect Feature Point System), totaling 24 pts/month for a 3-person team across all phases. | `audit-output/sow.md:7` | estimate-dependency | If measured velocity falls below 6 pts/dev/month due to unfamiliarity with the Banyan platform, CQRS patterns, or the Perl codebase's complexity. | Evergreen Tech Lead |
| ED-2 | We assume the 324-point scope estimate covers all 7 services with no scope expansion beyond the architecture documents (`output/phase-4-architecture/services/service-*.md`). | `audit-output/sow.md:153` | estimate-dependency | If new commands, queries, events, or read models are discovered during implementation that are not in the 7-service architecture docs. | Evergreen Tech Lead |
| ED-3 | We assume the Phase 2 service-bug estimate (largest single service at ~69 pts) holds, particularly the 15 Layer 2 policies and 14 event subscription handlers. | `audit-output/sow.md:55`–`audit-output/sow.md:97` | estimate-dependency | If the `StatusWorkflowConfig` aggregate requires per-product override complexity beyond ADR-003, or if custom field type validation proves harder than the generic event model (ADR-014). | Evergreen Tech Lead |
| ED-4 | We assume password hash migration (SHA-256 + salt compat) is achievable within the 4 pts estimated, with no undiscovered hash variants in the legacy database. | `audit-output/sow.md:134` | estimate-dependency | If legacy Bugzilla instances contain password hashes from older algorithms (e.g., DES-crypt or MD5) not accounted for in the SHA-256 compat verifier. | Evergreen Tech Lead |
| ED-5 | We assume the REST compatibility layer (40+ endpoints, snake_case ↔ camelCase mapping, integer ↔ UUID translation) can be built within 8 pts, with no undocumented wire-format dependencies beyond the 10 listed constraints. | `audit-output/sow.md:137` | estimate-dependency | If compatibility testing reveals additional wire-format dependencies (per `audit-output/integration-surface.md` — each new constraint adds ~2 pts). | Evergreen Tech Lead |
| ED-6 | We assume the Elasticsearch boolean-chart query builder (25+ operators) can be built within 8 pts, with each operator mapping cleanly to Elasticsearch DSL constructs. | `audit-output/sow.md:101` | estimate-dependency | If any of the 25+ operators (e.g., change-history tracking, pronoun substitution) prove significantly more complex to express in Elasticsearch than estimated. | Evergreen Tech Lead |

---

## Tribal Knowledge Holders

- **Customer Bugzilla Admin (name TBD)** — possesses deep knowledge of the Bugzilla Perl codebase's runtime behavior, including which of the 60+ extension hooks are actually used in production, which custom fields are active, and which workflow transitions are configured. Rows **TK-3** and **TK-1** depend on this role. If this person departs, the SOW triggers a 15% buffer addition per `audit-output/sow.md:223`.

- **Evergreen Tech Lead** — holds the architectural context from discovery through architecture phases, including the rationale for the 7-service decomposition (ADR-001), the three-way group permission split (ADR-006), and the resolution of 22 clarification items. Rows **TK-2** and **ED-2** depend on this role. If the tech lead rotates off the project, the clarification resolutions must be re-validated against the source code.

- **Customer Infrastructure/Operations Engineer (name TBD)** — knows the production Bugzilla deployment topology: database size, table counts, active extensions, cron jobs (whine scheduler, fulltext indexer), email relay configuration, and LDAP/RADIUS integration details. Rows **EI-1**, **EI-3**, **ES-2**, and **ED-4** depend on this role.

---

## Environmental Assumptions

### Infrastructure

We assume the deployment environment provides Docker Compose orchestration with sufficient resources to run 7 microservice containers, PostgreSQL, Redis, RabbitMQ, an Elasticsearch 8.x node (~2 GB RAM minimum), and Jaeger for distributed tracing (rows EI-1, EI-2). The Banyan platform's `@banyanai/platform-base-service` at version 1.0.509 provides the CQRS/Event Sourcing primitives; a breaking platform API change would require migration effort across all 7 services. S3-compatible object storage (S3 or MinIO) is assumed available for binary attachment migration from Bugzilla's DB BLOB + filesystem approach (row EI-3). If the deployment target is resource-constrained (e.g., a single VM with <8 GB RAM) or does not permit Docker Compose, the Elasticsearch dependency from ADR-004 becomes the first item to re-evaluate.

### Security

We assume API keys remain unscoped for v1, inheriting full user permissions as in Bugzilla today — no additional API-key-level permission restriction is required at launch (row ES-1). LDAP and RADIUS authentication adapters are deployed at the API gateway level, not inside `service-user`, and the customer provides working LDAP directory and RADIUS server configurations (row ES-2). If the customer's LDAP schema uses non-standard attributes (not `LDAPuidattribute` or `LDAPmailattribute` as documented), or if RADIUS is behind a network segment unreachable from the gateway, the authentication integration estimate grows. The Bugzilla SHA-256+salt password hash (`bz_crypt`) is assumed to be the only hash algorithm in the production database; discovery of older algorithm variants (DES-crypt, MD5) would expand the compat verifier scope.

### Compliance

We assume comment deletion is not a legal or compliance requirement; the `CommentAggregate` is append-only with a `SuppressComment` administrative command as the escape hatch if suppression (not deletion) is needed (row EC-1). The historical `bugs_activity` audit trail is replaced entirely by the event stream; legacy activity data does not need to be preserved as queryable events in the new system, only as a migration snapshot (row EC-2). If a GDPR right-to-erasure request or compliance audit requires actual removal of comment content from the event stream, the project would need event-stream rewriting infrastructure not currently in scope. The customer's Legal/Compliance lead should confirm both assumptions before Phase 4 data migration begins.

---

## Conditions That Would Invalidate Estimates

### Invalidates Phase 1 Estimates (82 pts)

- **ED-1's condition**: Measured developer velocity falls below 6 pts/dev/month, extending the Phase 1 timeline from ~5 weeks to ~7+ weeks.
- **EI-2's condition**: Banyan platform breaking change forces API migration in service-user and service-product before Phase 2 can consume their events.

### Invalidates Phase 2 Estimates (126 pts)

- **ED-3's condition**: `StatusWorkflowConfig` requires per-product override logic beyond ADR-003's global-workflow-with-flags model, growing service-bug by ~5 pts.
- **TK-2's condition**: `User.pm` decomposition reveals additional methods (beyond `can_see_bug`, `can_enter_product`, `wants_bug_mail`) that require cross-service logic placement decisions.
- **TK-3's condition**: The `status_workflow` table is discovered to have a `product_id` column in the customer's installation, invalidating the global-workflow assumption and requiring per-product workflow resolution logic.
- **ED-4's condition**: Legacy database contains pre-SHA-256 password hashes, expanding the compat verifier beyond the 4-pt estimate.

### Invalidates Phase 3 Estimates (68 pts)

- **ED-6's condition**: Elasticsearch query translation for the 25+ operators proves more complex than estimated, growing service-search by ~4 pts per operator category.
- **EI-1's condition**: Deployment environment cannot run Elasticsearch, forcing a fallback to PostgreSQL fulltext that requires porting the entire SQL-generation engine (Search.pm, 111 KB) to TypeScript.
- **ES-1's condition**: Security audit mandates scoped API keys before launch, adding a new permission model on top of the existing `resource:action` contracts.

### Invalidates Phase 4 Estimates (48 pts)

- **ED-5's condition**: Compatibility testing discovers undocumented wire-format dependencies beyond the 10 listed constraints, each adding ~2 pts.
- **EC-1's condition**: Legal/compliance requires actual comment deletion (not suppression) from the event stream, requiring event-stream rewriting infrastructure.
- **EC-2's condition**: Compliance auditor requires historical `bugs_activity` data to be queryable through the new system's API, not just archived.

### Invalidates the Entire SOW

- **CT-1's condition**: Customer allocates fewer than 3 developers or developers require significant CQRS/event-sourcing training, reducing effective velocity below the assumed 24 pts/month.
- **TK-1's condition**: The primary Bugzilla domain expert becomes unavailable, adding a 15% buffer to every remaining phase per the SOW's change-order trigger #5.
- **ED-2's condition**: Scope expansion beyond the 7-service architecture — new commands, queries, events, or read models not in `output/phase-4-architecture/services/service-*.md`.

---

## Synthesis gaps

1. **No `audit-output/risk-register.md` (W1)** — The SOW references risk-related re-estimation triggers derived from the clarification gap analysis, but a formal risk register was not available. If produced later, its risk inventory should be cross-referenced against the conditions in this log's "Conditions That Would Invalidate Estimates" section.

2. **Bugzilla domain expert identity unknown** — TK-1, TK-3, and the SOW's tribal-knowledge holder departure trigger all reference a "primary Bugzilla domain expert" whose identity has not been confirmed. This person's availability is a single point of failure for the entire engagement.

3. **Production schema not verified** — The assumption that `status_workflow` has no `product_id` column (TK-3) is based on code reading, not direct inspection of the customer's production database. If the customer has customized the schema, multiple ADRs (ADR-003, ADR-005, ADR-006) may need revision.

4. **Performance SLOs not specified** — The SOW mentions search latency "…<200ms instead of <1s" as a tightening scenario, but no concrete SLOs have been documented. Without baseline performance requirements, Phase 3 estimates for service-search and service-notification are provisional.
