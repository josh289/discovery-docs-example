# ADR-003: Global Status Workflow with Product-Level Flags

## Status

Accepted

## Context

Bugzilla's bug lifecycle is governed by a configurable status workflow — a directed graph of valid transitions between status values (e.g., `UNCONFIRMED → NEW → ASSIGNED → RESOLVED → VERIFIED → CLOSED`). This workflow is stored in two tables:

- **`bug_status`** — one row per status value, with `is_open` (boolean), `isactive`, `sortkey`, and `value` (the status name).
- **`status_workflow`** — directed edges with `old_status` (FK, nullable), `new_status` (FK), `require_comment` (boolean), and `isactive`. An `old_status` of `NULL` represents transitions available at bug creation time.

**Critical observation: the `status_workflow` table has no `product_id` column.** Every product in a Bugzilla installation shares the same set of status values and the same transition graph. The workflow is global by design.

However, Bugzilla does support limited per-product behavior variation:

1. **`allows_unconfirmed`** — a boolean on the product entity that controls whether newly filed bugs start as `UNCONFIRMED` or skip directly to `NEW`. This is a product-level flag, not a workflow modification.
2. **Group-based access control on transitions** — the `group_control_map` matrix (per-product) determines which user groups can perform which operations on bugs in that product. This restricts *who* can make certain transitions, not *which* transitions exist.
3. **`commenton*` parameters** — site-wide settings that require comments on specific transition types, enforced in `Bugzilla::Bug` rather than `Status.pm`.

This creates an architectural decision for the Banyan CQRS/Event Sourcing migration: should the migrated system replicate this global-workflow-plus-product-flags pattern, or should it offer per-product workflow definitions?

### Why this decision matters now

The choice between global vs. per-product workflows fundamentally shapes:

- **The Bug aggregate's state machine** — does it load one global workflow definition or resolve a per-product one?
- **The `TransitionBugStatus` command handler** — how does it validate that a requested transition is legal?
- **The `StatusWorkflowConfig` data model** — one aggregate/read model or many?
- **Administrative complexity** — one workflow to maintain or N product-specific copies?
- **Cross-service event design** — does `service-product` need to emit workflow-change events?

This decision blocks specification of the `TransitionBugStatus` command, the `StatusWorkflowReadModel`, and the `ValidStatusTransitionPolicy`.

## Decision

**We adopt a single global status workflow consumed by `service-bug`, with per-product customization limited to the `allows_unconfirmed` boolean flag and group-based access control on transitions applied at the policy/authorization layer.**

Specifically:

### 1. One `StatusWorkflowConfig` aggregate / read model

The workflow graph is modeled as a single configuration entity owned by `service-bug`. It contains:

- The set of valid status values (name, `is_open`, `isactive`, `sortkey`)
- The directed edge list (`old_status → new_status`, `require_comment`, `isactive`)
- Creation-time statuses (edges where `old_status` is null)

This is **not per-product**. There is exactly one `StatusWorkflowConfig` for the entire installation.

### 2. Product-level behavior flags on the Product aggregate

The `Product` aggregate (owned by `service-product`) carries:

- `allows_unconfirmed: boolean` — whether bugs in this product start as `UNCONFIRMED` (true) or `NEW` (false)

This flag is consumed by `service-bug` via a `ProductSettingsReadModel` projected from `service-product` events.

### 3. Group-based transition restrictions at Layer 2

Per-product group restrictions on transitions are enforced through Banyan's existing two-layer authorization model:

- **Layer 1**: `bugs:transition_status` permission on the `TransitionBugStatus` command contract — the gateway checks that the user has this permission before the handler runs.
- **Layer 2**: `ValidStatusTransitionPolicy` on the command handler — checks: (a) the transition exists in the global workflow graph, (b) the user's group memberships (sourced from the `ProductGroupControlsReadModel`) permit this transition for bugs in this product, (c) a comment is provided if `require_comment` is true on the edge.

### 4. Workflow data remains data-driven, not hardcoded

Consistent with Bugzilla's design, the workflow graph is stored as configuration data (read model seeded from admin commands), not baked into aggregate code. The `TransitionBugStatus` command handler loads the `StatusWorkflowReadModel` and validates the transition dynamically.

## Workflow Data Model

### StatusWorkflowConfig (configuration aggregate / read model)

```
StatusWorkflowConfig
├── statuses: StatusValue[]
│   ├── name: string              // e.g., "UNCONFIRMED", "NEW", "RESOLVED"
│   ├── isOpen: boolean           // whether this is an open status
│   ├── isActive: boolean         // whether this status is currently in use
│   ├── sortKey: number           // display ordering
│   └── isStatic: boolean         // cannot be removed (e.g., UNCONFIRMED)
│
└── transitions: StatusTransition[]
    ├── fromStatus: string | null // null = available at creation
    ├── toStatus: string
    ├── requireComment: boolean
    └── isActive: boolean
```

### ProductSettingsReadModel (projected from service-product events)

```
ProductSettingsReadModel
├── productId: string
├── productName: string
├── allowsUnconfirmed: boolean    // the key per-product workflow flag
└── groupControls: GroupControl[] // for Layer 2 policy evaluation
    ├── groupId: string
    ├── memberControl: string
    ├── otherControl: string
    ├── canEdit: boolean
    ├── editBugs: boolean
    └── canConfirm: boolean
```

### TransitionBugStatus command flow

```
1. Gateway: Check bugs:transition_status permission (Layer 1)
2. Handler loads BugAggregate (current status)
3. Handler loads StatusWorkflowReadModel
4. Handler validates transition exists and is active
5. Handler loads ProductSettingsReadModel for bug's product
6. ValidStatusTransitionPolicy evaluates:
   a. If transitioning FROM UNCONFIRMED → check product.allowsUnconfirmed
   b. If transitioning TO creation-only status → reject (not a valid edge)
   c. If product group controls restrict this action → check user groups
   d. If edge requires comment → verify comment is present
7. Aggregate applies transition, emits BugStatusTransitioned
8. Read model projections update BugListReadModel, BugDetailReadModel
```

### Admin workflow management commands

| Command | Effect |
|---------|--------|
| `CreateStatusValue` | Adds a new status to the global workflow |
| `UpdateStatusValue` | Modifies `isOpen`, `isActive`, `sortKey` |
| `RemoveStatusValue` | Deactivates a status (does not delete — in-flight bugs) |
| `CreateStatusTransition` | Adds a new edge (old_status → new_status) |
| `UpdateStatusTransition` | Modifies `requireComment`, `isActive` on an edge |
| `RemoveStatusTransition` | Deactivates a transition edge |
| `SetProductAllowsUnconfirmed` | Updates the product-level flag (in `service-product`) |

### Events emitted

| Event | Payload | Consumers |
|-------|---------|-----------|
| `bug.Events.BugStatusTransitioned` | `bugId`, `fromStatus`, `toStatus`, `resolution?`, `commentRequired` | `service-notification`, `service-search` |
| `bug.Events.StatusWorkflowConfigured` | Full workflow snapshot | `service-notification` (admin alerts) |

## Alternatives Considered

### A1: Per-product workflow definitions

Each product would have its own copy of the workflow graph, allowing fully independent status values and transitions per product.

**Rejected because:**
- Bugzilla's `status_workflow` table is explicitly global (no `product_id` column). There is no source-system justification for per-product workflows.
- Administrative burden explodes: an admin changing a transition must update N product workflows instead of one.
- Read model complexity increases: the `TransitionBugStatus` handler must resolve which workflow to load, adding a product-keyed lookup for every transition.
- Cross-product bug moves (changing a bug's product) would require status reconciliation against the new product's workflow — a complex edge case with no source-system precedent.
- The actual per-product variation in Bugzilla is limited to a single boolean and group permissions — both are naturally handled as overrides on a global graph.

### A2: Hardcoded state machine in the Bug aggregate

The workflow graph would be encoded as a TypeScript state machine within the `BugAggregate` class, with transitions defined in code rather than data.

**Rejected because:**
- Bugzilla's workflow is fully configurable at runtime by administrators. Hardcoding would remove this capability.
- Adding a new status or transition would require a code deployment rather than an admin command.
- The `status_workflow` table IS the state machine definition — data-driven design must be preserved to maintain feature parity.

### A3: Workflow as a separate microservice

A dedicated `service-workflow` would own the status graph and expose query APIs for `service-bug` to validate transitions.

**Rejected because:**
- The workflow is a configuration entity, not a business domain with its own transactions and invariants. It belongs to the service that consumes it.
- Adding a service-to-service synchronous call for every status transition introduces latency and a failure mode.
- A read model projected within `service-bug` provides the same data with local access and no network dependency.

## Consequences

### What becomes easier

- **Simpler workflow management**: Administrators maintain one global workflow graph. Changes propagate immediately to all products without cascading updates.
- **Consistent aggregate logic**: The `TransitionBugStatus` handler always loads the same `StatusWorkflowReadModel`. No product-keyed branching in the aggregate's state machine.
- **Clean separation of concerns**: The workflow graph (data) is separate from who can use it (authorization). Products customize behavior through flags and group controls, not by forking the graph.
- **Admin commands are straightforward**: `CreateStatusValue` and `CreateStatusTransition` modify one global config. No multi-product transactional updates.
- **Cross-product bug moves**: Moving a bug between products doesn't require status reconciliation — the valid transitions don't change. Only the `allows_unconfirmed` and group-control policies may differ.
- **Natural fit for Banyan's authorization layers**: `allows_unconfirmed` maps to a Layer 2 policy check; group-based transition restrictions map to Layer 2 `ValidStatusTransitionPolicy`. The workflow graph itself is authorization-independent.

### What becomes harder

- **Policy evaluation on every transition**: The `TransitionBugStatus` handler must evaluate group-based policies for every status change, requiring access to the `ProductGroupControlsReadModel`. This adds a read-model dependency to the critical path of every transition command.
- **No per-product transition customization**: If a future requirement emerges for product-specific transitions (e.g., product A allows `NEW → IN_PROGRESS` but product B does not), this must be modeled as a group control restriction (who can perform it) rather than removing the edge from the graph. This is semantically different — the transition still "exists" but is restricted by policy.
- **`allows_unconfirmed` is a special case**: The initial status of a bug depends on the product's `allows_unconfirmed` flag. The `CreateBug` command handler must load the `ProductSettingsReadModel` to determine whether the initial status should be `UNCONFIRMED` or `NEW`. This couples bug creation to a product read model.
- **Workflow changes affect all products simultaneously**: Deactivating a status or transition takes effect across every product immediately. There is no graduated rollout per product. Administrators must consider the global impact of any workflow change.
- **Testing matrix**: The `ValidStatusTransitionPolicy` must be tested across combinations of: global workflow state × product `allows_unconfirmed` × user group memberships. The global nature of the workflow does not reduce the test surface for policy interactions.

### Trade-offs accepted

- **Expressiveness vs. simplicity**: We sacrifice the ability to define truly independent per-product workflows in exchange for a single, maintainable global graph. This matches Bugzilla's actual data model and is the right trade-off for a bug tracker where status semantics are installation-wide.
- **Policy-layer coupling vs. data-layer customization**: Per-product behavior lives in the authorization layer (Layer 2 policies) rather than the data layer (separate workflow graphs). This means the workflow read model doesn't know about product restrictions — that knowledge lives in policy classes. This is architecturally cleaner but requires policy authors to understand the full interaction between workflow rules and group controls.
- **Global admin impact**: A single workflow means a single point of administrative control. This is a feature (consistency) and a risk (one bad change affects everything). Accepted because Bugzilla operates this way today and administrators are accustomed to it.
