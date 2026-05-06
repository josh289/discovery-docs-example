# ADR-001 — Service Boundary Decomposition

## Status

Accepted (2026-05-06)

## Context

Bugzilla is a mature, open-source web-based bug-tracking system written in Perl, developed by the Mozilla community and used by thousands of projects worldwide. The application is a monolith: 864 files, 6.3 MB of Perl source, dominated by a single god-object — `Bugzilla::Bug` (156 KB, ~5,100 lines) — which serves as both data-access object and command processor for all bug mutations. Supporting modules for users, products, attachments, flags, comments, search, and email notifications are tightly coupled through direct method calls, shared database transactions (via `bz_start_transaction` / `bz_commit_transaction`), and global state (memcached request cache, site-wide parameters).

The application has zero command/query separation, zero domain events, and zero event sourcing. State changes are written directly to MySQL/PostgreSQL tables within manual transactions, and side effects (email notifications, fulltext index updates, extension hooks) fire synchronously within the same transaction.

The migration target is the Banyan CQRS/Event Sourcing platform, which provides first-class primitives for aggregate roots, command handlers, event subscription handlers, read model projections, and a two-layer authorization model (Layer 1 permissions on `@Command` contracts; Layer 2 `@RequirePolicy` on handlers). Cross-service integration is achieved through subscriptions to `<service>.Events.<EventName>` strings.

The fundamental architectural question is: **how do we decompose this monolith into bounded contexts with clear aggregate-root ownership, and how many services should result?**

Six domain areas were identified during discovery (bug core, user accounts, product hierarchy, attachments/flags, comments, search). Exploration refined this to seven by elevating notifications to a first-class service boundary. The clarification phase resolved all decomposition ambiguities and confirmed the seven-service split is viable.

## Decision

We decompose Bugzilla into **7 Banyan CQRS/Event Sourcing microservices**, each owning one or more aggregate roots with clear domain responsibility. Services communicate exclusively through domain events via the RabbitMQ message bus; no synchronous cross-service calls are permitted during command handling.

### Service Inventory

| # | Service | Aggregate Roots | Mission | Primary Discovery Slice |
|---|---------|----------------|---------|------------------------|
| 1 | `service-bug` | `BugAggregate` | Own the Bug aggregate root — the central entity tracking issue lifecycle, fields, dependencies, CC lists, keywords, custom fields, status workflow, and activity history. | `bug-core-domain`, `bug-status-workflow` |
| 2 | `service-user` | `UserAggregate`, `GroupAggregate` | Manage user account lifecycle, group memberships, API keys, authentication, and per-user preferences. | `user-accounts-auth` |
| 3 | `service-product` | `ProductAggregate`, `ComponentAggregate`, `VersionAggregate`, `MilestoneAggregate` | Manage the Classification → Product → Component hierarchy, versions, milestones, and product-level group permission controls. | `product-hierarchy` |
| 4 | `service-comment` | `CommentAggregate` | Manage the append-only comment thread on bugs, including privacy (insider groups), user-applied tags, and system-generated comments for attachment/duplicate events. | `comments` |
| 5 | `service-attachment` | `AttachmentAggregate`, `FlagTypeAggregate` | Manage file attachments on bugs, all flag types (both bug-level and attachment-level), and the request/grant/deny flag workflow for reviews and approvals. | `attachments-flags` |
| 6 | `service-search` | `SavedSearchAggregate` | Provide complex bug search capabilities (boolean charts, quicksearch, fulltext via Elasticsearch), manage saved/shared searches, and maintain chart/reporting data. | `search-saved-queries` |
| 7 | `service-notification` | `ScheduledReportAggregate` (whine) | Subscribe to domain events from all services, compute notification recipients, render per-user templated emails, and manage scheduled report delivery. | `email-notifications` |

### Decomposition Principle

The boundary is drawn along **aggregate-root ownership and domain responsibility**, not along existing Perl module boundaries. Several modules contain logic belonging to multiple services:

- `Bugzilla::User` (103 KB) contains bug-visibility logic (`can_see_bug`, `visible_bugs`) → belongs in `service-bug`, not `service-user`.
- `Bugzilla::User` also contains product-access logic (`can_see_product`, `can_enter_product`) → belongs in `service-product`.
- `Bugzilla::User` also contains email notification preferences (`wants_bug_mail`) → belongs in `service-notification`.
- `Bugzilla::Flag` / `Bugzilla::FlagType` handle both bug-level and attachment-level flags → unified in `service-attachment` to avoid duplicating inclusion/exclusion resolution, grant-group checking, and retargeting logic.
- `Bugzilla::BugMail` is a cross-cutting notification pipeline → extracted as the dedicated `service-notification`.

### Key Boundary Decisions

1. **Notification is a first-class service**, not a library shared by other services. Recipient computation (roles + watchers + group visibility + per-user preferences) is complex enough to warrant its own bounded context. It is a terminal event consumer — it subscribes to events but no service subscribes to its events.

2. **Status workflow is configuration data owned by `service-bug`**, not a separate workflow service. The `status_workflow` directed graph is global (no `product_id` column); per-product customization is limited to the `allows_unconfirmed` boolean and group-based access control, both of which are authorization concerns layered on top of the global workflow, not separate workflow definitions.

3. **All flag logic (bug + attachment) lives in `service-attachment`**. Flag types, inclusion/exclusion rules, grant/request groups, and the `+`/`?`/`-` state machine are a single domain concept regardless of target type. `service-bug` references bug-level flags via a read model projected from `service-attachment` events.

4. **Group permission model is split three ways**: `service-user` owns group membership (who is in what group), `service-product` owns group control configuration (the `group_control_map` per-product permission matrix), and `service-bug` owns enforcement (consults a projected `ProductGroupControlsReadModel` during Layer 2 policy evaluation).

5. **Extension hooks do not produce a service**. The 60+ hook points map to existing Banyan primitives: lifecycle hooks → event subscriptions, mutation hooks → Layer 2 policies, notification hooks → `service-notification` subscriptions, API hooks → new command/query contracts. All five bundled extensions are disabled and are deferred.

6. **Search is primarily a query-side service**. The SQL-generation engine (Search.pm, 111 KB) is fundamentally rewritten as an Elasticsearch query builder. The only write-side aggregate is `SavedSearch`; the core search execution is read-only against an Elasticsearch index populated by event subscriptions.

### Cross-Service Event Flow Pattern

```
service-user  ──UserCreated, GroupMemberAdded, GroupMemberRemoved──→  (all services)
service-product ──ProductCreated, VersionRenamed, MilestoneRenamed──→  service-bug (validation/denorm sync)
service-product ──GroupControlsUpdated──→  service-bug (read model projection)
service-bug  ──BugCreated, BugUpdated, BugStatusTransitioned, CommentAdded──→  service-notification, service-search
service-comment  ──CommentCreated──→  service-bug (fulltext sync), service-notification
service-attachment  ──AttachmentCreated, AttachmentFlagSet──→  service-comment (system comments), service-notification
service-notification  ──(subscribes to all, emits nothing)──→  terminal consumer
```

### Authorization Mapping

Each service applies Banyan's two-layer authorization model:

- **Layer 1**: Permissions on `@Command` contracts (e.g., `bugs:create`, `users:update`, `attachments:view-private`, `flags:set`).
- **Layer 2**: Business policies on handlers (e.g., `CanSeeBugPolicy`, `CanChangeFieldPolicy`, `CanSetFlagPolicy`, `OwnsProfilePolicy`).

Group membership for authorization decisions is resolved via a projected `UserGroupMembershipReadModel` (sourced from `service-user` events), kept locally in each consuming service to avoid synchronous cross-service calls.

## Consequences

### Easier

- **Clear bounded contexts** — each service has a well-defined mission, explicit aggregate roots, and a known set of commands, events, and queries. A developer working on bug lifecycle does not need to understand notification rendering or search indexing.
- **Natural audit trail** — event sourcing replaces the `bugs_activity` table entirely. Every state change is an event; activity history is a read model projection, not a separate logging mechanism.
- **Cross-service subscriptions are first-class** — the Banyan platform's `@EventHandlerDecorator('<service>.Events.<EventName>')` pattern makes asynchronous integration natural. Email notifications, search indexing, and system comment generation are all subscription handlers reacting to domain events.
- **Independent deployment and scaling** — each service can be deployed, scaled, and tested independently. `service-search` can be scaled up during heavy query load without affecting `service-bug` command processing.
- **Technology upgrade path** — the monolithic Perl codebase is replaced incrementally, service by service, with TypeScript on the Banyan platform.
- **Testability** — each service can be behavior-tested in isolation using contract-level command/event/query stubs, with cross-service flows verified via integration tests.

### Harder

- **Distributed data consistency** — the monolith's shared-database transactions are replaced by eventual consistency across services. The bidirectional dependency invariant (bug A depends on bug B), the `noresolveonopenblockers` check (queries all dependencies before allowing resolution), and string-based version/milestone rename propagation all require careful design using read models, event-driven sagas, or accepted eventual consistency.
- **Cross-service debugging** — a single user action (e.g., updating a bug) now fans out across multiple services (bug → notification → email, bug → search → index update, bug → comment → system comment). Tracing requires correlation IDs and distributed tracing infrastructure (Jaeger).
- **Data migration complexity** — migrating 60+ tables from a monolithic MySQL/PostgreSQL database into 7 separate event stores is a non-trivial one-time operation. Historical `bugs_activity` data must be replayed or snapshotted into event streams. Custom field values (`cf_*` columns) must be mapped to a generic event encoding.
- **Operational overhead** — 7 services means 7 Docker containers, 7 sets of read model tables, 7 event stores, and a RabbitMQ message bus. This is significantly more infrastructure than the single-process Perl application with a shared database.
- **Event schema evolution** — once domain events are published, their schemas are contracts that consumers depend on. Adding fields is generally safe; removing or renaming fields requires versioned events with upcasters.
- **Group permission model complexity** — the three-way split (user owns membership, product owns configuration, bug owns enforcement) means that a permission check in `service-bug` depends on data originating in `service-user` and `service-product`. If either source's events are delayed, the enforcement read model may be stale. This is an accepted trade-off.

### Accepted Trade-offs

- **7 services is more than the minimum** (a single `service-bug` with everything else as modules) but less than the maximum (splitting flags, search, and notifications into even finer services). The 7-service split balances domain cohesion against operational complexity.
- **Eventual consistency for dependency displays** — the "blocks" direction of bug dependencies is projected from events, not guaranteed to be atomically consistent with the "depends on" direction. This is acceptable for a bug tracker where dependency displays are informational.
- **All flags in `service-attachment`** creates a cross-cutting dependency (bug-level flags require routing through the attachment service's contracts), but avoids duplicating the complex inclusion/exclusion resolution and retargeting logic across two services.
- **`service-notification` carries no domain authority** — it is a pure event consumer that can be disabled or replaced without affecting any other service's correctness. This is intentional: notification delivery is an operational concern, not a domain invariant.
- **`service-search` depends on Elasticsearch** — adding an external search engine increases the Docker stack footprint but is necessary given the complexity of Bugzilla's boolean chart search (25+ operators, pronoun substitution, change-history tracking). A TypeScript SQL port would perpetuate database coupling and limit query performance.
- **Classification entities are not event-sourced** — they are simple CRUD entities in `service-product` with no invariants or cross-service dependencies. Event sourcing would add complexity without value.

## Alternatives Considered

### Alternative 1: Fewer Services (3-service split)

**Description**: Combine related domains into fewer, larger services: `service-bug` (bugs + comments + attachments + flags), `service-platform` (users + products + classifications), and `service-infra` (search + notifications).

**Rejected because**: The whole point of the migration is to escape the monolith's god-object problem. A 3-service split would recreate the `Bug.pm` entanglement inside `service-bug` — bug creation would still need to handle comment creation, attachment metadata, and flag initialization in the same transaction. The CQRS/Event Sourcing model works best when each service owns a focused aggregate with clear invariants. The 7-service split gives each aggregate a natural home.

### Alternative 2: More Services (fine-grained per-entity)

**Description**: Separate each entity into its own service: `service-flag`, `service-classification`, `service-version`, `service-milestone`, `service-keyword`, `service-whine`, `service-dependency`.

**Rejected because**: Fine-grained services that don't have independent lifecycles, distinct authorization models, or meaningful cross-service event flows create operational overhead without architectural benefit. A `service-version` with one aggregate and three commands (`CreateVersion`, `RenameVersion`, `DeleteVersion`) is too small to justify its own Docker container, event store, and read model database. The 7-service split groups entities with cohesive domain logic and shared authorization boundaries.

### Alternative 3: Big Ball of Mud with CQRS Wrappers

**Description**: Keep a single service with CQRS/Event Sourcing decorators, splitting into microservices later if needed.

**Rejected because**: This defers the hard decomposition work and risks building a "distributed monolith" — one event-sourced service with all aggregates intermixed. The Banyan platform's subscription model (`@EventHandlerDecorator` with service-prefixed event names like `bug.Events.BugCreated`) assumes services are distinct publishers. Retrofitting service boundaries after the fact is more expensive than designing them upfront.

### Alternative 4: Domain-Event-First Without Service Boundaries

**Description**: Define domain events first, then let service boundaries emerge organically from event flow analysis.

**Rejected because**: Bugzilla's monolith has no domain events to analyze — all side effects fire synchronously within shared transactions. Without a priori boundary decisions, the decomposition would be driven by implementation convenience rather than domain cohesion, likely recreating the original module structure. The discovery → exploration → clarification pipeline explicitly identified bounded contexts; the architecture phase codifies them.
