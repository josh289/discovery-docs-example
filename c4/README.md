# C4 Audit — Diagram Index

## Level 1 — System Context
- [context.md](./context.md) — C4Context diagram: actors, the Evergreen Bug Tracker system, and external dependencies (SMTP, S3/MinIO, LDAP/RADIUS, legacy Bugzilla import)

## Level 2 — Containers
- [container.md](./container.md) — C4Container diagram: API gateway, seven microservices, PostgreSQL event/read-model stores, Redis, RabbitMQ, Elasticsearch, S3/MinIO, SMTP, Jaeger

## Level 3 — Components (per service)
- [component-bug.md](./component-bug.md) — service-bug: BugAggregate, StatusWorkflowConfig, status workflow policies, dependency checking, duplicate marking, product change propagation
- [component-user.md](./component-user.md) — service-user: UserAggregate, GroupAggregate, multi-method auth, group membership lifecycle, API keys, user settings
- [component-product.md](./component-product.md) — service-product: ProductAggregate, ComponentAggregate, VersionAggregate, MilestoneAggregate, group controls, ProductCreationSaga
- [component-comment.md](./component-comment.md) — service-comment: CommentAggregate, append-only lifecycle, privacy/insider-group toggling, user-applied tags, system comments
- [component-attachment.md](./component-attachment.md) — service-attachment: AttachmentAggregate, FlagTypeAggregate, flag request/grant/deny state machine, BugMovedFlagRetargetHandler
- [component-search.md](./component-search.md) — service-search: SavedSearchAggregate, Elasticsearch integration, boolean chart AST, quicksearch parser, security filter
- [component-notification.md](./component-notification.md) — service-notification: ScheduledReportAggregate, email rendering pipeline, recipient computation, 14 event subscription handlers

## Level 4 — Code (HIGH-risk services)
- [code-bug-code.md](./code-bug-code.md) — service-bug class diagram: BugAggregate hierarchy, StatusWorkflowConfig value objects, policy evaluation order, event emission points
- [code-attachment-code.md](./code-attachment-code.md) — service-attachment class diagram: AttachmentAggregate + FlagInstance children, FlagTypeAggregate + FlagTypeScope, BugMovedFlagRetargetHandler cross-service logic

## Stability Ratings
- [stability-ratings.md](./stability-ratings.md) — Per-element stability table (L2 + L3) with Mermaid classDef color-coding snippets

## Manifest
- [manifest.json](./manifest.json) — Scout's slice plan enumerating expected slicer outputs
