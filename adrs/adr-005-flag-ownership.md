# ADR-005: All Flag Logic Resides in service-attachment

## Status

Accepted

## Context

Bugzilla's flag system is a unified domain concept that spans two target surfaces: bugs and attachments. In the existing Bugzilla schema, the `flags` table stores every flag instance in a single table with a `target_type` discriminator (`'b'` for bug, `'a'` for attachment) and a nullable `attach_id` foreign key — when `attach_id IS NULL`, the flag targets the bug itself; otherwise it targets the specified attachment.

The flag domain includes several tightly interwoven subsystems that are identical regardless of target type:

1. **Flag types** (`flagtypes` table) — define review/approval workflows (e.g., "review", "approval", "feedback") with attributes like `is_requestable`, `is_requesteeble`, `is_multiplicable`, `grant_group_id`, and `request_group_id`. A single flag type definition can apply to bugs, attachments, or both.

2. **Inclusion/exclusion rules** (`flaginclusions` / `flagexclusions` tables) — scope flag type availability to specific product/component combinations using a junction table pattern with NULL wildcards. The resolution logic (`INNER JOIN flaginclusions` + `LEFT JOIN flagexclusions` + `WHERE e.type_id IS NULL`) is target-type-agnostic.

3. **State machine** — flags transition through `?` (requested) → `+` (granted) / `-` (denied) / `X` (cleared). The transition rules, grant/request group checks, and setter tracking are the same whether the flag sits on a bug or an attachment.

4. **Retargeting** — when a bug moves between products or components, flags (both bug-level and attachment-level) may become invalid and need to be retargeted or cleaned up. This logic (`Flag->force_retarget`) operates across both target types in a single pass.

5. **Multiplicable flag logic** — multiple flags of the same type can coexist on a single target when `is_multiplicable` is true. The de-duplication and creation logic is shared.

During architecture design, a key question arose (Q4 in the clarification gap analysis): should bug-level flags live in `service-bug` and attachment-level flags in `service-attachment`, or should all flag logic be co-located in a single service?

Splitting flags across two services would require:
- Duplicating inclusion/exclusion resolution logic in both services.
- Duplicating grant group / request group permission enforcement in both services.
- Duplicating the `? → + / - / X` state machine in both services.
- Coordinating retargeting when a bug changes product (attachment-level flags also need retargeting, requiring cross-service orchestration).
- Sharing `FlagType` definitions between services (or introducing a third flag-type service).

## Decision

**All flag logic — both bug-level and attachment-level — resides in `service-attachment`.**

The `target_type` attribute (`bug` vs `attachment`) is modeled as a field on the flag entity, not as a service boundary. Bug-level flags are represented as flag instances where `attach_id` is null and `target_type` is `bug`, exactly mirroring the existing Bugzilla data model.

### Concrete mechanics

- **Contracts**: `service-attachment` publishes both attachment-flag commands (`SetAttachmentFlag`, `ClearAttachmentFlag`) and bug-flag commands (`SetBugFlag`, `ClearBugFlag`) in its contract package (`@banyanai/service-attachment-contracts`).
- **Events**: The service emits `BugFlagSet`, `BugFlagCleared`, `BugFlagGranted`, `BugFlagDenied` for bug-level flags, alongside the attachment-flag equivalents. These are broadcast on the message bus.
- **Read models for service-bug**: `service-bug` subscribes to `service-attachment.Events.BugFlagSet`, `service-attachment.Events.BugFlagCleared`, etc., and projects a `BugFlagListReadModel` that provides bug-level flag visibility without cross-service queries.
- **FlagType aggregate**: Lives in `service-attachment` as a single authority for flag type definitions, inclusion/exclusion rules, and group permission configuration.
- **Retargeting**: When `service-bug` emits `BugMoved` (bug changes product/component), `service-attachment` subscribes and runs retargeting for all flags — both bug-level and attachment-level — in a single handler, using its local flag type inclusion/exclusion data.

### Authorization

- **Layer 1**: Commands carry permissions like `flags:set` and `flags:request` on the `@Command` decorator, enforced at the API gateway.
- **Layer 2**: `CanSetFlagPolicy` and `FlagTypeApplicabilityPolicy` run inside `service-attachment` command handlers, checking grant/request groups and inclusion/exclusion rules respectively.

## Consequences

### What becomes easier

- **Single source of truth for flag types**: One service owns all flag type definitions, inclusion/exclusion rules, and group permission configuration. No cross-service coordination for flag type CRUD.
- **Unified retargeting**: When a bug moves between products, a single subscription handler in `service-attachment` retargets or cleans up all flags (bug-level and attachment-level) in one pass, avoiding distributed saga complexity.
- **Simpler flag state machine**: One implementation of the `? → + / - / X` transitions, grant group checks, requestee validation, and multiplicable flag logic.
- **Simpler inclusion/exclusion resolution**: The SQL-like resolution logic exists in one place and applies uniformly to both target types.
- **Consistent notification routing**: All flag change events originate from one service, making it straightforward for `service-notification` to subscribe and render flag-related emails (review requests, grant/deny notifications, CC list alerts).

### What becomes harder

- **Larger contract surface for service-attachment**: The `@banyanai/service-attachment-contracts` package must include bug-flag commands, queries, and events in addition to attachment-specific ones. This increases the package size and the number of published event types.
- **Cross-service read model dependency**: `service-bug` depends on `service-attachment` events to project bug-level flags. If `service-attachment` is down, bug-level flag data in `service-bug` read models becomes stale. This is an accepted trade-off for eventual consistency.
- **Semantic naming tension**: Commands like `SetBugFlag` living in `service-attachment-contracts` may feel surprising to developers expecting bug-related operations in `service-bug-contracts`. Documentation and API gateway routing must make the ownership clear.
- **Testing scope**: `service-attachment` behavioral tests must cover both bug-level and attachment-level flag scenarios, including cross-cutting concerns like retargeting on product move and obsolete cascade to pending flags.

### Accepted trade-offs

- **Eventual consistency of bug-level flag displays** in `service-bug` is acceptable because flags are informational metadata — they affect review/approval workflows and display, not data integrity invariants. A brief delay in flag read model projection does not corrupt data.
- **Increased contract surface area** is preferred over the complexity of splitting a single domain concept across two services. The flag subsystem is cohesive; splitting it would introduce a distributed monolith anti-pattern where every flag change requires coordination between services.

## Alternatives Considered

### Alternative 1: Split flags — bug-level flags in service-bug, attachment-level flags in service-attachment

**Rejected because**:
- Flag types, inclusion/exclusion rules, and the state machine are identical for both target types. Splitting would require either duplicating this logic or introducing a shared library (creating tight coupling through code sharing rather than event-based loose coupling).
- Retargeting when a bug changes product would require `service-bug` to coordinate with `service-attachment` to also retarget attachment-level flags — a distributed saga for what is currently a single SQL pass.
- The `FlagType` entity would need to be owned by one service and referenced by the other, creating a cross-service dependency for every flag operation.

### Alternative 2: Dedicated service-flags microservice

**Rejected because**:
- Adding an eighth service for a subsystem that is already tightly coupled to both bugs and attachments introduces operational overhead (Docker container, message bus routing, monitoring) disproportionate to the domain complexity.
- Flag operations are always in the context of a bug or attachment — there are no standalone flag workflows. A dedicated service would be a pass-through for context that lives elsewhere.
- The retargeting problem remains: `service-flags` would need to subscribe to `BugMoved` from `service-bug` and `AttachmentCreated` from `service-attachment`, and would need read access to product/component configuration for inclusion/exclusion resolution — creating a hub of dependencies rather than reducing coupling.
