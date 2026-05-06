# Appendix B — Glossary

Plain-English definitions for non-engineer readers. Domain terms are sourced from `output/phase-1-discovery/discovery.md` and the SERVICE_SPECs at `output/phase-5-specification/specs/`.

### Aggregate

A cluster of related domain objects treated as a single unit for data changes. Every change to an aggregate is atomic — either the whole change succeeds or none of it does. Each aggregate has one root entity that external code interacts with. [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md]

### Bounded Context

A logical boundary inside a domain where a particular model applies consistently. Two bounded contexts may use the same term ("user", "report") to mean different things, because each context has its own model optimized for its own purpose. [source: output/phase-1-discovery/discovery.md]

### Bug

The central record in Bugzilla representing a defect, enhancement request, or task. Owns its own status workflow, assignee, reporter, CC list, dependencies, custom fields, group visibility restrictions, and comment thread. In the migration, bugs live in the `service-bug` bounded context as `BugAggregate` instances. [source: output/phase-1-discovery/discovery.md]

### Command

An instruction to change the state of the system — for example, "create a bug" or "assign this bug to a user." Commands are validated by the receiving service; if validation passes, the service applies the change and emits one or more events recording what happened. [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md]

### CQRS

Short for Command Query Responsibility Segregation — a pattern that separates operations that change data (commands) from operations that read data (queries). The write side (aggregates, event store) and the read side (read models, projections) are optimized independently. [source: output/phase-4-architecture/services/service-bug.md]

### Evergreen (framework)

Greenfield Labs's proprietary framework for building production services in **TypeScript** and **.NET**. Provides CQRS, Event Sourcing, read-model projections, message bus integration, and an opinionated service skeleton (decorator-driven aggregates, command/query handlers, event subscribers, read models). The recommended target architecture for this audit is built on the Evergreen platform; service contracts are published as `@evergreen/service-{name}-contracts` and runtime libraries as `@evergreen/platform-*`. Earlier audit drafts referred to this framework as "Banyan"; the canonical name is **Evergreen**.

### Event

An immutable record of something that happened in the system — for example, "BugCreated" or "CommentAdded." Events are the source of truth in an event-sourced system: the current state of any aggregate is computed by replaying its event history. [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md]

### Event Sourcing

A persistence pattern where instead of storing the current state of an entity, the system stores every state-changing event in the order it occurred. The current state is derived by replaying those events from the beginning. This provides a complete audit trail by design. [source: output/phase-4-architecture/services/service-bug.md]

### Product

A top-level container in the Bugzilla hierarchy (e.g., "Firefox", "Thunderbird"). Owns components, versions, and milestones. Controls bug-filing access through group permission bindings. [source: output/phase-1-discovery/discovery.md]

### Read Model

A pre-computed, optimized view of domain data built by listening to events. Read models power the query side of CQRS — they are tailored for specific screens or reports and can be rebuilt from the event store at any time. [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md]

### Saga / Process Manager

A long-running coordinator that manages a multi-step sequence across one or more services. Each step emits an event that triggers the next step. The Product Creation Saga, for example, creates the product row, auto-creates the first version, creates the default milestone, and optionally creates a bug group — all as a single logical operation. [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md]

### Service

An independently deployable unit of software that owns a bounded context. In this migration, each service (service-bug, service-user, service-product, service-attachment, service-comment, service-search, service-notification) has its own event store, read models, and API. Services communicate via domain events on a message bus. [source: output/phase-1-discovery/discovery.md]

### Wire Format

The encoding and protocol used to send data over the network between systems. Bugzilla's legacy wire formats include REST (JSON over HTTP), XMLRPC, and JSONRPC. In Evergreen, the wire format is JSON over HTTPS via an API gateway that routes to individual services. [source: audit-output/integration-surface.md]

### Component

A subdivision of a Product in the Bugzilla hierarchy — for example, "Layout" within the "Firefox" product. Each component has a default assignee, default QA contact, and a default CC list. Components cannot exist without a parent product. [source: output/phase-1-discovery/discovery.md]

### Group

A collection of users used for access control. Groups can control product visibility, bug visibility, edit permissions, and flag grant/request authority. Membership can be explicit (admin-assigned) or automatic (regex-based on email address). Groups support inheritance through a directed acyclic graph. [source: output/phase-1-discovery/discovery.md]
