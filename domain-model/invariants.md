# Domain Invariants

Harvested from `output/phase-4-architecture/services/service-*.md` and
`output/phase-5-specification/specs/service-*/SERVICE_SPEC.md`.

| Invariant ID | Service | Statement | Source |
|---|---|---|---|
| I1 | service-bug | A bug cannot depend on itself. Enforced by DependencyLoopPolicy on AddBugDependency. | output/phase-4-architecture/services/service-bug.md:455 |
| I2 | service-bug | Adding a dependency cannot create a cycle in the dependency graph. The full dependency tree is traversed in both directions; if the union of dependson-tree and blocked-tree has any overlap, the operation is rejected. | output/phase-4-architecture/services/service-bug.md:458 |
| I3 | service-bug | A bug cannot be resolved as FIXED if any of its dependencies are still in an open status (noresolveonopenblockers). Enforced by NoOpenBlockersPolicy. | output/phase-4-architecture/services/service-bug.md:461 |
| I4 | service-bug | When transitioning to any closed status, a resolution must be provided. When transitioning back to open, resolution and duplicateOf are cleared. | output/phase-4-architecture/services/service-bug.md:464 |
| I5 | service-bug | Every status transition must correspond to an active edge in the StatusWorkflowReadModel directed graph. Self-loops (remaining in current status) are always allowed. | output/phase-4-architecture/services/service-bug.md:467 |
| I6 | service-bug | Bug aliases must be globally unique across all bugs. Max 40 characters, cannot be purely numeric, no commas or spaces. | output/phase-4-architecture/services/service-bug.md:470 |
| I7 | service-bug | The reporterId field is set at creation and cannot be changed by any subsequent command. | output/phase-4-architecture/services/service-bug.md:473 |
| I8 | service-bug | Custom field values are validated against their field type definitions (freetext max length, single_select legal value, multi_select array of legal values, datetime/date valid format, bug_id visible reference no loops, integer signed INT32 range). | output/phase-4-architecture/services/service-bug.md:476 |
| I9 | service-bug | Field validators must execute in topological order per VALIDATOR_DEPENDENCIES (component depends on product, version depends on product, etc.). | output/phase-4-architecture/services/service-bug.md:479 |
| I10 | service-bug | Time-tracking fields (estimatedTime, remainingTime, deadline) belong to service-bug. remainingTime is zeroed when a bug is closed or marked as duplicate. Only users in timetrackinggroup can view or modify these fields. | output/phase-4-architecture/services/service-bug.md:482 |
| I11 | service-bug | Bug commands use optimistic concurrency. The command handler checks the aggregate's expected stream version before applying changes; on mismatch, a structured conflict error is returned. | output/phase-4-architecture/services/service-bug.md:485 |
| I12 | service-bug | everConfirmed is derived from event history (true if the bug has ever been in any status other than UNCONFIRMED). It is NOT stored as mutable state. | output/phase-4-architecture/services/service-bug.md:488 |
| I13 | service-bug | The BugDuplicateReadModel follows the transitive duplicate chain to prevent infinite loops in duplicate references. | output/phase-4-architecture/services/service-bug.md:491 |
| I14 | service-bug | Custom fields marked is_mandatory must have non-empty values on bug creation and update, unless the field is not visible or is a single-select with only one legal value. | output/phase-4-architecture/services/service-bug.md:494 |
| I15 | service-bug | Bugs with group restrictions are only visible to users who are members of at least one of those groups OR to users with the admin role. | output/phase-4-architecture/services/service-bug.md:497 |
| P1 | service-product | Product names must be globally unique, checked case-insensitively. | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:304 |
| P2 | service-product | Every product must have at least one Version at all times. The last version in a product cannot be deleted (MinimumVersionPolicy). | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:305 |
| P3 | service-product | Every product must have at least one Component at all times. The last component in a product cannot be deleted (MinimumComponentPolicy). | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:306 |
| P4 | service-product | The product's current default milestone cannot be deleted (NotDefaultMilestonePolicy). An administrator must first set a different default milestone. | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:307 |
| P5 | service-product | The membercontrol/othercontrol matrix on group controls must satisfy the legality constraint: othercontrol must be at or below membercontrol in the hierarchy (NA ≤ SHOWN ≤ DEFAULT ≤ MANDATORY). | output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:313 |
| U1 | service-user | loginName must be unique across all users, compared case-insensitively. | output/phase-4-architecture/services/service-user.md:28 |
| U2 | service-user | loginName must be a valid email address format. | output/phase-4-architecture/services/service-user.md:29 |
| U3 | service-user | Group inheritance DAG must be acyclic. Adding an inheritance edge that would create a cycle is rejected. | output/phase-4-architecture/services/service-user.md:42 |
| U4 | service-user | The system must prevent disabling the last remaining user who is a member of the admin group. | output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:112 |
| U5 | service-user | externId, if provided, must be unique across all users. | output/phase-4-architecture/services/service-user.md:31 |
| A1 | service-attachment | When isObsolete transitions from false to true, every flag instance on the attachment with status = '?' must be canceled (obsolete cascade). | output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:241 |
| A2 | service-attachment | A flag type is available for a target only when the target's product/component matches at least one inclusion scope AND does not match any exclusion scope. | output/phase-4-architecture/services/service-attachment.md:99 |
| A3 | service-attachment | A flag type with isActive = false cannot have new flag instances created. Existing flags remain in their current state. | output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:245 |
| A4 | service-attachment | If a flag type has isMultiplicable = false, at most one flag of that type may exist on a given target. | output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:247 |
| C1 | service-comment | Comment body is immutable after creation. No EditComment command exists. | output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:93 |
| C2 | service-comment | Tags must be 3–24 characters, match the pattern [\w\d\._-]+, and are unique per comment (case-insensitive). | output/phase-4-architecture/services/service-comment.md:26 |
| C3 | service-comment | Comment type 0 is the only user-authorable type. Types 1, 2, 5, and 6 are set exclusively by system event subscription handlers. | output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:97 |
| N1 | service-notification | A scheduled report must always have at least one query binding and at least one mail target while in the Active state. | output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:210 |
| N2 | service-notification | The runNext timestamp must always be recomputed whenever the schedule pattern changes or a scheduled execution completes. | output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:211 |
| N3 | service-notification | Only the owner user can transition a report to Updated or Deleted states. | output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:212 |
| S1 | service-search | A saved search name must be unique among all saved searches owned by the same user. | output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:86 |
| S2 | service-search | Deleting a saved search is rejected if any ScheduledReportAggregate references it. | output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:90 |
