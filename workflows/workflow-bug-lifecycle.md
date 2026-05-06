# Workflow: Bug Lifecycle — Creation to Resolution

**Workflow ID**: `workflow-bug-lifecycle`
**Scope**: Cross-service bug lifecycle from initial filing through status transitions, resolution, and notification — covering bug creation, comments, attachments, dependency management, duplicate marking, and email notification delivery.

---

## Workflow Overview

This workflow models the complete lifecycle of a Bugzilla bug as it traverses the Evergreen CQRS/Event Sourcing microservice architecture. It spans **six services**: `service-bug` (initiator and primary owner), `service-product` (configuration and validation), `service-user` (identity and group membership), `service-comment` (discussion thread), `service-attachment` (files and flags), and `service-notification` (email delivery).

The workflow is **event-driven** — each primary action (creation, update, status transition, comment, attachment) emits domain events that are consumed by downstream services via RabbitMQ subscriptions. Cross-service handoff happens exclusively through events and locally projected read models; there are no synchronous service-to-service calls in the hot notification path (ADR-012).

The BPMN definition captures **seven sub-flows** that together represent the bug lifecycle:

1. **Bug Creation** — User files a bug; validates product access, resolves initial status, creates the aggregate, and fans out to notification, search, comment scope, and attachment scope.
2. **Comment Addition** — User adds a comment; creates the CommentAggregate, notifies stakeholders, and projects `work_time` into time-tracking.
3. **Attachment Addition** — User uploads a file; stores binary in object storage, creates the AttachmentAggregate, notifies stakeholders, and creates a system comment.
4. **Field Update** — User changes bug fields; applies validator dependency ordering, emits update events with full diff payloads, and notifies stakeholders.
5. **Status Transition** — User changes bug status; validates against the data-driven workflow, checks the `noresolveonopenblockers` invariant for FIXED resolution, applies the transition, and cascades dependency notifications on resolution.
6. **Duplicate Marking** — User marks a bug as duplicate; transitions status, creates system comments on both bugs, and notifies stakeholders.
7. **Dependency Addition** — User adds a dependency; validates no loops/cycles, creates the relationship, and notifies stakeholders.

---

## Participants

| Service | Role | Responsibility in This Workflow |
|---------|------|---------------------------------|
| `service-bug` | **Initiator** | Owns the BugAggregate and StatusWorkflowConfig. Handles CreateBug, UpdateBug, TransitionBugStatus, MarkBugDuplicate, AddBugDependency. Projects read models for product/user data. Emits all bug-related domain events. |
| `service-product` | Participant | Owns the product/component/version/milestone hierarchy and `group_control_map`. Projects `ProductGroupControlsReadModel` consumed by `service-bug` for entry/edit/confirm permission checks. Emits `GroupControlsUpdated` events. |
| `service-user` | Participant | Owns user identity, group membership, and authentication. `AuthenticatedUser` context flows through all command handlers. Group membership projected into local read models by consuming services for authorization checks. |
| `service-comment` | Participant | Owns the append-only CommentAggregate. Creates user comments and system comments (duplicate markers, attachment events). Emits `CommentCreated` consumed by `service-bug` (time-tracking) and `service-notification` (email). |
| `service-attachment` | Participant | Owns the AttachmentAggregate and all flag logic (bug-level + attachment-level, ADR-005). Binary data stored in external object storage (S3/MinIO). Emits `AttachmentCreated` consumed by `service-comment` (system comment) and `service-notification` (email). |
| `service-notification` | Participant | Terminal event consumer (ADR-006) — emits no events of its own. Subscribes to domain events from bug, comment, and attachment services. Computes recipients (roles + watchers + global watchers + preferences), renders per-user email templates (text + HTML), and dispatches via SMTP. |

---

## Trigger

The workflow begins when a **user submits the "File a Bug" form** through the React SPA. The API gateway receives the REST request, authenticates the user, checks the `bugs:create` permission (Layer 1), and routes the `CreateBug` command to `service-bug`.

Subsequent lifecycle events are triggered by user actions:
- **Adding a comment** → `CreateComment` command to `service-comment`
- **Uploading an attachment** → `CreateAttachment` command to `service-attachment`
- **Updating fields** → `UpdateBug` command to `service-bug`
- **Changing status** → `TransitionBugStatus` command to `service-bug`
- **Marking as duplicate** → `MarkBugDuplicate` command to `service-bug`
- **Adding a dependency** → `AddBugDependency` command to `service-bug`

Each action follows the same pattern: command → aggregate mutation → domain event emitted → subscription handlers react.

---

## Steps

### Step 1: Product Access Validation

**BPMN node**: `validate-product-access`

When the `CreateBug` command reaches `service-bug`, the `CanEnterProductPolicy` (Layer 2) verifies that the authenticated user has entry access to the specified product. This is resolved by querying the locally projected `ProductGroupControlsReadModel`, which is kept in sync by subscribing to `product.Events.GroupControlsUpdated`. If the product's `entry` group control requires membership in a specific group, the user's group membership (from `GroupMembershipReadModel`, projected from `user.Events.GroupMemberAdded/Removed`) is checked.

Simultaneously, the product's `allowsUnconfirmed` flag is resolved (from `ProductSummaryReadModel`), and the component's `initial_cc`, `defaultAssignee`, and `defaultQaContact` are loaded (from `ComponentSummaryReadModel`).

**Eventual consistency note**: These read models are projected from cross-service events. They may be slightly stale. The worst case is that a very recent group-control change has not yet been projected, allowing or denying entry incorrectly. This is acceptable because group-control changes are infrequent and the window is sub-second.

### Step 2: Initial Status Resolution

**BPMN node**: `validate-workflow-status`

The `CreateBug` handler queries the `StatusWorkflowReadModel` for creation-time statuses (transitions where `old_status IS NULL`). The product's `allowsUnconfirmed` flag determines the initial status:
- If `allowsUnconfirmed = true` → initial status is `UNCONFIRMED`
- If `allowsUnconfirmed = false` → initial status is `NEW`

This preserves Bugzilla's per-product behavior where some products auto-confirm bugs while others require manual confirmation.

### Step 3: Bug Aggregate Creation

**BPMN node**: `create-bug-aggregate`

The `BugAggregate` is created with all core fields, CC list (including component's `initial_cc`), keywords, aliases, custom fields, and group visibility restrictions. Fields are validated in topological dependency order per `VALIDATOR_DEPENDENCIES` (e.g., component depends on product, version depends on product, assigned_to depends on component).

The `MandatoryFieldPolicy` enforces that required custom fields have values. The `CanChangeFieldPolicy` checks field-level permissions based on the user's role (reporter, assignee, QA contact, editbugs group).

Upon successful creation, the aggregate emits `bug.Events.BugCreated` with the full bug snapshot (ADR-012: full diff payloads on events).

### Step 4: Post-Creation Fan-Out

**BPMN node**: `gateway-post-creation`

The `BugCreated` event is broadcast on RabbitMQ and consumed by four services in parallel:

1. **service-notification** (`BugCreatedNotificationHandler`): Computes recipients (reporter, assignee, QA contact, CC list, watchers, global watchers), filters by preferences and bug visibility (group restrictions), and sends "new bug" email to each qualifying recipient in their preferred format (text/HTML) and language.

2. **service-search**: Inserts the new bug into the fulltext search index (Elasticsearch per ADR-009).

3. **service-comment**: Prepares the comment thread scope for the new bug (no initial comment is created by the bug service; the first comment comes from the user's submission if they provided one, via a separate `CreateComment` command).

4. **service-attachment**: Initializes the attachment scope and resolves applicable flag types for the bug's product/component.

### Step 5: Comment Addition (Lifecycle)

**BPMN nodes**: `create-comment` → `notify-comment-created` + `project-comment-work-time`

When a user adds a comment, the `CreateComment` command is routed to `service-comment`. The handler validates:
- Bug existence and product-edit permission (`CanCommentOnBugPolicy`)
- Insider group membership if `isPrivate = true` (`IsInsiderPolicy`)
- Body length ≤ 65,535 characters after normalization

The `CommentAggregate` is created and emits `comment.Events.CommentCreated`, which is consumed by:
- **service-notification**: Sends "new comment" email to bug stakeholders. Private comments are excluded for non-insider recipients.
- **service-bug**: Projects `workTime` from the event payload into `BugTimeTrackingReadModel` (G4: time-tracking field ownership in service-bug). Also triggers fulltext search re-indexing.

### Step 6: Attachment Addition (Lifecycle)

**BPMN nodes**: `create-attachment` → `notify-attachment-created` + `create-attachment-system-comment`

When a user uploads a file, the `CreateAttachment` command is routed to `service-attachment`. The handler:
- Validates bug visibility via synchronous `GetBug` query to `service-bug`
- Sanitizes filename, validates MIME type, checks file size limits
- Writes binary data to S3/MinIO (ADR-005: out-of-band storage)
- Creates the `AttachmentAggregate` with metadata and `storageKey`
- Optionally creates initial flag instances

Emits `attachment.Events.AttachmentCreated`, consumed by:
- **service-notification**: Sends "new attachment" email
- **service-comment**: `AttachmentCreatedSubscriptionHandler` creates a system comment (type `ATTACHMENT_CREATED`) on the bug

### Step 7: Field Update (Lifecycle)

**BPMN nodes**: `update-bug` → `notify-bug-updated`

The `UpdateBug` command applies batch field changes with validator dependency ordering. The handler:
- Enforces `CanEditBugPolicy` (editbugs group OR assignee/reporter/QA with field-specific restrictions)
- Enforces `CanChangeFieldPolicy` for fine-grained field-level permissions
- Uses optimistic concurrency (expected version check) to detect concurrent edits (G5)
- Emits `bug.Events.BugUpdated` with full `changes` map: `{ field: { old, new } }`
- Also emits specialized events: `BugAssigned`, `BugCcChanged`, `BugCustomFieldChanged`, etc.

**service-notification** consumes `BugUpdated` and sends changed-field notification emails with rendered diffs. Forced recipients (e.g., a user just removed from CC) receive one final notification about their removal.

### Step 8: Status Transition (Lifecycle)

**BPMN nodes**: `validate-transition` → `gateway-transition-type` → `check-open-blockers` (conditional) → `apply-status-transition` → fan-out

This is the most complex step in the lifecycle. When a user changes bug status:

1. **Validate transition**: `ValidStatusTransitionPolicy` queries `StatusWorkflowReadModel` to verify the `(currentStatus, newStatus)` edge is active. If `require_comment` is configured for this edge, the command must include a non-empty comment.

2. **Conditional blocker check**: If the resolution is `FIXED`, the `NoOpenBlockersPolicy` queries `BugDependencyReadModel` for open blockers. The read model has two rows per dependency (bidirectional) and tracks the blocking bug's current status. If any blocker is still open, the transition is rejected.

3. **Apply transition**: The aggregate updates `status` and `resolution`. If the transition results in a closed status, `BugResolved` is emitted. If reopening, `BugReopened` is emitted. The `everConfirmed` flag is set to `true` if the bug leaves `UNCONFIRMED`.

4. **Fan-out**:
   - **service-notification**: Sends status change email. For resolution events, also sends dependency-cascade (`dep_only`) notifications to stakeholders of dependent bugs — informing them that a blocker was resolved. The event payload includes `affectedDependentBugIds` and `dependentBugSnapshots` so the notification service has full context without querying back (ADR-012).
   - **service-search**: Updates search index with new status/resolution.

### Step 9: Duplicate Marking (Lifecycle)

**BPMN nodes**: `mark-duplicate` → `create-duplicate-system-comments` + `notify-duplicate`

The `MarkBugDuplicate` command:
- Validates the target bug is visible and the transition is legal
- Sets `duplicateOf`, transitions to `duplicateOrMoveBugStatus`, sets resolution `DUPLICATE`
- Auto-CCs the target bug's reporter
- Emits `bug.Events.BugMarkedDuplicate`

Consumed by:
- **service-comment**: Creates two system comments — `DUPE_OF` on the duplicate bug and `HAS_DUPE` on the original bug
- **service-notification**: Sends duplicate notification to stakeholders of both bugs

### Step 10: Dependency Addition (Lifecycle)

**BPMN nodes**: `add-dependency` → `notify-dependency-added`

The `AddBugDependency` command:
- `DependencyLoopPolicy`: No self-reference, no circular dependencies (traverses full dependency tree from `BugDependencyReadModel`)
- `CanSeeBugPolicy`: Referenced bug must be visible to the current user
- Emits `bug.Events.BugDependencyAdded` with single-command approach (ADR-008): the reverse "blocks" relationship is projected into the read model, not enforced via cross-aggregate transaction

Consumed by:
- **service-notification**: Notifies stakeholders of the dependent bug that a new blocker was added

---

## Event Flow

### Primary Events (published by `service-bug`)

| Event | Payload Highlights | Consumers |
|-------|-------------------|-----------|
| `bug.Events.BugCreated` | `bugId`, `summary`, `status`, `productId`, `reporterId`, `assignedTo`, `ccList`, `groupIds` | service-notification, service-search, service-comment, service-attachment |
| `bug.Events.BugUpdated` | `bugId`, `changes` (map), `changedBy`, `forcedRecipients` | service-notification, service-search |
| `bug.Events.BugStatusTransitioned` | `bugId`, `oldStatus`, `newStatus`, `resolution`, `changedBy` | service-notification, service-search |
| `bug.Events.BugResolved` | `bugId`, `resolution`, `affectedDependentBugIds`, `dependentBugSnapshots` | service-notification |
| `bug.Events.BugReopened` | `bugId`, `oldStatus`, `newStatus` | service-notification |
| `bug.Events.BugMarkedDuplicate` | `bugId`, `duplicateOfBugId`, `changedBy` | service-notification, service-comment |
| `bug.Events.BugDependencyAdded` | `bugId`, `dependsOnBugId`, `changedBy` | service-notification |
| `bug.Events.BugAssigned` | `bugId`, `oldAssigneeId`, `newAssigneeId` | service-notification |
| `bug.Events.BugCcChanged` | `bugId`, `added`, `removed` | service-notification |

### Secondary Events (published by other services)

| Event | Publisher | Consumers |
|-------|-----------|-----------|
| `comment.Events.CommentCreated` | service-comment | service-notification, service-bug (time-tracking), service-search |
| `attachment.Events.AttachmentCreated` | service-attachment | service-notification, service-comment (system comment) |
| `product.Events.GroupControlsUpdated` | service-product | service-bug (authorization read model) |
| `product.Events.ProductCreated` | service-product | service-bug (product read model) |
| `user.Events.GroupMemberAdded` | service-user | service-bug (group membership read model) |

### Event Flow Diagram

```
                         ┌─────────────┐
          CreateBug ────▶│ service-bug │───▶ BugCreated
                         └──────┬──────┘          │
                               │                  ├─────────────▶ service-notification (email)
                               │                  ├─────────────▶ service-search (index)
                               │                  ├─────────────▶ service-comment (scope)
                               │                  └─────────────▶ service-attachment (scope)
                               │
          TransitionBugStatus   │
          ───────────────────▶  │───▶ BugStatusTransitioned
                               │          │
                               │          ├─────▶ service-notification (status email)
                               │          │         └── dep_only cascade on resolve
                               │          └─────▶ service-search (reindex)
                               │
          CreateComment ───────┼───▶ (routed to service-comment)
                               │          │
                               │     CommentCreated
                               │          │
                               │          ├─────▶ service-notification (comment email)
                               │          └─────▶ service-bug (time-tracking projection)
                               │
          CreateAttachment ────┼───▶ (routed to service-attachment)
                                       │
                                  AttachmentCreated
                                       │
                                       ├─────▶ service-notification (attachment email)
                                       └─────▶ service-comment (system comment)
```

---

## Error Handling & Compensation

### Creation Failures

| Failure Point | Error | Recovery |
|---------------|-------|----------|
| `CanEnterProductPolicy` denies access | Command rejected. User sees "You are not allowed to file bugs in this product." | User selects a different product or contacts admin for group membership. |
| `MandatoryFieldPolicy` fails | Command rejected with list of missing mandatory fields. | User fills required fields and resubmits. |
| `ValidStatusTransitionPolicy` fails (invalid initial status) | Command rejected. This indicates a configuration error in `StatusWorkflowConfig`. | Admin must configure creation-time statuses. |
| Optimistic concurrency conflict on creation | Extremely rare (aggregate is new). If it occurs, the command is retried with a new aggregate ID. | Automatic retry by the API gateway. |

### Status Transition Failures

| Failure Point | Error | Recovery |
|---------------|-------|----------|
| `ValidStatusTransitionPolicy` — illegal transition | Command rejected with "You are not allowed to change bug status from X to Y." | User selects a valid target status. The UI queries `GetValidStatusTransitions` to only show legal options. |
| `require_comment` not satisfied | Command rejected with "A comment is required for this status change." | User adds a comment and resubmits. |
| `NoOpenBlockersPolicy` — open dependencies | Command rejected with "You cannot resolve this bug as FIXED because it has open dependencies." | User resolves blocking bugs first, or selects a different resolution. |
| `CanConfirmBugPolicy` — user lacks confirmation rights | Command rejected with "You are not allowed to confirm this bug." | User with `canconfirm` rights performs the transition. |

### Comment and Attachment Failures

| Failure Point | Error | Recovery |
|---------------|-------|----------|
| Bug not found (comment) | Command rejected. | User refreshes — bug may have been deleted. |
| File too large (attachment) | Command rejected with file size limit error. | User reduces file size or admin increases limit. |
| Binary storage write fails (attachment) | Aggregate creation is aborted. Orphaned binary cleaned up by GC. | User retries. If persistent, infrastructure issue with S3/MinIO. |
| MIME type validation fails | Command rejected. | User provides a file with a legal content type. |

### Compensation Strategy

**No compensating actions are needed for most failures** because:
- Commands are rejected *before* the aggregate is mutated — no event is emitted, no side effects occur.
- Event-sourced aggregates are atomic: either the full command succeeds and events are emitted, or nothing happens.

**For successful operations that need "undo"**:
- **Status transition**: A reopen (closed → open) is a separate `TransitionBugStatus` command, not a compensation. It clears `resolution` and `duplicateOf`.
- **Dependency removal**: `RemoveBugDependency` is a separate command that emits `BugDependencyRemoved`.
- **CC removal**: `RemoveBugCc` is a separate command. The removed user receives one forced notification about their removal, then no further notifications.
- **Duplicate unmarking**: Reopening the bug (closed → open) clears `duplicateOf` and `resolution`.

**Eventual consistency tolerance**:
- Notification emails are best-effort. If `service-notification` is temporarily down, events are queued in RabbitMQ and processed when the service recovers. Duplicate emails may be sent on replay — acceptable for a notification system.
- System comment creation (attachment/duplicate events) has a brief eventual-consistency delay. The system comment may appear a few seconds after the triggering event. This is acceptable.
- The `noresolveonopenblockers` check queries a read model that may be slightly stale. In the worst case, a very recently resolved blocker may not yet be reflected, allowing a resolve that should have been blocked. This is the same consistency model Bugzilla uses.

---

## Data Consistency Model

### Eventual Consistency Boundaries

This workflow involves **five distinct eventual-consistency boundaries**:

1. **Bug aggregate → Read models**: After `CreateBug` or `TransitionBugStatus`, the `BugDetailReadModel` and `BugListReadModel` are updated asynchronously by the event-sourcing platform's projection engine. Queries immediately after a command may return stale data. The UI should use optimistic updates or `waitForProjection` semantics when it needs to read back immediately.

2. **Cross-service event propagation**: Events published by `service-bug` are delivered to downstream services (`service-notification`, `service-comment`, `service-search`) via RabbitMQ. Delivery is guaranteed (at-least-once) but not instantaneous. Typical latency is sub-second; backpressure or service outages may increase this to minutes.

3. **Cross-service read model projections**: `service-bug` maintains local read models projected from `service-product` and `service-user` events (e.g., `ProductGroupControlsReadModel`, `GroupMembershipReadModel`). These are used for authorization checks during command handling. The data is eventually consistent — a very recent group change may not be reflected. This is acceptable because group changes are infrequent and the authorization decision is based on the best-available data at command time.

4. **Dependency read model**: The `BugDependencyReadModel` is projected from `BugDependencyAdded/Removed` events and also subscribes to `BugStatusTransitioned` to track blocker open/closed state. The `NoOpenBlockersPolicy` queries this read model, accepting that it may be slightly stale (Q9).

5. **Notification delivery**: Email dispatch by `service-notification` is asynchronous and best-effort. SMTP delivery failures are logged but do not trigger compensating actions. The `EmailDeliveryLogReadModel` tracks delivery status for diagnostics.

### Consistency Guarantees

| Operation | Consistency | Mechanism |
|-----------|-------------|-----------|
| Bug field mutation | **Strong** (per aggregate) | Event sourcing ensures atomic aggregate state transitions. Optimistic concurrency prevents lost updates. |
| Status transition validation | **Eventual** (workflow config) | `StatusWorkflowReadModel` is projected from `StatusWorkflowConfig` aggregate events. Admin workflow changes propagate within seconds. |
| Authorization check (group controls) | **Eventual** (projected read model) | `ProductGroupControlsReadModel` is projected from `product.Events.GroupControlsUpdated`. No synchronous call to `service-product`. |
| `noresolveonopenblockers` check | **Eventual** (dependency read model) | `BugDependencyReadModel` may be slightly stale. Worst case: allowing a resolve that should have been blocked. Acceptable for a bug tracker (Q9). |
| Email notification delivery | **Best-effort** (async SMTP) | Events are queued in RabbitMQ. At-least-once delivery. Duplicate emails possible on replay. |
| System comment creation | **Eventual** (event subscription) | System comments (attachment events, duplicate markers) appear after the triggering event propagates. Brief delay is acceptable. |
| Binary attachment storage | **Strong** (synchronous write) | Binary is written to S3/MinIO before aggregate save. If aggregate save fails, orphaned binary is cleaned up by GC. |

### Cross-Aggregate Invariants

The workflow has two cross-aggregate invariants that require special design:

1. **`noresolveonopenblockers`**: Enforced by querying `BugDependencyReadModel` (which tracks blocker statuses) rather than loading all dependent aggregates. This accepts eventual consistency — the read model may not yet reflect a very recent blocker resolution.

2. **No circular dependencies**: `DependencyLoopPolicy` traverses the full dependency tree from `BugDependencyReadModel` in both directions. If the union of the depends-on tree and blocked-by tree has any overlap, the operation is rejected. This check reads from a read model, not the aggregates themselves, accepting that very recent dependency changes may not yet be projected.

### `waitForProjection` Usage

The following operations should use `waitForProjection` semantics in the UI or integration tests:

- After `CreateBug`: Wait for `BugDetailReadModel` before redirecting to the bug detail page
- After `TransitionBugStatus`: Wait for `BugListReadModel` update before refreshing bug list views
- After `AddBugDependency`: Wait for `BugDependencyReadModel` before re-rendering dependency graphs

The BPMN's parallel fan-out after creation and status transitions implies that the UI should not assume read-model consistency immediately after the command succeeds.

## Decision Rules Referenced

| Gateway ID | Gateway label | Rule ID | Rule citation |
|------------|---------------|---------|---------------|
| gateway-post-creation | Post-creation fan-out | [unciteable: TODO] | [unciteable: TODO] |
| gateway-post-transition | Post-transition fan-out | BR-workflow-bug-lifecycle-GW-is-closed-gateway | output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md#state-machines |
| gateway-post-transition | Post-transition fan-out | BR-bug-004 | output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md#business-rule-4 |
| gateway-post-transition | Post-transition fan-out | BR-bug-012 | output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md#business-rule-12 |
| gateway-transition-type | Transition type? | BR-workflow-bug-lifecycle-GW-is-resolving-gateway | output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md#state-machines |
| gateway-transition-type | Transition type? | BR-bug-008 | output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md#business-rule-8 |
| gateway-transition-type | Transition type? | BR-bug-POLICY-NoOpenBlockersPolicy | output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md#authorization-layer-2 |
