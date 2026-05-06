# ADR-007: Event Payload Design — Full Diffs vs Query-Back

## Status

Accepted

## Context

Bugzilla's email notification system (BugMail) is a cross-cutting workflow that reacts to bug state changes and delivers templated, role-filtered email to users. In the current monolithic Perl architecture, BugMail reads the full bug object (`Bugzilla::Bug`), all comments, and activity diffs directly from the database to render notification emails. It computes what changed since `lastdiffed` by querying `bugs_activity` joined with `fielddefs`, fetches comments added in the diff window, and determines recipients based on roles (reporter, assignee, QA, CC), watchers, and global watchers.

In the Banyan CQRS/Event Sourcing migration, the `service-notification` service is a **pure consumer** — it subscribes to domain events from multiple services (`service-bug`, `service-comment`, `service-attachment`, `service-user`) and never publishes events of its own. It renders and delivers emails based on the data it receives through those events.

This creates a fundamental design question: **how much data should domain events carry in their payloads?**

### The problem

There are two extremes:

1. **Fat events (full diffs)**: Each domain event includes the changed fields, old/new values, and sufficient context (bug summary, product name, reporter identity) for consumers to act without further queries.
2. **Thin events (references only)**: Each domain event carries only the aggregate ID and a change type. Consumers must query back to the source service for details.

In Bugzilla, the notification pipeline needs:
- Which fields changed and their old/new values (for diff display in emails)
- New comments added (for comment notification bodies)
- Bug context: summary, product name, component, reporter, assignee, QA contact, CC list
- Recipient role information (who is reporter, assignee, QA, CC, watcher)
- Visibility metadata (is the change on a time-tracking field? is a comment private?)

Without diff payloads, `service-notification` must make **N synchronous queries** back to `service-bug`, `service-comment`, and `service-attachment` for **every single event** it receives — at minimum: one query for the bug detail, one for the activity diff, one for new comments, and potentially more for attachment metadata or user preferences.

### Why now

This decision affects every domain event class in every service. It is foundational to the contract design (`@banyanai/service-*-contracts` packages) and cannot be easily reversed after events are in production. The event payload schema is the primary integration contract between services, and getting it wrong requires rework across all service boundaries.

## Decision

**Domain events carry full diff payloads including changed fields, old/new values, and sufficient context for downstream consumers (especially `service-notification`) to render complete notifications without synchronous query-back to source services.**

Specifically, every mutable-state domain event (e.g., `BugUpdated`, `BugStatusChanged`, `CommentAdded`, `AttachmentAdded`) includes:

1. **Change envelope**: the field names that changed, with old and new values for each.
2. **Aggregate snapshot context**: enough denormalized context from the aggregate for rendering — e.g., for a `BugUpdated` event: `bugId`, `summary`, `productName`, `componentName`, `reporterId`, `assigneeId`, `qaContactId`, `ccList`.
3. **Event-specific payload**: the particular data relevant to the event type — e.g., for `CommentAdded`: the new comment's `commentId`, `body`, `authorId`, `isPrivate`, `creationTime`; for `AttachmentAdded`: `attachmentId`, `fileName`, `description`, `contentType`.
4. **Metadata**: `aggregateId`, `aggregateType`, `eventId`, `eventTimestamp`, `userId` (who triggered the change).

### What events do NOT include

- The full text of **all** comments on the bug — only the **new or changed** comment relevant to the event.
- Full attachment binary data (attachments are stored in object storage; events carry metadata only).
- Complete user objects — events carry `userId` references; the notification service resolves user details (email, display name, preferences) from a cached read model projected from `service-user` events.
- Historical activity — the event represents a single change, not a queryable history. Historical queries use the `BugActivityReadModel` projected from the event stream.

### Example payload shape

```typescript
// BugUpdated event payload (conceptual)
{
  eventId: "evt-uuid-123",
  eventType: "BugUpdated",
  aggregateId: "bug-456",
  aggregateType: "BugAggregate",
  timestamp: "2026-05-06T10:30:00Z",
  userId: "user-789",

  // Change envelope
  changes: [
    { field: "status", oldValue: "NEW", newValue: "ASSIGNED" },
    { field: "assigneeId", oldValue: null, newValue: "user-789" }
  ],

  // Aggregate snapshot context
  context: {
    bugId: "bug-456",
    summary: "Login page throws 500 error",
    productName: "WebApp",
    componentName: "Auth",
    reporterId: "user-100",
    assigneeId: "user-789",
    qaContactId: null,
    ccList: ["user-200", "user-300"]
  }
}
```

## Alternatives Considered

### Alternative 1: Thin events with synchronous query-back

Events carry only aggregate ID and event type. `service-notification` queries `service-bug` via query handlers to fetch bug details, activity diffs, and comments for every notification.

**Rejected because:**
- Introduces **tight runtime coupling** between `service-notification` and every source service. If `service-bug` is down or slow, notifications fail or degrade.
- Creates **N+1 query patterns**: one event → multiple synchronous HTTP/RPC calls per recipient.
- **Time-consistency risk**: by the time the notification service queries the bug detail, the bug may have changed again (e.g., another update happened between the event and the query). The notification would render stale or incorrect data.
- Defeats the purpose of event-driven architecture: events should be self-contained facts.
- Increases latency for notification delivery due to synchronous query chains.

### Alternative 2: Hybrid — thin events with materialized view

Events carry minimal data, but `service-notification` maintains a materialized read model of all bug state by subscribing to all bug events and projecting a local copy.

**Rejected because:**
- `service-notification` effectively becomes a read-model replica of `service-bug`, `service-comment`, and `service-attachment` — massive data duplication for a service whose primary job is sending emails.
- Still requires eventual consistency for the materialized view; a fast succession of events may produce stale renders.
- Adds operational complexity: the notification service must manage and migrate its own read-model database schema for entities it doesn't own.
- The materialized view approach is better suited for a search index or dashboard, not a notification pipeline.

### Alternative 3: Snapshot events (full aggregate state on every change)

Every event includes the complete current state of the aggregate.

**Rejected because:**
- Excessive payload sizes — a bug with 50 custom fields, a long CC list, and extensive metadata would produce events measured in tens or hundreds of KB on every minor field change.
- Most consumers don't need the full state — they need the diff and enough context for their specific purpose.
- Encourages lazy event design where consumers cherry-pick fields, creating hidden coupling.

## Consequences

### What becomes easier

- **Notification rendering**: `service-notification` can render complete emails from a single event payload without any synchronous calls to source services. This eliminates an entire class of failure modes (source service unavailable, network timeouts, cascading retries).
- **Time-consistent snapshots**: The event payload captures the state **at the time of the change**, not at query time. Email recipients see exactly what changed, not what the bug looks like hours later when the notification is processed.
- **Event replay and debugging**: Fat events are self-documenting. Replaying events for debugging, audit, or re-processing notifications requires no additional infrastructure — the event contains everything needed.
- **Decoupled deployment**: Source services can be taken offline for maintenance without affecting in-flight notification processing. Events already in the queue carry all necessary data.
- **Simplified subscription handlers**: Each `@EventHandlerDecorator` handler in `service-notification` works with a single, self-contained input object. No need to inject query clients or manage circuit breakers for cross-service calls.
- **Testing**: Event handlers can be unit-tested with simple mock event payloads, no need for stubbed query handlers or service mocks.

### What becomes harder

- **Larger messages on the bus**: Event payloads are larger than thin reference events. For a bug tracker, this is measured in single-digit KB per event — well within RabbitMQ's comfortable range (messages up to hundreds of MB). The throughput impact is negligible for the expected event volume.
- **Event contract versioning**: Richer payloads mean more fields that can change over time. Consumers must tolerate unknown fields (forward compatibility) and producers must not remove existing fields (backward compatibility). This is manageable with a `changes` map structure and optional context fields, but requires discipline.
- **Aggregate must populate context at emission time**: Command handlers must gather the necessary context (product name, component name, user references) before emitting the event. This is a natural extension of the command handler's existing responsibilities — it already validates these relationships during command processing.
- **Potential for context drift**: If the aggregate's context fields (e.g., product name) change between event emission and consumption, the event carries the name at emission time, not the current name. This is **correct behavior** for notifications (the email should reflect what was true at the time of the change), but consumers that need current state must query separately.

### Accepted trade-offs

- **Message bus storage**: Larger events consume more space in the event store and message bus. For a bug tracker with moderate event volume (orders of magnitude less than a financial trading system), this is an acceptable cost. Event retention policies can prune old events after read models are caught up.
- **Context denormalization**: Some data (product name, component name) is denormalized into event payloads even though it could be resolved by querying the owning service. This is a deliberate trade-off: we accept duplicated reference data in events to eliminate runtime coupling.
- **Not a universal rule**: This ADR establishes the default pattern (fat payloads), but individual events may choose to be thinner when the context is clearly unnecessary for all known consumers. The decision is made per event type, with the bias toward completeness.

## Payload Schema Guidelines

The following guidelines govern the design of event payloads across all services:

### 1. Every mutable-state event includes a `changes` array

```typescript
changes: Array<{
  field: string;       // Field name matching the contract property name
  oldValue: unknown;   // Previous value (null if previously unset)
  newValue: unknown;   // New value
}>
```

- For creation events (e.g., `BugCreated`), `oldValue` is always `null`.
- For removal events (e.g., `CcRemoved`), `newValue` is `null`.
- For list/set fields (CC, keywords, dependencies), the `changes` entry reflects the delta (added/removed items), not the full list.

### 2. Every event includes an `eventContext` object

```typescript
eventContext: {
  aggregateId: string;
  aggregateType: string;
  eventId: string;
  eventTimestamp: string;   // ISO 8601
  userId: string | null;    // Who triggered the change
  [key: string]: unknown;   // Service-specific context fields
}
```

### 3. Bug-related events include a `bugContext` snapshot

All events from `service-bug` that are consumed by `service-notification` include:

```typescript
bugContext: {
  bugId: string;
  summary: string;
  productName: string;
  componentName: string;
  reporterId: string;
  assigneeId: string | null;
  qaContactId: string | null;
  ccList: string[];
  status: string;
  resolution: string | null;
}
```

This context is populated from the aggregate state at the time of event emission.

### 4. Comment-related events include the new/changed comment only

```typescript
comment: {
  commentId: string;
  bugId: string;
  authorId: string;
  body: string;
  isPrivate: boolean;
  creationTime: string;
  tags?: string[];
  workTime?: number;    // If time-tracking is applicable
}
```

Events do **not** include all comments on the bug — only the comment relevant to this event.

### 5. Attachment-related events include metadata only

```typescript
attachment: {
  attachmentId: string;
  bugId: string;
  fileName: string;
  description: string;
  contentType: string;
  isPatch: boolean;
  isObsolete: boolean;
  isPrivate: boolean;
  creatorId: string;
  // Binary data is NOT included — stored in object storage
}
```

### 6. User reference data is carried as IDs, not full objects

Events carry `userId` strings. The notification service resolves user details (email, display name, notification preferences) from its own cached read model projected from `service-user` events. This prevents user data duplication in every event while keeping the notification service self-sufficient for rendering.

### 7. Custom field changes use the generic event pattern

Custom field changes (`cf_*` fields) follow the generic `BugCustomFieldChanged` event pattern with `fieldName` and `value` in the `changes` array. No event-class-per-custom-field.

### 8. Events are immutable contracts

Once an event type is published with a specific payload shape, fields must not be removed or renamed. New fields may be added as optional. Consumers must ignore unknown fields. Breaking changes require a new event type (e.g., `BugUpdatedV2`).
