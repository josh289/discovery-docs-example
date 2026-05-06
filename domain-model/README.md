# Domain Model — Audit Output

This directory contains the domain-class audit artifacts for the Bugzilla Evergreen decomposition into seven bounded contexts.

## Per-Service Class Diagrams (Slicer-Owned)

These files were produced by individual domain-model slicers. Each contains a Mermaid class diagram of the aggregates, child entities, and value objects within a single bounded context, plus a cross-context foreign-key references section.

- [class-attachment.md](class-attachment.md) — service-attachment: AttachmentAggregate, FlagTypeAggregate
- [class-bug.md](class-bug.md) — service-bug: BugAggregate, StatusWorkflowConfig
- [class-comment.md](class-comment.md) — service-comment: CommentAggregate
- [class-notification.md](class-notification.md) — service-notification: ScheduledReportAggregate
- [class-product.md](class-product.md) — service-product: ProductAggregate, ComponentAggregate, VersionAggregate, MilestoneAggregate
- [class-search.md](class-search.md) — service-search: SavedSearchAggregate
- [class-user.md](class-user.md) — service-user: UserAggregate, GroupAggregate

## Synthesised Integration Documents

These files were produced by the audit-domain-synthesizer agent by reading all slicer outputs and the underlying architecture/spec docs.

- [cross-context.md](cross-context.md) — Mermaid classDiagram of aggregate-root-to-aggregate-root foreign-key edges crossing bounded contexts, with a full edge inventory table and boundary-dispute notes.
- [invariants.md](invariants.md) — Consolidated table of domain invariants across all seven services, harvested from architecture and specification documents (37 invariants total).
