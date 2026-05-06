# ADR-002: Event Sourcing Replaces `bugs_activity`

## Status

Accepted

## Context

Bugzilla logs every field-level mutation on a bug to the `bugs_activity` table. Each row records:

| Column | Content |
|--------|---------|
| `bug_id` | The bug that changed |
| `who` | User ID of the changer |
| `bug_when` | Timestamp of the change |
| `fieldid` | Foreign key into `fielddefs` (which field changed) |
| `removed` | Old value (string, truncated at 254 chars with multi-row splitting) |
| `added` | New value (string, same truncation) |
| `comment_id` | Optional link to a comment that triggered the change |
| `attach_id` | Optional link to an attachment that triggered the change |

Activity is written by the `LogActivityEntry` function, called from `Bugzilla::Bug::update()` for **every** mutable field: standard fields (status, resolution, priority, severity, assignee, QA contact, product, component, version, milestone, whiteboard, deadline, URL, time-tracking fields), CC list changes, alias changes, keyword changes, dependency changes, group-visibility changes, comment-privacy changes, work-time changes, see-also changes, flag changes, and duplicate-of changes.

The UI consumes activity via `Bugzilla::Bug::get_activity()`, which groups rows into "operations" (same `who` + `bug_when`) containing multiple field changes, with access-control filtering (time-tracking fields hidden from non-timetrackers, private-comment activity hidden from non-insiders).

### Why this matters for the migration

The `bugs_activity` table is Bugzilla's **manual event log** — a bespoke, append-only audit trail recording *who changed what, when, from what, to what*. It is structurally isomorphic to a domain event stream: each row is a field-level fact that occurred in the past, ordered by time, and never mutated after insertion.

In the Banyan CQRS/Event Sourcing platform, the event stream **is the source of truth**. Every command that mutates the `BugAggregate` emits one or more domain events (e.g., `BugStatusChanged`, `BugAssigned`, `BugPriorityChanged`, `CcAdded`). These events are persisted by the platform's event store, fully ordered per aggregate, and immutable by definition.

This creates a direct overlap: both `bugs_activity` and the event stream exist to record *the same information* — every mutation to a bug. Migrating `bugs_activity` as a separate table would introduce a redundant, potentially inconsistent duplicate of data that the platform already captures as a first-class citizen.

## Decision

**The `bugs_activity` table will not be migrated.** Domain events in the Banyan event store replace it entirely.

Specifically:

1. **No `bugs_activity` table or equivalent** will exist in any Banyan service. The event stream is the authoritative audit trail.

2. **A `BugActivityReadModel`** will be projected from the bug aggregate's domain events to serve the activity-log UI. This read model maps events into the operation-based display format that the Bugzilla UI expects (grouped by timestamp + user, with field-level old/new value pairs).

3. **Historical `bugs_activity` data from the legacy system** will be handled by a one-time data-migration process that replays legacy activity rows as synthetic historical events (or snapshots), rather than maintaining a parallel table.

### Event-to-activity mapping

The following event types carry the information previously stored in `bugs_activity` rows:

| Legacy `fieldid` | Banyan Domain Event | Old/New Value Source |
|-------------------|---------------------|----------------------|
| `bug_status` | `BugStatusChanged` | `fromStatus` / `toStatus` |
| `resolution` | `BugStatusChanged` | `fromResolution` / `toResolution` (nested in same event) |
| `assigned_to` | `BugAssigned` | `fromAssignee` / `toAssignee` |
| `priority` | `BugPriorityChanged` | `from` / `to` |
| `bug_severity` | `BugSeverityChanged` | `from` / `to` |
| `short_desc` | `BugSummaryChanged` | `from` / `to` |
| `product` | `BugProductChanged` | `from` / `to` |
| `component` | `BugComponentChanged` | `from` / `to` |
| `version` | `BugVersionChanged` | `from` / `to` |
| `target_milestone` | `BugMilestoneChanged` | `from` / `to` |
| `cc` | `CcAdded` / `CcRemoved` | `added` / `removed` (collection diffs) |
| `dependson` / `blocked` | `BugDependencyAdded` / `BugDependencyRemoved` | `dependencyId` + direction |
| `keywords` | `BugKeywordAdded` / `BugKeywordRemoved` | `keyword` |
| `bug_group_map` | `BugGroupAdded` / `BugGroupRemoved` | `groupId` |
| `status_whiteboard` | `BugWhiteboardChanged` | `from` / `to` |
| `deadline` | `BugDeadlineChanged` | `from` / `to` |
| `estimated_time` | `BugTimeEstimateChanged` | `from` / `to` |
| `remaining_time` | `BugTimeRemainingChanged` | `from` / `to` |
| `dup_id` | `BugMarkedDuplicate` | `duplicateOfId` |
| Custom fields (`cf_*`) | `BugCustomFieldChanged` | `fieldName` + `from` / `to` |

### BugActivityReadModel design

The read model will use the Banyan `@ReadModel` decorator pattern:

```
@ReadModel({ tableName: 'rm_bug_activity', aggregateType: 'BugAggregate' })
```

It projects from all bug-related events, extracting per-field change tuples. The UI queries this read model (via a `GetBugActivityQuery`) with filtering for:
- **Access control**: Time-tracking fields excluded for non-`bugs:edit_timetracking` users; private-comment-linked activity excluded for non-insiders.
- **Grouping**: Events within the same command execution (same timestamp + user) are grouped into "operations" matching the legacy UI format.
- **Pagination**: By timestamp range or offset, matching the existing activity-log display pattern.

### Cross-service activity

Activity for entities owned by other services (comments, attachments, flags) is recorded by those services' own event streams. The bug activity read model projects only bug-aggregate events. Where cross-referencing is needed (e.g., "attachment added to bug"), the `AttachmentCreated` event in `service-attachment` includes `bugId`, and the bug's notification/activity display aggregates information from multiple read models at the query layer.

## Alternatives Considered

### Alternative 1: Migrate `bugs_activity` as a separate denormalized table

Keep a dedicated `bugs_activity`-equivalent table in `service-bug`, populated by event subscription handlers.

**Rejected because:**
- Introduces a redundant data store that must be kept consistent with the event stream.
- Any bug in the subscription handler creates a silent divergence between the event stream (truth) and the activity table (display).
- The Banyan platform's read-model infrastructure already solves the "project events into a queryable table" problem — a hand-rolled table is unnecessary duplication of platform capability.
- Adds operational burden: schema migrations, index tuning, and data cleanup for two copies of the same data.

### Alternative 2: Query the event stream directly (no read model)

Serve the activity UI by querying the event store directly, without an intermediate read model.

**Rejected because:**
- The event store is optimized for appending and replay, not for ad-hoc querying by field name, user, or time range.
- The legacy UI expects grouped "operations" with access-control filtering — this requires projection logic that doesn't belong in a raw event-store query.
- Would create tight coupling between the UI layer and the event store's internal schema, making event schema evolution harder.
- The Banyan platform's read-model pattern exists precisely to bridge this gap between event-stream shape and query shape.

### Alternative 3: Fine-grained per-field events (one event per `bugs_activity` row)

Emit a separate domain event for each field change (e.g., `BugStatusChanged`, `BugPriorityChanged`, `BugSummaryChanged`) rather than a single `BugUpdated` event with a changes map.

**Partially adopted.** The chosen approach uses **semantically named events per change type** (status changes, assignments, priority changes, etc.) because:
- Named events enable type-safe subscribers (e.g., `service-notification` subscribing to `BugStatusChanged` without filtering a generic payload).
- Named events map cleanly to Banyan's cross-service subscription pattern (`bug.Events.BugStatusChanged`).
- The `BugActivityReadModel` projects from all of these events uniformly — the naming is for consumers, not for the read model.

However, we stop short of a fully generic `BugFieldChanged(fieldName, oldValue, newValue)` event because that loses the type safety and domain expressiveness that Banyan's subscription model provides. The exception is custom fields, where a generic `BugCustomFieldChanged` is necessary because field names are dynamic.

## Consequences

### What becomes easier

- **No dual-write consistency risk.** The event stream is the single source of truth for all mutations. There is no `bugs_activity` table to drift out of sync.
- **Simpler command handlers.** Handlers emit events; they don't also call `LogActivityEntry`. The platform handles persistence.
- **Inherent immutability.** Event sourcing guarantees that the audit trail is append-only and tamper-evident — no risk of `UPDATE bugs_activity` or `DELETE FROM bugs_activity` corrupting history.
- **Event replay for recovery.** If the `BugActivityReadModel` is corrupted or the schema changes, it can be rebuilt by replaying the event stream — no need for a separate repair process.
- **Cross-service audit.** Events are the integration mechanism; `service-notification`, `service-search`, and `service-attachment` consume the same events that form the audit trail. One mechanism serves two purposes.
- **Schema evolution.** Adding new fields or change types means adding new event types or extending existing ones — no `fielddefs` lookup, no `MAX_LINE_LENGTH` splitting logic.

### What becomes harder

- **Historical data migration.** Legacy `bugs_activity` rows must be converted into synthetic domain events (or a pre-populated read model snapshot) during the one-time migration. This requires careful mapping of `fieldid` references to event types and handling of the legacy value-splitting logic (values exceeding 254 chars were split across multiple rows with comma-delimited boundaries).
- **Read model must handle event replay.** The `BugActivityReadModel` must be idempotent under replay. If the read model is rebuilt from the event stream, it must produce identical results. This means projection logic must be deterministic and side-effect-free.
- **Access-control filtering shifts to the query layer.** In Bugzilla, the `get_activity()` method filters rows at read time. In Banyan, the read model may contain all activity, with filtering in the query handler. This is architecturally cleaner but requires the query handler to know about time-tracking and insider-group permissions (Layer 2 policy checks on reads).
- **Activity granularity mismatch.** Bugzilla's `update()` method is a batch operation — one call changes multiple fields, producing multiple `bugs_activity` rows with the same timestamp. Banyan events are emitted per-change-type within a single command execution. Grouping these back into "operations" for UI display requires the read model or query handler to correlate events by aggregate version range or command execution ID (a pattern the platform may or may not natively support).
- **Long-value handling.** Bugzilla's `LogActivityEntry` splits values exceeding 254 characters across multiple rows. The event stream has no such limitation, but the read model must handle projecting potentially large field values (e.g., multi-select custom fields, long whiteboard strings) into a display-friendly format.

### Accepted trade-offs

- **Eventual consistency of the activity display.** The `BugActivityReadModel` is projected asynchronously. Activity may not appear instantly after a command completes. This is acceptable because Bugzilla's activity display was never real-time (it was written in the same DB transaction but read on the next page load).
- **Read model rebuild cost.** For bugs with extensive history (thousands of events), rebuilding the read model from the full event stream may be slow. Mitigation: snapshot-based rebuild optimization in the platform, or pre-computed migration snapshots for high-volume bugs.
- **No direct SQL querying of activity.** Users and reporting tools that historically queried `bugs_activity` directly must use the `GetBugActivityQuery` API instead. This is a deliberate encapsulation benefit, but it requires any custom reporting to go through the query layer.
