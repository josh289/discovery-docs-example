# Statement of Work — Bugzilla Modernization to Banyan CQRS/Event Sourcing

**Document Date**: 2026-05-06
**Engagement Type**: Legacy migration — full re-architecture
**Source**: Bugzilla (Perl monolith, 864 files, 6.3 MB)
**Target**: Banyan CQRS/Event Sourcing microservices (TypeScript, 7 services)
**Estimation Method**: OTR Connect Feature Point System (1–8 pts per feature; `~/.claude/skills/estimate/SKILL.md`). Story-point totals are reported per phase. Calendar-time projections are intentionally omitted — they vary too much by team experience with CQRS/Event-Sourcing patterns. As a rough yardstick, assume **~100 story-points per month** of team velocity for an experienced TypeScript / event-sourcing team.

---

## Phased Plan

The migration is sequenced into **4 phases** ordered by dependency depth: foundational services first, then dependent services, then the query/notification leaves, then migration cutover. Each phase builds on the prior phase's running services and verified contracts.

---

### Phase 1 — Foundation Services (service-user, service-product)

Story points: 94 pts

These two services have zero inbound cross-service dependencies (service-user is a pure producer; service-product depends only on service-user events). They must be running and emitting events before any other service can begin integration testing.

| Feature | Tier | Points | Source |
|---------|------|--------|--------|
| **service-user** | | | |
| UserAggregate — create, update profile, disable, enable, change password | Large | 8 | `output/phase-4-architecture/services/service-user.md` — Commands table: CreateUser, UpdateUserProfile, DisableUser, EnableUser, ChangePassword |
| Email change (token-based confirmation flow) | Large | 6 | `output/phase-4-architecture/services/service-user.md` — ChangeEmail command |
| GroupAggregate — CRUD + regex auto-membership | Large | 8 | `output/phase-4-architecture/services/service-user.md` — GroupAggregate with userRegexp, CreateGroup, UpdateGroup |
| Group membership — add/remove/inheritance DAG flattening | Large | 7 | `output/phase-4-architecture/services/service-user.md` — AddGroupMember, RemoveGroupMember, AddGroupInheritance, RemoveGroupInheritance |
| API key management (create/revoke) | Medium | 3 | `output/phase-4-architecture/services/service-user.md` — CreateAPIKey, RevokeAPIKey |
| AuthenticateUser — password verification with SHA-256 compat + transparent rehash | Large | 6 | `output/phase-3-clarification/clarification.md` Q14; `output/phase-4-architecture/services/service-user.md` — AuthenticateUser command |
| User settings + email preferences | Medium | 4 | `output/phase-4-architecture/services/service-user.md` — UpdateUserSetting, UpdateEmailPreferences |
| Read models (6 total): UserProfile, UserGroupMembership, GroupList, APIKeyList, UserSettings, UserEmailPreferences | Medium | 4 | `output/phase-4-architecture/services/service-user.md` — Read Models section |
| Queries (11 total): GetUser, GetUserByEmail, ListUsers, GetGroup, ListGroups, ListGroupMembers, ListUserGroups, ListAPIKeys, GetUserSettings, GetEmailPreferences, GetGroupByName | Medium | 5 | `output/phase-4-architecture/services/service-user.md` — Queries table (11 entries) |
| **service-product** | | | |
| ProductAggregate — CRUD + allows_unconfirmed + defaultMilestoneId | Large | 8 | `output/phase-4-architecture/services/service-product.md` — ProductAggregate |
| Product Creation Saga (product → version → milestone → optional group) | Large | 7 | `output/phase-4-architecture/services/service-product.md` — Product Creation Saga section |
| ComponentAggregate — CRUD with default assignee/QA/CC | Large | 6 | `output/phase-4-architecture/services/service-product.md` — ComponentAggregate |
| VersionAggregate + MilestoneAggregate — CRUD (ID-based refs per Q8) | Medium | 5 | `output/phase-4-architecture/services/service-product.md` — VersionAggregate, MilestoneAggregate; `output/phase-3-clarification/clarification.md` Q8 |
| Classification CRUD (not event-sourced per Q11) | Small | 2 | `output/phase-4-architecture/services/service-product.md` — Classification Strategy section; `output/phase-3-clarification/clarification.md` Q11 |
| UpdateGroupControls + GroupControlMapReadModel (ADR-006) | Large | 6 | `output/phase-4-architecture/services/service-product.md` — UpdateGroupControls command; `output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md` |
| Read models (8 total): ProductList, ProductDetail, ComponentList, VersionList, MilestoneList, GroupControlMap, Classification, BugCount | Medium | 5 | `output/phase-4-architecture/services/service-product.md` — Read Models section |
| Queries (9 total) + event subscription (UserDisabled → clear default assignee) | Medium | 4 | `output/phase-4-architecture/services/service-product.md` — Queries table; ClearDefaultAssigneeOnUserDisabledHandler |

#### Rollback

Both services are greenfield — no legacy state to preserve. If Phase 1 fails quality gates, the rollback is to discard the deployed services and rebuild from spec. No data loss risk because no production data migration has occurred yet. Checkpoint: all unit tests pass, `tsc --noEmit` clean, RabbitMQ event emission verified with integration test consumer.

---

### Phase 2 — Core Domain (service-bug, service-comment, service-attachment)

Story points: 133 pts

These three services form the core bug-tracking domain. They depend on service-user and service-product events (projected into local read models) and on each other's events for cross-cutting concerns (comment creation triggers fulltext sync; attachment events trigger system comments).

| Feature | Tier | Points | Source |
|---------|------|--------|--------|
| **service-bug** | | | |
| BugAggregate — create, update, transitionStatus, assign, setProduct | Large | 8 | `output/phase-4-architecture/services/service-bug.md` — BugAggregate behaviors; 24 commands |
| Dependency management (add/remove + DependencyLoopPolicy + read-model reverse-link projection) | Large | 7 | `output/phase-4-architecture/services/service-bug.md` lines 19, 102 — AddBugDependency, RemoveBugDependency, BugDependencyAdded for read-model reverse-link projection; `output/phase-3-clarification/clarification.md` Q2/§Decision Log line 409 (recommended ADR for single-command dependencies, not yet adopted in `output/phase-4-architecture/decisions/`) |
| CC list, keywords, aliases, see-also, groups management | Large | 6 | `output/phase-4-architecture/services/service-bug.md` — AddBugCc, RemoveBugCc, AddBugKeyword, AddBugAlias, AddBugSeeAlso, AddBugGroup commands |
| Mark duplicate + duplicate chain resolution | Medium | 4 | `output/phase-4-architecture/services/service-bug.md` — MarkBugDuplicate; BugDuplicateReadModel |
| Custom fields (generic BugCustomFieldChanged event — recommended ADR-014 from clarification, not yet adopted in `output/phase-4-architecture/decisions/`) | Medium | 5 | `output/phase-4-architecture/services/service-bug.md` — BugCustomFieldChanged; `output/phase-3-clarification/clarification.md` Q7 and §Decision Log line 415 (recommended ADR-014, not yet adopted) |
| Time-tracking fields (estimated, remaining, deadline) | Medium | 3 | `output/phase-4-architecture/services/service-bug.md` — BugTimeTrackingUpdated event; `output/phase-3-clarification/clarification.md` G4 |
| StatusWorkflowConfig aggregate — data-driven state machine (ADR-003) | Large | 7 | `output/phase-4-architecture/services/service-bug.md` — StatusWorkflowConfig aggregate; `output/phase-4-architecture/decisions/ADR-adr-003-global-workflow-with-product-flags.md` |
| Layer 2 Policies (15 total): CanEnterProduct, CanEditBug, CanChangeField, ValidStatusTransition, ResolutionRequired, NoOpenBlockers, CanConfirm, DependencyLoop, StrictIsolation, CanSeeBug, MandatoryField, MustHaveMilestone, CanEditTimeTracking, CanMarkDuplicate, CanChangeProduct | Large | 8 | `output/phase-4-architecture/services/service-bug.md` — Authorization Layer 2 Policies table (15 entries) |
| Read models (9 total): BugDetail, BugList, BugActivity, BugDependency, BugCustomField, BugDuplicate, BugUserLastVisit, StatusWorkflow, BugFieldMetadata | Large | 7 | `output/phase-4-architecture/services/service-bug.md` — Read Models section (9 read models) |
| Event subscription handlers (14 total): product events (8) + user events (4) + comment events (1) + local read model projections | Large | 7 | `output/phase-4-architecture/services/service-bug.md` — Events Consumed table (14 source events) |
| Queries (11 total): GetBug, SearchBugs, GetBugHistory, GetBugFields, GetLegalValues, GetValidStatusTransitions, GetCreationStatuses, GetStatuses, GetResolutions, GetBugDependencies, GetBugUserLastVisit | Medium | 5 | `output/phase-4-architecture/services/service-bug.md` — Queries table (11 entries) |
| Optimistic concurrency (G5) | Medium | 3 | `output/phase-3-clarification/clarification.md` G5; `output/phase-4-architecture/services/service-bug.md` — Invariant I11 |
| **service-comment** | | | |
| CommentAggregate — append-only create, setPrivacy, addTag, removeTag, suppress | Large | 7 | `output/phase-4-architecture/services/service-comment.md` — Commands table: CreateComment, SetCommentPrivacy, AddCommentTag, RemoveCommentTag, SuppressComment |
| System comment generation (3 subscription handlers: BugMarkedDuplicate → types 1+2, AttachmentCreated → type 5, AttachmentUpdated → type 6) | Medium | 4 | `output/phase-4-architecture/services/service-comment.md` — System Comment Generation section |
| Read models (4 total): CommentList, CommentDetail, CommentTagWeight, CommentTagActivity | Medium | 3 | `output/phase-4-architecture/services/service-comment.md` — Read Models section |
| Queries (3): GetComment, GetBugComments, SearchCommentTags | Small | 2 | `output/phase-4-architecture/services/service-comment.md` — Queries table |
| Layer 2 Policies (5): CanCommentOnBug, IsInsider, IsCommentTagger, CanSeePrivateComments, IsAdmin | Medium | 4 | `output/phase-4-architecture/services/service-comment.md` — Authorization Layer 2 section |
| **service-attachment** | | | |
| AttachmentAggregate — create (binary → S3/MinIO), update metadata, mark obsolete, delete | Large | 8 | `output/phase-4-architecture/services/service-attachment.md` — AttachmentAggregate; ADR-005 for binary storage |
| Flag state machine — request/grant/deny/clear on both bug and attachment targets | Large | 8 | `output/phase-4-architecture/services/service-attachment.md` — Flag Workflow State Machine section; `output/phase-4-architecture/decisions/ADR-adr-005-flag-ownership.md` |
| FlagTypeAggregate — CRUD with inclusion/exclusion rules + retargeting | Large | 7 | `output/phase-4-architecture/services/service-attachment.md` — FlagTypeAggregate; UpdateFlagTypeInclusions/Exclusions |
| Event subscription handlers (6): BugMoved → retarget, BugDeleted → cleanup, GroupMemberAdded/Removed → project membership, ProductUpdated, ComponentUpdated → project product/component | Large | 6 | `output/phase-4-architecture/services/service-attachment.md` — Events Consumed table; BugMovedFlagRetargetHandler |
| Read models (5): AttachmentList, AttachmentFlag, BugFlagList, FlagType, FlagTypeScope | Medium | 5 | `output/phase-4-architecture/services/service-attachment.md` — Read Models section |
| Queries (4): GetBugAttachments, GetAttachmentData, GetAvailableFlagTypes, QueryFlags | Medium | 3 | `output/phase-4-architecture/services/service-attachment.md` — Queries (implied from architecture) |
| Layer 2 Policies (7): CanEditAttachment, CanSetFlag, FlagTypeApplicability, RequesteeVisibility, MultiplicableFlag, CanEditPrivateAttachment, CanDeleteAttachment | Large | 6 | `output/phase-4-architecture/services/service-attachment.md` — Authorization Layer 2 Policies section |

#### Rollback

At this phase, services are deployed but no legacy system data has been migrated. Rollback = undeploy services, revert Docker Compose. The legacy Bugzilla instance remains fully operational. Checkpoint: all behavioral tests pass (186+ Gherkin scenarios for service-bug alone per `output/phase-6-spec-readiness/readiness-report.md`), cross-service event flow verified with integration test consumer.

---

### Phase 3 — Query & Notification Leaves (service-search, service-notification)

Story points: 74 pts

These are leaf services with no downstream dependents. They subscribe to events from all prior services. They can be built in parallel once Phase 2 services emit events.

| Feature | Tier | Points | Source |
|---------|------|--------|--------|
| **service-search** | | | |
| Elasticsearch index mapping + bulk migration from bugs_fulltext | Large | 7 | `output/phase-4-architecture/services/service-search.md` — Elasticsearch Index mapping section; `output/phase-4-architecture/decisions/ADR-adr-004-elasticsearch-search-engine.md` |
| Boolean chart AST → Elasticsearch DSL query builder (25+ operators) | Large | 8 | `output/phase-4-architecture/services/service-search.md` — Query Builder Design; operator-map.ts with 25+ operators |
| Quicksearch parser (mini-language: #word, @user, field:value, P1-P5) | Medium | 5 | `output/phase-4-architecture/services/service-search.md` — quicksearch-parser.ts |
| Security filter injection (group visibility + accessibility flags → ES filter) | Medium | 4 | `output/phase-4-architecture/services/service-search.md` — security-filter.ts |
| Saved search CRUD + sharing via groups + footer linking | Large | 6 | `output/phase-4-architecture/services/service-search.md` — SavedSearchAggregate; CreateSavedSearch through ToggleFooterLink |
| ScheduledReport (Whine) CRUD + scheduler execution loop | Large | 6 | `output/phase-4-architecture/services/service-search.md` — ScheduledReportAggregate; Scheduler Execution Flow |
| Event subscription handlers (9): BugCreated, BugUpdated, BugStatusTransitioned, BugResolved, CommentCreated, CommentPrivacyChanged, AttachmentCreated, ProductRenamed, ComponentRenamed | Large | 6 | `output/phase-4-architecture/services/service-search.md` — Event Subscription Handlers section (9 handlers) |
| Chart/reporting queries (GetChartData, GetVisibleSeries) + SeriesCatalogReadModel | Medium | 4 | `output/phase-4-architecture/services/service-search.md` — Queries table; SeriesCatalogReadModel |
| **service-notification** | | | |
| NotificationRecipientResolver — role computation, watcher expansion, preference filtering, forced recipients | Large | 7 | `output/phase-4-architecture/services/service-notification.md` — Recipient Computation section (6-step pipeline) |
| Email template rendering (Handlebars, 11 templates, text+HTML, locale support) | Large | 6 | `output/phase-4-architecture/services/service-notification.md` — Email Template & Rendering section |
| Event subscription handlers (14): BugCreated, BugUpdated, BugStatusChanged, BugAssigned, BugResolved, BugMarkedDuplicate, CCAdded/Removed, DependencyAdded/Removed, CommentCreated, AttachmentCreated, FlagSet/Requested, UserPreferencesChanged, UserWatchingChanged | Large | 7 | `output/phase-4-architecture/services/service-notification.md` — Handler Inventory table (14 handlers) |
| SMTP dispatch + delivery logging | Medium | 4 | `output/phase-4-architecture/services/service-notification.md` — SMTP Configuration table; EmailDispatcher |
| ScheduledReport (Whine) in notification context — scheduler queries service-search + renders digest emails | Medium | 4 | `output/phase-4-architecture/services/service-notification.md` — Whine Scheduled Reports section |

#### Rollback

Both services are pure consumers — disabling them causes no data integrity issues. Bugs simply stop sending email notifications and search index goes stale. Rollback = stop Docker containers, re-enable legacy Bugzilla's email + search. Checkpoint: end-to-end notification delivery test (bug update → email received); search relevance tests pass against seeded data.

---

### Phase 4 — Migration Cutover & Compatibility Layer

Story points: 48 pts

This phase covers data migration from the legacy Bugzilla MySQL/PostgreSQL database into the Banyan event store and read models, deployment of a REST compatibility layer for existing client scripts, and the legacy-to-new traffic cutover.

| Feature | Tier | Points | Source |
|---------|------|--------|--------|
| Data migration ETL — 60+ tables → event streams + read models (bugs, comments, attachments, users, products, groups, dependencies, CC, keywords, custom fields, flags) | Large | 8 | `output/phase-3-clarification/clarification.md` G2; `output/phase-4-architecture/data-migration-strategy.md` |
| User password hash migration (SHA-256 + salt compat) | Medium | 4 | `output/phase-3-clarification/clarification.md` Q14; `audit-output/integration-surface.md` — Password Hash Compatibility |
| Binary attachment migration (DB BLOB + filesystem → S3/MinIO) | Large | 6 | `output/phase-4-architecture/services/service-attachment.md` — Binary Storage Strategy / Migration from Bugzilla |
| REST compatibility layer — legacy `/rest/*` shim mapping snake_case + integer IDs → camelCase + UUID | Large | 8 | `audit-output/integration-surface.md` — REST Endpoints table (40+ endpoints); Compatibility Requirements section |
| Field name mapping (snake_case → camelCase) + ID format (integer → UUID) in compat layer | Medium | 5 | `audit-output/integration-surface.md` — Wire-Format Constraints: Field Names, Field Types |
| API gateway routing configuration — map REST/XMLRPC/JSONRPC paths to Banyan command/query contracts | Large | 6 | `audit-output/integration-surface.md` — API Surface Inventory (40+ REST endpoints) |
| LDAP/RADIUS gateway adapters | Medium | 4 | `output/phase-4-architecture/services/service-user.md` — "LDAP/RADIUS/Env authentication becomes an API gateway concern"; `audit-output/integration-surface.md` — LDAP Directory, RADIUS Authentication Server |
| Email notification format preservation (subject, body template, threading headers) | Medium | 4 | `audit-output/integration-surface.md` — SMTP Outbound Email Notifications; Wire-Format Constraints |
| Co-existence testing — parallel-run legacy + new, compare outputs for correctness | Medium | 3 | `audit-output/integration-surface.md` — Preserve vs. Redesign Matrix |

#### Rollback

The legacy Bugzilla instance remains running throughout Phase 4. If cutover fails, DNS/traffic is routed back to the legacy instance. The event store and read models can be wiped and re-seeded. No irreversible state changes until the legacy instance is decommissioned (explicit customer sign-off required). Checkpoint: legacy REST API parity test — all 40+ endpoints return structurally equivalent responses; data integrity audit — migrated bug count matches source.

---

**Grand Total**: 94 + 133 + 74 + 48 = **349 story points**

At the velocity yardstick noted in the header (~100 pts/month, experienced team), this is roughly a **~3.5-month engagement**. Faster or slower teams should rescale linearly. This is the only calendar-time figure in the document; per-phase month-counts have intentionally been removed because they varied too widely by team to be useful as commitments.

---

## Phase Gates

### Gate: Phase 1 to Phase 2

**Entry Criteria**:
- service-user and service-product Docker containers start successfully
- All contract classes compile (`tsc --noEmit` clean)
- RabbitMQ event emission verified for UserCreated, GroupMemberAdded, GroupMemberRemoved, ProductCreated, ComponentCreated, VersionCreated, MilestoneCreated, GroupControlsUpdated

**Exit Criteria**:
- All behavioral tests pass (unit + integration)
- service-bug and service-attachment teams can subscribe to Phase 1 events and project local read models
- Password hash compatibility verified: existing SHA-256+salt hashes authenticate successfully

**Re-estimation Trigger**: If the group permission model (ADR-006) requires synchronous cross-service calls instead of projected read models, Phase 2 service-bug scope grows by ~8 pts for synchronous call infrastructure.

### Gate: Phase 2 to Phase 3

**Entry Criteria**:
- service-bug, service-comment, service-attachment Docker containers start and consume Phase 1 events
- All 186+ Gherkin scenarios for service-bug pass (per `output/phase-6-spec-readiness/readiness-report.md` — 20 feature files)
- Cross-service event flow verified: CreateBug → BugCreated → comment service creates comment #0; AddAttachment → AttachmentCreated → comment service creates system comment type 5

**Exit Criteria**:
- End-to-end workflow test: create bug → add comment → add attachment → request flag → flag granted → resolve bug → verify bug closed
- No uncommitted spec gaps in readiness report (Gate 1 and Gate 3 failures resolved — TypeScript signatures and contract code blocks added)
- Optimistic concurrency conflicts handled correctly (concurrent UpdateBug returns structured conflict error)

**Re-estimation Trigger**: If the StatusWorkflowConfig aggregate requires per-product override complexity beyond what ADR-003 describes, service-bug scope grows by ~5 pts. If custom field type validation proves more complex than the generic event model recommended in `output/phase-3-clarification/clarification.md` line 415 (recommended ADR-014, not yet adopted), add ~4 pts.

### Gate: Phase 3 to Phase 4

**Entry Criteria**:
- Elasticsearch index seeded with test data; boolean chart queries return correct results
- Email notification delivery verified end-to-end (SMTP relay reachable, templates render correctly)
- Security filtering verified: user A (no group membership) cannot see bugs restricted to group X

**Exit Criteria**:
- Search relevance test suite passes (baseline 25+ operators verified against known result sets)
- Notification recipient computation matches Bugzilla BugMail behavior for 5+ representative scenarios
- Whine scheduled reports execute and deliver email on schedule

**Re-estimation Trigger**: If Elasticsearch query translation for any of the 25+ operators proves significantly more complex than estimated, service-search scope grows by ~4 pts per operator category.

### Gate: Phase 4 to Go-Live

**Entry Criteria**:
- Legacy Bugzilla instance running and accessible
- Data migration ETL script tested against a copy of production data
- REST compatibility layer passes parity test against legacy API for all 40+ endpoints

**Exit Criteria**:
- Migrated data integrity verified: bug count, user count, comment count, attachment count match source within 0.01% (allowing for known data quality issues)
- Legacy client scripts (CI pipelines, automation) work against compatibility layer without modification
- Email notifications from new system are structurally identical to legacy BugMail
- Customer sign-off on parallel-run comparison report

**Re-estimation Trigger**: If undocumented wire-format dependencies are discovered during compatibility testing (per `audit-output/integration-surface.md` — Wire-Format Constraints section lists 10 constraints; discovery of additional constraints adds ~2 pts each). If `audit-output/risk-register.md` grows >20% from its 20-item baseline (R-001..R-020), re-estimate all downstream phases.

### Cross-Phase Re-estimation Triggers

The following conditions apply at **any phase boundary** and force a re-estimation conversation:

1. **Risk register growth >20% from baseline of 20** (R-001..R-020 in `audit-output/risk-register.md`): New risks discovered during implementation that expand scope beyond original estimates.
2. **Undocumented integration discovered**: Any wire format, API contract, or data dependency not captured in `audit-output/integration-surface.md`.
3. **Clarification reopened**: Any of the 22 resolved items (R1–R10, Q1–Q12) or 6 gaps (G1–G6) from `output/phase-3-clarification/clarification.md` is challenged by implementation reality.
4. **Tribal-knowledge holder departure**: If the primary Bugzilla domain expert becomes unavailable, add 15% buffer to the current phase.
5. **SLO tightening**: If performance requirements change (e.g., search latency <200ms instead of <1s), re-estimate Phase 3.

---

## Wire-Format Preservation

The following wire formats MUST be preserved during migration to maintain compatibility with existing client integrations. Sourced from `audit-output/integration-surface.md` and verified against the discovery/exploration phases.

| Format / Contract | Preserve Scope | Migration Strategy | Source |
|-------------------|---------------|-------------------|--------|
| REST API paths (`/rest/bug/*`, `/rest/user/*`, `/rest/product/*`, `/rest/group/*`, `/rest/component/*`, `/rest/classification/*`, `/rest/flag_type/*`) | Wire-compatible: same paths, JSON response structure | Compatibility shim at API gateway mapping legacy paths to Banyan commands/queries | `audit-output/integration-surface.md` — REST Endpoints table (40+ endpoints) |
| Field names (snake_case: `bug_id`, `creation_time`, `is_open`) | Preserve in `/rest/` compat layer; new v2 API uses camelCase | Field name mapping in compat layer | `audit-output/integration-surface.md` — Wire-Format Constraints: Field Names |
| ID format (integer bug IDs) | Preserve in compat layer; new API uses UUID strings | ID mapping table (integer → UUID) in compat layer | `audit-output/integration-surface.md` — Wire-Format Constraints: Field Types |
| ISO 8601 date format | Preserve exactly | Direct pass-through | `audit-output/integration-surface.md` — Wire-Format Constraints: Date Format |
| Error envelope (`{ "error": true, "message": "...", "code": 123 }`) | Preserve per transport | Error code mapping in API gateway | `audit-output/integration-surface.md` — Wire-Format Constraints: Error Envelope Shape |
| Pagination (offset + limit) | Preserve in compat layer | Offset/limit → cursor translation in compat layer | `audit-output/integration-surface.md` — Wire-Format Constraints: Pagination |
| `include_fields` / `exclude_fields` response projection | Preserve | Field filtering in query handlers | `audit-output/integration-surface.md` — Wire-Format Constraints: Field Filtering |
| Email notification format (subject, body, threading headers In-Reply-To/References) | Preserve: recipients must not notice migration | Handlebars templates replicating BugMail structure | `audit-output/integration-surface.md` — SMTP Outbound Email Notifications |
| Password hash format (SHA-256 + salt via `bz_crypt`) | Preserve: existing users must log in without reset | Compatible verifier with transparent rehash on next login (Q14) | `audit-output/integration-surface.md` — Password Hash Compatibility |
| LDAP search filter + attribute mapping (`LDAPBaseDN`, `LDAPuidattribute`, `LDAPmailattribute`) | Preserve: enterprise SSO must not break | Gateway-level LDAP adapter | `audit-output/integration-surface.md` — LDAP Directory |
| RADIUS shared secret + NAS IP protocol | Preserve | Gateway-level RADIUS adapter | `audit-output/integration-surface.md` — RADIUS Authentication Server |
| Inbound email parsing (`@command=value` subject syntax) | Preserve: email-in workflow must continue | IMAP polling or webhook service | `audit-output/integration-surface.md` — Inbound Email |
| GET request CSRF protection (reject cookie auth on GET) | Preserve | API gateway enforces same rule | `audit-output/integration-surface.md` — GET Requests Reject Cookie Auth |
| UTF-8 encoding + line-ending normalization (`\r\n` → `\n`) | Preserve | Direct pass-through in handlers | `audit-output/integration-surface.md` — Encoding: UTF-8 |
| Comment ordering (ascending by creation time, then comment ID) | Preserve | `GetBugComments` query handler enforces ordering | `audit-output/integration-surface.md` — Array Ordering |
| Cross-service event contracts (45 events E-01 to E-45) | Internal: MAY be redesigned (all consumers updated in lockstep) | No external versioning needed | `audit-output/integration-surface.md` — Message and Event Contracts table |

---

## Migration Sequencing

The migration follows a **strangler fig** pattern: the new Banyan services are deployed alongside the legacy Bugzilla instance. Traffic is gradually shifted from the legacy system to the new services. The legacy instance remains operational until all traffic is migrated and verified.

### Sequencing Order

| Step | Action | Co-existence State | Duration |
|------|--------|-------------------|----------|
| 1 | Deploy service-user + service-product. Both are greenfield; no legacy interaction. | Legacy Bugzilla runs untouched. New services emit events into empty RabbitMQ. | Phase 1 |
| 2 | Deploy service-bug + service-comment + service-attachment. Seed local read models from service-user/product events. | Legacy Bugzilla continues handling all traffic. New services are "warm" but receive no user requests. | Phase 2 |
| 3 | Deploy service-search + service-notification. Subscribe to all event streams. Elasticsearch seeded from legacy data snapshot. | Legacy Bugzilla still primary. New search + notifications running in shadow mode (events consumed but emails suppressed). | Phase 3 |
| 4a | Data migration ETL: bulk-export legacy MySQL → seed event store + read models + Elasticsearch. Verify data integrity. | Legacy Bugzilla frozen (read-only mode). New system loaded with historical data. | Phase 4, step 1 |
| 4b | Deploy REST compatibility layer at API gateway. Route `/rest/*` traffic to Banyan services. | Legacy Bugzilla still running (fallback). All new API traffic goes to Banyan. | Phase 4, step 2 |
| 4c | Parallel-run period: both systems receive traffic. New system is primary; legacy is shadow/fallback. | New system handles reads + writes. Legacy system available for emergency rollback. | Phase 4, step 3 |
| 5 | Customer sign-off. Decommission legacy Bugzilla. Remove compatibility shim (optional). | Only Banyan services remain. Legacy instance shut down. | Post-Phase 4 |

### Co-existence Guarantees

During the parallel-run period (steps 4b–4c):

1. **Write-through**: All writes go to Banyan services only. Legacy Bugzilla is read-only.
2. **Read consistency**: API gateway routes reads to Banyan. If Banyan returns 5xx, gateway falls back to legacy read-only instance (stale data acceptable during fallback).
3. **Email deduplication**: During shadow mode, email notifications are sent from one system only (Banyan after step 4b; legacy before).
4. **Search index**: Elasticsearch is the sole search provider after step 4a. Legacy `bugs_fulltext` is not updated after data migration.
5. **Data integrity**: A nightly reconciliation job compares Banyan read model counts against legacy database counts. Discrepancy >0.1% triggers an alert.

### Legacy Retirement Order

1. **First**: Legacy email notification pipeline (BugMail + TheSchwartz) — replaced by service-notification
2. **Second**: Legacy search engine (Search.pm + bugs_fulltext) — replaced by service-search + Elasticsearch
3. **Third**: Legacy REST/XMLRPC/JSONRPC API surface — replaced by API gateway + Banyan services
4. **Fourth**: Legacy Perl application (CGI/PSGI + Template Toolkit) — replaced by React SPA (separate frontend track)
5. **Last**: Legacy MySQL/PostgreSQL database — retained until data migration is verified and sign-off received, then archived

---

## Rollback Boundaries

### Rollback: Phase 1

| Aspect | Detail |
|--------|--------|
| **Checkpoint format** | Docker images for service-user and service-product tagged `phase1-stable`. All contract packages published to npm. |
| **Reversible state** | All services are greenfield. No production data exists. Full rollback = `docker compose down`. |
| **Irreversible state** | None. RabbitMQ exchanges/queues can be deleted and recreated. |
| **Mitigation** | Tag all container images. Store contract package versions. |

### Rollback: Phase 2

| Aspect | Detail |
|--------|--------|
| **Checkpoint format** | Docker images tagged `phase2-stable`. Event stream snapshots for service-bug, service-comment, service-attachment. |
| **Reversible state** | All services can be stopped. Read models can be rebuilt from event replay. |
| **Irreversible state** | None during Phase 2 — no production data has been migrated yet. |
| **Mitigation** | Integration test suite covers all cross-service event flows. If >50% of integration tests fail, halt and rebuild from Phase 2 checkpoint. |

### Rollback: Phase 3

| Aspect | Detail |
|--------|--------|
| **Checkpoint format** | Docker images tagged `phase3-stable`. Elasticsearch snapshot (`_snapshot` API). Email template directory committed to git. |
| **Reversible state** | Search index can be rebuilt from event replay. Notification service is stateless (pure consumer). |
| **Irreversible state** | None — no external data has been written. |
| **Mitigation** | Email delivery failures are logged in `EmailDeliveryLogReadModel`. If delivery failure rate >5%, disable notification service and investigate before re-enabling. |

### Rollback: Phase 4

| Aspect | Detail |
|--------|--------|
| **Checkpoint format** | Data migration script tagged. Legacy database snapshot (full `mysqldump`/`pg_dump`). API gateway routing table snapshot. |
| **Reversible state** | Before step 4c (parallel-run): all Banyan services can be torn down and rebuilt from event store replay. |
| **Irreversible state** | After step 4c sign-off: legacy Bugzilla decommissioned. Mitigation: retain archived database dump for 90 days post-decommission. Event store provides full audit trail. |
| **Mitigation** | Parallel-run period allows comparison of legacy vs. new outputs. Customer must sign off before legacy decommission. If reconciliation job shows >0.1% discrepancy, halt cutover and investigate. |
| **Rollback procedure** | (1) Re-route DNS/load balancer to legacy Bugzilla instance. (2) Re-enable legacy Bugzilla write mode. (3) Disable Banyan API gateway routing. (4) Investigate data drift during parallel-run period. |

---

## Change-Order Triggers

The following observable conditions require a formal change-order conversation between the delivery team and the customer. Each trigger includes the measurement method and the expected action.

| # | Trigger | Measurement | Action |
|---|---------|-------------|--------|
| 1 | **Scope expansion** — new commands, queries, events, or read models discovered outside the 7-service architecture | Diff between service architecture docs (`output/phase-4-architecture/services/service-*.md`) and implementation | Change-order with re-estimated phase points. New features go into a backlog; current phase continues unless the expansion blocks a dependency. |
| 2 | **Undocumented integration discovered** — wire format, API contract, or data dependency not in `audit-output/integration-surface.md` | Any integration test failure caused by a contract not listed in the 45-event or 40-endpoint inventory | Add to integration surface document. Estimate points for the new integration. If >4 pts, issue change-order. |
| 3 | **Risk register growth >20% or HIGH-risk surfacing of an undocumented integration** — new risks beyond the 20 already cataloged in `audit-output/risk-register.md` (R-001..R-020), OR an existing HIGH-likelihood/HIGH-impact risk (R-004 event-naming inconsistencies, R-010 FETL no-rollback) surfaces a previously-unknown integration during implementation | Count of new risks added to `audit-output/risk-register.md` vs. baseline of 20; OR a HIGH risk's mitigation triggers a contract change not in `audit-output/integration-surface.md` | Re-estimate affected phases. Add contingency buffer. For HIGH-risk-driven discoveries, also apply Trigger #2 (Undocumented integration). |
| 4 | **Reopened clarification** — any of R1–R10 (resolved items) or Q1–Q12 (HIGH/MEDIUM questions) or G1–G6 (gaps) from `output/phase-3-clarification/clarification.md` is challenged by implementation reality | Architect or lead developer flags that an accepted ADR (ADR-001 through ADR-008) needs revision | Pause affected service. Re-open architecture discussion. Issue change-order with re-estimated impact. |
| 5 | **Tribal-knowledge holder departure** — the primary Bugzilla domain expert becomes unavailable | Key personnel departure notification | Add 15% buffer to current phase. Identify alternative domain knowledge sources (source code comments, existing documentation). Issue change-order if alternative sources are insufficient. |
| 6 | **SLO tightening** — performance requirements change (e.g., search latency <200ms instead of <1s; notification delivery <5s instead of <30s) | Customer or stakeholder revises performance requirements | Re-estimate affected service(s). Search latency changes affect Phase 3 (service-search + Elasticsearch). Notification latency changes affect Phase 3 (service-notification + SMTP). Issue change-order. |
| 7 | **Spec readiness gate failure** — spec gaps identified in `output/phase-6-spec-readiness/readiness-report.md` are not resolved before Phase 2 begins | Gate 1 (aggregate signatures) or Gate 3 (contract code blocks) remain FAIL | Pause implementation. Return to specification phase. Estimated fix: 2–3 days of spec writing per service. Issue change-order if delay exceeds 5 business days. |
| 8 | **Data migration quality issues** — legacy data contains quality problems not anticipated in the ETL design (corrupt attachments, orphan references, encoding issues) | Data migration ETL error rate >0.1% on first production run | Investigate and fix ETL. Add data cleansing steps. Issue change-order if cleansing adds >4 pts. |
| 9 | **Platform dependency version change** — Banyan platform (`@banyanai/platform-base-service`, `@banyanai/platform-event-sourcing`) releases a breaking change | `pnpm install` fails or behavioral tests break after platform upgrade | Pin platform versions. Evaluate upgrade impact. Issue change-order if migration to new platform APIs adds >4 pts per service. |
| 10 | **Customer-requested feature addition** — new features requested after specification freeze (e.g., Voting extension per `output/phase-3-clarification/clarification.md` Q13, API key scoping per Q15) | Customer formal request for previously deferred feature | Estimate points. Add to backlog. Issue change-order. Deferred features are explicitly excluded from current SOW scope. |

---

## Synthesis Gaps & Risk Cross-References

The risk register at `audit-output/risk-register.md` (W1, 20 risks R-001..R-020) is the authoritative input for risk-driven planning in this SOW. The mappings below tie each high-priority risk (HIGH likelihood and/or HIGH impact in the categories most likely to drive change-orders: wire format, data migration, tribal knowledge, downtime) to the SOW section it most directly affects.

### Wire-Format risks (affect §Wire-Format Preservation, Phase 2 / Phase 4)

- **R-004 (H × H) — Event naming inconsistencies across service architecture docs** [`audit-output/risk-register.md` row R-004]: The four mismatches (`BugProductChanged` vs `BugMoved`, `CommentTagAdded` vs `CommentTagged`, `EmailPreferencesUpdated` vs `UserPreferencesChanged`, plus flag event variations) MUST be reconciled before Phase 2 begins. The Phase 1→Phase 2 gate's exit criterion "RabbitMQ event emission verified" implicitly depends on the reconciled names; a typo silently produces dead `@EventHandlerDecorator` subscriptions. Add ~2 pts of cross-service contract-lint scaffolding in Phase 1.
- **R-005 (M × H) — `bug.Events.BugCreated` payload drift across 4 consumers** [`audit-output/risk-register.md` row R-005]: Drives the §Wire-Format Preservation row "Cross-service event contracts (45 events E-01 to E-45) — Internal: MAY be redesigned (all consumers updated in lockstep)". Lockstep update is only safe if event envelope versioning is in place. If versioning is deferred, escalate to a change-order before any payload field is added or removed.
- **R-019 (L × M) — Bug-level flag events cross service-bug / service-attachment boundary** [`audit-output/risk-register.md` row R-019]: Reinforces ADR-005's flag-ownership boundary; service-bug's `BugFlagListReadModel` is read-only. Captured in Phase 2 service-bug subscription handler scope.

### Data-migration risks (affect §Migration Sequencing and Phase 4)

- **R-010 (H × H) — One-time FETL cutover with no rollback** [`audit-output/risk-register.md` row R-010]: This is the single largest operational risk in the SOW. The §Rollback Boundaries entry for Phase 4 already describes the 90-day legacy retention; the risk register's mitigation (30-day frozen read-only fallback, full dry-run on staging, event-count validation) is incorporated. Change-Order Trigger #8 ("Data migration quality issues") is the operational lever for this risk.
- **R-018 (M × H) — Binary attachment migration is non-transactional** [`audit-output/risk-register.md` row R-018]: Captured in Phase 4 as "Binary attachment migration (DB BLOB + filesystem → S3/MinIO)" (6 pts). Manifest + checksum verification is a pre-cutover gate; if checksum mismatch >0% on production-scale dry-run, escalate via Change-Order Trigger #8.
- **R-012 (H × M) — Orphan version/milestone string references** [`audit-output/risk-register.md` row R-012]: Pre-scan and placeholder-record creation are part of the 8-pt "Data migration ETL" line item in Phase 4. Orphan count is reported in the migration verification report (Phase 4 → Go-Live exit criterion).
- **R-011 (M × M) — Large historical event streams degrade rehydration** [`audit-output/risk-register.md` row R-011]: Snapshot+activity hybrid covered by ADR-002 and the Phase 4 ETL line item.

### Tribal-knowledge risks (affect Change-Order Trigger #5 and Phase 2 estimates)

- **R-001 (M × H) — Bug.pm god-object decomposition loses cross-table side effects** [`audit-output/risk-register.md` row R-001]: Concentrates risk in Phase 2 service-bug. The 70 pts allocated to service-bug already assume a side-effect checklist validated at the spec-readiness gate; if the checklist surfaces undocumented behaviours during implementation, escalate via Change-Order Trigger #4 (Reopened clarification).
- **R-009 (H × M) — Zero test coverage for 6 of 7 services** [`audit-output/risk-register.md` row R-009]: Reinforces the rationale for the 186+ Gherkin scenarios in service-bug at the Phase 1→Phase 2 gate. The "treat `unknown` stability as `fragile`" mitigation is implicitly priced into the Large-tier sizing for behavior-rich aggregates (Bug, Attachment, Flag).
- **R-002 (M × H) — Three-way group permission split** [`audit-output/risk-register.md` row R-002]: Drives the Phase 1→Phase 2 gate's re-estimation trigger ("If the group permission model (ADR-006) requires synchronous cross-service calls instead of projected read models, +~8 pts").
- **R-007 (M × M) — Password hash auto-upgrade hidden write side effect** [`audit-output/risk-register.md` row R-007]: Captured in Phase 1 as the 6-pt "AuthenticateUser — password verification with SHA-256 compat + transparent rehash". Cross-referenced in §Wire-Format Preservation row "Password hash format".
- **R-008 (M × M) — `see_also` recursive reverse-linking** [`audit-output/risk-register.md` row R-008]: Captured under Phase 2 service-bug "CC list, keywords, aliases, see-also, groups management" (6 pts).

### Downtime / cutover risks (affect §Migration Sequencing and §Rollback Boundaries)

- **R-010 (H × H)** — see above.
- **R-020 (M × M) — Secrets management gap for `localconfig` replacement** [`audit-output/risk-register.md` row R-020]: If runtime secret rotation requires service restarts, factor into Phase 4 cutover runbook. May trigger Change-Order Trigger #2 (Undocumented integration discovered) if Banyan platform docs reveal a different rotation model than expected.
- **R-014 (M × H) — Memcached global state has no Banyan equivalent** [`audit-output/risk-register.md` row R-014]: Drives the projection-lag SLO assumption baked into the Phase 1→Phase 2 gate exit criterion (read models must use real-time projection, alert on >5 s lag for security-critical models).

### Items considered and excluded

- **Voting extension migration** (clarification.md Q13): explicitly deferred per `audit-output/risk-register.md` §"Methodology Notes". Listed under Change-Order Trigger #10 (Customer-requested feature addition).

---

*End of Statement of Work*
