# Product Creation Saga

## Workflow Overview

The Product Creation Saga orchestrates the multi-step process of creating a new product in the Bugzilla-to-Banyan migration. In the original Bugzilla monolith, `Product->create()` performs all steps within a single database transaction: inserting the product row, auto-creating the first version, auto-creating the default milestone, optionally creating a bug group and charting series, and firing extension hooks. In the Banyan CQRS/Event Sourcing architecture, these steps are decomposed into a saga (process manager) that issues commands across three services — `service-product`, `service-user`, and `service-search` — tolerating eventual consistency and providing compensation logic for partial failures.

The saga is triggered by an admin action and proceeds through three mandatory steps (product, version, milestone) and up to three optional steps (bug group, group control map, charting series). The `ProductCreated` event is emitted when the product aggregate is first saved; downstream services receive it along with subsequent `VersionCreated` and `MilestoneCreated` events via the RabbitMQ message bus. By the time a user navigates to the product in the UI, all projections are consistent.

**ADR-013 impact**: Version and milestone references in bugs use IDs, not denormalized strings. This eliminates the need for rename-propagation events to service-bug — `VersionRenamed` and `MilestoneRenamed` are handled entirely within service-product's projections.

## Trigger

**User action**: An administrator creates a new product through the admin API or UI.

The admin submits a `CreateProduct` command via the API gateway, providing:

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `name` | string | — | Required, unique, max `MAX_PRODUCT_SIZE` |
| `description` | string | — | Required, non-blank |
| `classificationId` | string | `"1"` | Defaults to the default classification |
| `defaultMilestone` | string | — | Required; the milestone value to create |
| `version` | string | `"unspecified"` | First version value |
| `allowsUnconfirmed` | boolean | `true` | Whether UNCONFIRMED status is legal |
| `isactive` | boolean | `true` | Product active on creation |

Two global configuration parameters control optional branching:

- **`makeproductgroups`** — when `true`, the saga creates a bug group in `service-user` and a `group_control_map` entry in `service-product`.
- **Charting enabled** — when `true`, the saga creates charting series in `service-search`.

## Participants

| Service | Role | Responsibility |
|---------|------|---------------|
| **service-product** | **Initiator** | Owns ProductAggregate, VersionAggregate, MilestoneAggregate. Issues `CreateProduct`, `CreateVersion`, `CreateMilestone`, `UpdateGroupControls` commands. Emits `ProductCreated`, `VersionCreated`, `MilestoneCreated`, `GroupControlsUpdated`, `ProductDeactivated` events. |
| **service-user** | **Participant** | Owns GroupAggregate. Receives `CreateGroup` command from the saga. Emits `GroupCreated` event. The group enables product-level bug visibility control. |
| **service-search** | **Participant** | Owns charting/series data. Receives `CreateChartingSeries` command. Emits `ChartingSeriesCreated` event. Series track bug counts over time per status/resolution. |

### Downstream consumers (not saga participants)

| Consumer | Events consumed | Purpose |
|----------|----------------|---------|
| `service-bug` | `ProductCreated`, `VersionCreated`, `MilestoneCreated`, `GroupControlsUpdated` | Initialize valid bug field values, update product/group read models |
| `service-notification` | `ProductCreated` | Send product-creation notifications to admins |

## Steps

### Step 1 — Create Product Aggregate
**BPMN**: `start` → `create-product` → `check-product`

The saga begins when the `CreateProduct` command is submitted to `service-product`. The ProductAggregate is created with the provided name, description, classification ID, `allowsUnconfirmed` flag, and `defaultMilestone` value. The aggregate validates uniqueness of the product name (case-insensitive collision check) and emits `ProductCreated` to its event stream.

This event is consumed by `service-bug` via an event subscription to begin initializing valid bug field values for the product.

**Failure**: If the product name is a duplicate or validation fails, the aggregate rejects the command. No compensation is needed — nothing was persisted.

### Step 2 — Auto-Create First Version
**BPMN**: `check-product` → `create-first-version` → `check-version`

Following successful product creation, the saga issues a `CreateVersion` command targeting the newly created product with the value `'unspecified'` (Bugzilla's default version constant). The VersionAggregate is created within `service-product` and emits `VersionCreated`.

A product must have at least one version — this invariant is enforced by the saga ensuring this step completes before the product is considered ready. In the original Bugzilla code, `Product->create()` calls `Bugzilla::Version->create()` directly within the same transaction.

**Failure**: If version creation fails (timeout, service error), the saga compensates by deactivating the product. The product exists but is marked inactive and unusable.

### Step 3 — Auto-Create Default Milestone
**BPMN**: `check-version` → `create-default-milestone` → `check-milestone`

The saga issues a `CreateMilestone` command for the product with the value matching `product.defaultMilestone`. The MilestoneAggregate is created and emits `MilestoneCreated`. Like versions, a product must have at least one milestone.

In the original Bugzilla, the `defaultmilestone` column on the `products` table stores the **milestone value string**, not the milestone ID. In Banyan (per ADR-013), this is changed to an ID-based reference — the product aggregate stores `defaultMilestoneId`, and the read model resolves it to the milestone name.

**Failure**: If milestone creation fails, the saga compensates by deactivating the product (and implicitly marking the created version as part of an inactive product).

### Step 4 (Optional) — Create Bug Group
**BPMN**: `check-milestone` → `gateway-optional-group` → `create-bug-group`

If the `makeproductgroups` global parameter is enabled, the saga issues a `CreateGroup` command to `service-user`. This creates a new Bugzilla group named after the product (e.g., `"ProductNamebugs"`). The group enables product-level bug visibility control — bugs in this product can be restricted to group members.

`service-user` emits `GroupCreated`, which the saga reacts to by proceeding to the next step.

**Proposed command**: `user.Commands.CreateGroup` is implied by the group model in `service-user` but not explicitly listed in the product-hierarchy exploration slice. It is a logical consequence of the group ownership model confirmed in R5/R10.

### Step 5 (Optional) — Create Group Control Map Entry
**BPMN**: `create-bug-group` → `create-group-control-map`

Immediately after the group is created, the saga issues an `UpdateGroupControls` command to `service-product`, creating a `group_control_map` entry linking the new group to the product with:

- `membercontrol = DEFAULT` — group members see product bugs by default
- `othercontrol = NA` — non-members cannot see product bugs
- `entry = true` — only group members can file bugs in this product

The `GroupControlsUpdated` event is emitted and projected into the `ProductGroupControlsReadModel` consumed by `service-bug` for authorization enforcement (R5, Q5).

### Step 6 (Optional) — Create Charting Series
**BPMN**: `gateway-optional-series` → `create-charting-series`

If charting is enabled, the saga issues a `CreateChartingSeries` command to `service-search`. Bugzilla's `_create_series()` creates one series per bug-status × resolution combination, plus an aggregate "all open" series. These series track bug counts over time for the new product and populate the `series`, `series_data`, and `series_categories` tables.

**Proposed command**: `search.Commands.CreateChartingSeries` is not in any exploration service doc. It is derived from the Bugzilla source code's `_create_series()` method on `Product.pm`. If service-search does not support charting in the initial migration, this step can be disabled without affecting product functionality.

### Step 7 — Finalization
**BPMN**: `create-charting-series` / `gateway-optional-series` → `finalize-product` → `end-success`

After all mandatory and optional steps complete, the saga performs post-creation finalization:

1. **Clear configuration caches** — memcached config cache in the original Bugzilla; in Banyan, this maps to invalidating read model caches for product lists and user-product access.
2. **Clear user-product caches** — ensure all users see the new product in their available products list.
3. **Fire extension hooks** — the Bugzilla `product_end_of_create` hook maps to Banyan event subscriptions. Any registered subscription handlers in `service-product` are invoked.

The product is now fully created and available for bug filing.

> **Timing note**: The `ProductCreated` event was emitted at Step 1 when the aggregate was first saved. Downstream consumers (service-bug) receive it via RabbitMQ subscription and begin initializing projections. The `VersionCreated` and `MilestoneCreated` events follow in rapid succession. Due to eventual consistency, all three events are typically processed within seconds. The Banyan platform's `waitForProjection` semantics ensure that any read-model query after saga completion reflects the complete state.

## Event Flow

```
service-product                    service-user                  service-search
     │                                  │                              │
     │ ① CreateProduct                  │                              │
     │──ProductCreated──────────────────│──────────────────────────────│
     │         │                        │                              │
     │ ② CreateVersion                  │                              │
     │──VersionCreated──────────────────│──────────────────────────────│
     │         │                        │                              │
     │ ③ CreateMilestone                │                              │
     │──MilestoneCreated────────────────│──────────────────────────────│
     │         │                        │                              │
     │ ④ [if makeproductgroups]         │                              │
     │ CreateGroup ──────────────────→  │                              │
     │         │              GroupCreated                             │
     │ ⑤ UpdateGroupControls            │                              │
     │──GroupControlsUpdated────────────│──────────────────────────────│
     │         │                        │                              │
     │ ⑥ [if charting]                  │                              │
     │ CreateChartingSeries ────────────────────────────────────────→  │
     │         │                                     ChartingSeriesCreated
     │         │                        │                              │
     │ ⑦ Finalize                       │                              │
     │ ✓ end-success                    │                              │
```

### Key events broadcast to consumers

| Event | Source Service | Consumers | Purpose |
|-------|---------------|-----------|---------|
| `product.Events.ProductCreated` | service-product | service-bug, service-notification | Initialize valid bug field values; notify admins |
| `product.Events.VersionCreated` | service-product | service-bug | Add version to product's valid version list |
| `product.Events.MilestoneCreated` | service-product | service-bug | Add milestone to product's valid milestone list |
| `product.Events.GroupControlsUpdated` | service-product | service-bug | Update `ProductGroupControlsReadModel` for auth enforcement |
| `user.Events.GroupCreated` | service-user | service-product (saga) | Saga continuation trigger |
| `search.Events.ChartingSeriesCreated` | service-search | — | Analytics/reporting (no downstream consumers in current architecture) |
| `product.Events.ProductDeactivated` | service-product | service-bug, service-notification | Compensation: remove from valid products, prevent new bug filing |

### Cross-service subscription model

In Banyan, cross-service handoff happens exclusively through **event subscriptions** (`@EventHandlerDecorator`):

- `service-bug` subscribes to `product.Events.*` and `user.Events.GroupCreated` to maintain its own read models
- `service-product` subscribes to `user.Events.GroupCreated` as part of the saga continuation (the saga process manager reacts to the event)
- `service-search` subscribes to no product events in this workflow — it receives a command, not an event

No synchronous HTTP calls occur between services during this saga. All communication is asynchronous via RabbitMQ.

## Error Handling & Compensation

### Compensation strategy

The saga follows a **compensate-on-failure** pattern. If any mandatory step fails, compensating actions return the system to a consistent state.

| Failed Step | Compensation Action | Notes |
|-------------|-------------------|-------|
| Step 1: CreateProduct | None | Aggregate was never persisted; no cleanup needed |
| Step 2: CreateVersion | `DeactivateProduct` on the created product | Product exists without required version → mark inactive |
| Step 3: CreateMilestone | `DeactivateProduct` | Product + version exist without required milestone → mark inactive |
| Step 4: CreateGroup (optional) | **Strict**: `DeactivateProduct`. **Lenient**: log & continue | In lenient mode, admin can create the group manually later |
| Step 5: UpdateGroupControls (optional) | **Strict**: delete group + `DeactivateProduct`. **Lenient**: log & continue | Group exists but has no effect without control map |
| Step 6: CreateChartingSeries (optional) | Log & continue | Charting is non-critical; only affects reports |

### Compensation flow (BPMN)

The BPMN models a single `compensate-deactivate` service task that issues `DeactivateProduct` to the product aggregate. The `ProductDeactivated` event cascades through read models:

1. `rm_product_list` marks the product as inactive (removed from navigation dropdowns)
2. `service-bug` read models remove the product from valid bug-filing targets
3. Any created versions/milestones remain in the event store but are inaccessible via inactive-product read models

For a production implementation, the saga process manager should maintain a **compensation log** recording each completed step, enabling targeted cleanup:

```
compensation_log: [
  { step: "create-product", productId: "abc-123", status: "completed" },
  { step: "create-first-version", versionId: "def-456", status: "completed" },
  { step: "create-default-milestone", milestoneId: "ghi-789", status: "failed" }
]
→ compensate: DeactivateProduct(abc-123)
```

### Retry strategy

| Failure type | Retry behavior | Max attempts |
|-------------|---------------|-------------|
| Transient (network, timeout) | Exponential backoff: 1s → 2s → 4s | 3 |
| Validation (duplicate name, bad data) | No retry — fail immediately with error details | 0 |
| Optional step failure | Single retry; if still fails, log warning and continue | 1 |

### Idempotency requirements

All commands in the saga **must be idempotent** to support retry semantics:

| Command | Idempotency mechanism |
|---------|----------------------|
| `CreateProduct` | If product with same name exists, return error (name is unique) |
| `CreateVersion` | If version with same `(productId, value)` exists, return success |
| `CreateMilestone` | If milestone with same `(productId, value)` exists, return success |
| `CreateGroup` | If group with same name exists, return existing group |
| `UpdateGroupControls` | If entry for `(productId, groupId)` exists, update in place |
| `CreateChartingSeries` | If series for same product/status/resolution exists, skip |
| `DeactivateProduct` | If already inactive, return success (no-op) |

## Failure Modes

### FM-1: Duplicate Product Name
- **Trigger**: `CreateProduct` command rejected — a product with the same name already exists.
- **Detection**: Aggregate enforces unique name invariant (case-insensitive collision check).
- **Recovery**: No compensation needed. Return structured error to admin with the conflicting product name.
- **User impact**: Admin must choose a different name.

### FM-2: Version Creation Timeout
- **Trigger**: `CreateVersion` command times out or `service-product` is temporarily unavailable.
- **Detection**: Saga process manager detects command timeout (configurable threshold, e.g., 30 seconds).
- **Recovery**: Retry with exponential backoff (up to 3 attempts). If all retries fail, compensate by issuing `DeactivateProduct`. Admin can retry product creation manually.
- **User impact**: Product exists but is inactive and not visible for bug filing.

### FM-3: Milestone Creation Validation Failure
- **Trigger**: `CreateMilestone` fails due to invalid `defaultMilestone` value or database constraint violation.
- **Detection**: Command handler returns validation error.
- **Recovery**: Single retry (in case of transient issue). If still fails, compensate with `DeactivateProduct`. The already-created version remains in the event store but is inaccessible via the inactive product.
- **User impact**: Same as FM-2.

### FM-4: Group Creation Failure (Optional)
- **Trigger**: `CreateGroup` command to `service-user` fails (service unavailable, group name conflict, network partition).
- **Detection**: Command handler returns error or saga timeout.
- **Recovery**:
  - **Strict mode**: Compensate with `DeactivateProduct`. Entire product creation is rolled back.
  - **Lenient mode** (recommended): Log warning, continue to charting/finalization. The product is usable but lacks the bug group. Admin can create the group manually via the group management UI.
- **User impact**: In lenient mode, bugs in this product are not group-restricted until the group is manually created.

### FM-5: Charting Series Creation Failure (Optional)
- **Trigger**: `CreateChartingSeries` command to `service-search` fails (service not deployed, network error).
- **Detection**: Command handler returns error or saga timeout.
- **Recovery**: Log warning and continue. Charting series are non-critical — they affect only reporting charts, not bug filing or product functionality. Series can be created retroactively.
- **User impact**: Product charts show no data until series are created. Bug filing is unaffected.

### FM-6: ProductCreated Event Lost in Transit
- **Trigger**: The `ProductCreated` event is published to RabbitMQ but not consumed by `service-bug` due to bus issues, subscription handler crash, or consumer lag.
- **Detection**: `service-bug`'s product field read model does not include the new product.
- **Recovery**: Banyan's event bus guarantees **at-least-once delivery**. If the subscription handler is offline, events are queued and delivered when it reconnects. Read models eventually catch up. No manual intervention needed.
- **User impact**: Brief window (< 2 seconds under normal load) where the product doesn't appear in bug-filing dropdowns.

### FM-7: Concurrent Product Creation Race
- **Trigger**: Two admins simultaneously submit `CreateProduct` with the same product name.
- **Detection**: The second command fails the aggregate's unique name invariant.
- **Recovery**: Return error to the second admin with details of the conflict. No compensation needed for the failed attempt. The first admin's product is unaffected.
- **User impact**: Second admin must choose a different name or wait and retry.

### FM-8: Saga Process Manager Crash (Orphaned Saga)
- **Trigger**: The saga process manager crashes mid-workflow (e.g., after Step 2 but before Step 3).
- **Detection**: Saga state is persisted to the event store. On restart, the manager detects incomplete sagas via a periodic sweep or startup reconciliation.
- **Recovery**: Resume saga from the last **completed** step using persisted state. If a step was in-flight when the crash occurred, retry it idempotently (all commands must be idempotent — see above). If the step had partially completed, the idempotency guarantee ensures no duplicate side effects.
- **User impact**: Product creation is delayed until the process manager restarts and reconciles (typically seconds to minutes depending on deployment).

### FM-9: Projection Lag After Step Completion
- **Trigger**: The saga completes successfully, but `service-bug`'s read models haven't processed all events yet.
- **Detection**: A user attempts to create a bug in the new product and receives a "product not found" error from service-bug.
- **Recovery**: No recovery needed — eventual consistency will resolve within seconds. The admin UI should use `waitForProjection` semantics after the saga completes, polling the product's read model until it confirms availability before enabling bug filing.
- **User impact**: Brief window where the product is visible in the product service's listings but not yet in service-bug's valid fields. Mitigated by UI waiting for projection confirmation.

## Data Consistency Model

### Eventual consistency boundaries

The Product Creation Saga spans three services. Consistency guarantees vary by boundary:

| Boundary | Consistency Model | Notes |
|----------|------------------|-------|
| **Within service-product** (ProductAggregate, VersionAggregate, MilestoneAggregate) | **Strong per-aggregate** | Each aggregate has its own event stream with strong consistency. Cross-aggregate operations (product + version + milestone) are eventually consistent with each other. |
| **service-product ↔ service-user** (group + group_control_map) | **Eventual** | `CreateGroup` in service-user → `GroupCreated` event → `UpdateGroupControls` in service-product. The product's group controls are eventually consistent with the user service's group state. Lag: typically < 1 second. |
| **service-product ↔ service-search** (charting series) | **Best-effort / fire-and-forget** | Charting series creation is non-critical. Failure does not affect product functionality. |
| **service-product → service-bug** (projections) | **Eventual** | service-bug projects `ProductCreated`, `VersionCreated`, `MilestoneCreated` events into its own read models. There is a projection lag between product creation and bug-service readiness. |

### Projection lag tolerance

- The product list read model (`rm_product_list`) is projected **within** `service-product` and updates quickly (typically < 500ms).
- `service-bug`'s product field configuration read model updates **asynchronously** via event subscription. A user attempting to create a bug in the new product immediately after creation may encounter stale projections.
- **Mitigation**: The admin UI should use `waitForProjection` semantics — polling or WebSocket notification — to confirm that service-bug's projection reflects the new product before enabling bug filing.
- **Acceptable lag**: < 2 seconds under normal load. If lag exceeds 10 seconds, alert operations.

### ADR-013: ID-based references (consistency impact)

Per ADR-013 (from clarification Q8), version and milestone references in bugs use **IDs, not denormalized strings**. This has significant consistency implications:

| Aspect | Before (string-based) | After (ID-based) |
|--------|----------------------|-------------------|
| Version rename propagation | Requires `UPDATE bugs SET version = ?` across all bugs | No propagation needed — read model resolves ID → name |
| Milestone rename propagation | Requires `UPDATE bugs SET target_milestone = ?` AND `UPDATE products SET defaultmilestone = ?` | No propagation needed — read model resolves ID → name |
| Cross-service consistency | Synchronous (within transaction) | Eventual (read model lag) |
| Data integrity risk | Orphaned string references if rename fails | None — ID reference is stable |

This eliminates an entire class of consistency issues present in the original Bugzilla codebase. The tradeoff is that version/milestone names must be resolved via read model lookups at query time, adding a trivially small latency to bug list/detail queries.

### Aggregate boundary invariants

Two invariants span aggregate boundaries and require saga-level enforcement:

1. **"Product must have ≥ 1 version"**: Enforced by the saga — `CreateVersion` is a mandatory step. If it fails, the product is deactivated. Deletion of the last version is blocked by a `MinimumVersionPolicy` in the `DeleteVersion` command handler.

2. **"Product must have ≥ 1 milestone"**: Same pattern — `CreateMilestone` is mandatory. `MinimumMilestonePolicy` prevents deletion of the last milestone.

These invariants cannot be enforced within a single aggregate boundary (Product vs. Version/Milestone are separate aggregates). The saga + policy combination provides the guarantee: the saga ensures creation, and policies prevent invalid deletion.

### Cross-service event ordering

Banyan's RabbitMQ message bus delivers events in the order they are published **per aggregate stream**. However, cross-aggregate and cross-service ordering is not guaranteed. This means:

- `ProductCreated` may arrive at service-bug before or after `VersionCreated` — the subscription handlers must tolerate out-of-order delivery.
- Read models must be designed for **incremental projection** — each event adds information without assuming prior events have been processed.
- The `waitForProjection` helper in Banyan polls the read model until a specific condition is met, abstracting away ordering concerns.

## Decision Rules Referenced

| Gateway ID | Gateway label | Rule ID | Rule citation |
|------------|---------------|---------|---------------|
| check-product | Product aggregate created successfully? | [BR-product-001](../decision-rules.md#br-product-001) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#business-rule-1 |
| check-product | Product aggregate created successfully? | [BR-product-ST-new-Active](../decision-rules.md#br-product-st-new-active) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#state-machines |
| check-product | Product aggregate created successfully? | [BR-product-012](../decision-rules.md#br-product-012) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#business-rule-12 |
| check-version | First version created successfully? | [BR-product-002](../decision-rules.md#br-product-002) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#business-rule-2 |
| check-version | First version created successfully? | [BR-product-006](../decision-rules.md#br-product-006) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#business-rule-6 |
| check-version | First version created successfully? | [BR-product-012](../decision-rules.md#br-product-012) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#business-rule-12 |
| check-milestone | Default milestone created successfully? | [BR-product-007](../decision-rules.md#br-product-007) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#business-rule-7 |
| check-milestone | Default milestone created successfully? | [BR-product-012](../decision-rules.md#br-product-012) | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md#business-rule-12 |
| gateway-optional-group | makeproductgroups global param enabled? | [unciteable: TODO] | searched: BR-product-*, BR-user-*; no rule formalizes the makeproductgroups global parameter — branch is configuration-driven. BR-product-012 governs compensation if step fails. |
| gateway-optional-series | Charting series enabled? | [unciteable: TODO] | searched: BR-product-*, BR-search-*; no rule formalizes "charting enabled" global parameter — branch is configuration-driven. BR-product-012 governs compensation if step fails. |
