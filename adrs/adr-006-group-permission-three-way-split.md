# ADR-006: Group Permission Model — Three-Way Split

## Status

Accepted

## Context

Bugzilla's monolithic `User.pm` (103 KB, ~3,400 lines) is a god-object that mixes identity management, authentication, group membership, bug visibility checks, product access control, and notification preference logic. One of the most entangled cross-cutting concerns is the **group-based permission model**, which spans three conceptual domains:

1. **Group membership** — which users belong to which groups, including direct assignment, regex-derived membership, and group-to-group inheritance (`user_group_map`, `group_group_map`).
2. **Per-product permission configuration** — the `group_control_map` matrix that defines, for each group × product pair, what members and non-members can do (entry, membercontrol, othercontrol, canedit, editcomponents, editbugs, canconfirm).
3. **Permission enforcement at bug level** — checks like `can_see_bug` (group visibility + CC/reporter/assignee), `can_enter_product` (entry group control), `can_change_field` (editbugs/canconfirm group controls), and `can_confirm_bug` (canconfirm group control).

In the current Perl codebase, all three domains are handled inside `User.pm` through methods like `in_group(name, product_id?)`, `can_see_bug(bug_id)`, `can_see_product(name)`, and `can_enter_product(name)`. The `group_control_map` is both read and written within `Product.pm` (via `set_group_controls()`), creating a tight two-way coupling between the User and Product modules.

**Why now**: Migrating to Banyan CQRS/Event Sourcing microservices requires decomposing this monolith into bounded contexts. The group permission model cannot be owned by a single service without creating a shared-god-service anti-pattern. A clear ownership split must be decided before specification because it affects:
- Aggregate boundaries (GroupAggregate in service-user, ProductAggregate in service-product, BugAggregate in service-bug)
- Cross-service event contracts (what events carry group/permission data)
- Read model projections (which service projects which data)
- Layer 2 policy placement (where authorization policies execute)

This decision resolves **Clarification R5** (User god-object decomposition scope) and **R10** (group permission model is cross-cutting), and codifies the recommendation from **Q5** (configuration vs. enforcement split).

## Decision

We adopt a **three-way ownership split** for group permissions across three services:

### 1. `service-user` — Group Membership Owner

**Owns**: Group definitions, user-to-group membership, and group-to-group inheritance.

- **Aggregate**: `GroupAggregate` — manages group lifecycle (create, update, deactivate), membership rules (userregexp), and inheritance DAG (group_group_map).
- **Commands**: `CreateGroup`, `UpdateGroup`, `AddGroupMember`, `RemoveGroupMember`
- **Events emitted** (broadcast):
  - `user.Events.GroupMemberAdded(groupId, userId)` — consumed by service-product and service-bug to re-evaluate per-product permissions and bug visibility.
  - `user.Events.GroupMemberRemoved(groupId, userId)` — same consumers.
  - `user.Events.GroupCreated(groupId, name, userregexp)` — consumed by service-product to initialize default group_control_map entries if `makeproductgroups` is enabled.
  - `user.Events.GroupUpdated(groupId, name, userregexp)` — when regex changes, service-user re-derives regex-based memberships internally and emits individual `GroupMemberAdded`/`GroupMemberRemoved` events for each affected user.
- **Read models**: `GroupListReadModel`, `UserGroupMembershipReadModel` (per-user group membership including inherited groups).

**Does NOT own**: Any product-scoped permission logic, bug visibility rules, or `group_control_map` data.

### 2. `service-product` — Group Control Configuration Owner

**Owns**: The `group_control_map` matrix — per-product, per-group permission configuration.

- **Aggregate**: `ProductAggregate` — includes group control map as a managed collection. The `UpdateGroupControls` command validates the membercontrol/othercontrol legality matrix and manages cascading effects.
- **Commands**: `UpdateGroupControls(productId, groupId, { entry, membercontrol, othercontrol, canedit, editcomponents, editbugs, canconfirm })`
- **Events emitted** (broadcast):
  - `product.Events.GroupControlsUpdated(productId, groupId, controls)` — consumed by service-bug to update its projected `ProductGroupControlsReadModel`.
  - `product.Events.GroupControlsMadeMandatory(productId, groupId)` — special-case event when membercontrol is changed to MANDATORY, triggering service-bug to retroactively add the group to all existing bugs in that product.
- **Read models**: `GroupControlsReadModel` (per-product group permission matrix), for admin UI consumption.

**Does NOT own**: Any bug-level authorization enforcement or user membership data.

### 3. `service-bug` — Permission Enforcement Owner

**Owns**: All runtime permission checks that determine whether a user can see, create, or modify bugs.

- **Read models projected from cross-service events**:
  - `ProductGroupControlsReadModel` — projected from `product.Events.GroupControlsUpdated`. Stores the per-product group permission matrix locally so service-bug never makes synchronous calls to service-product during command handling.
  - `UserGroupMembershipReadModel` — projected from `user.Events.GroupMemberAdded` / `GroupMemberRemoved`. Stores per-user group membership (including transitive inheritance) locally.
- **Layer 2 Policies** (business policies on command handlers):
  - `CanSeeBugPolicy` — checks bug visibility: user's groups vs. `bug_group_map` (groups on the bug), plus `cclist_accessible`, `reporter_accessible`, and assignee/reporter/QA checks. Consults both `ProductGroupControlsReadModel` and `UserGroupMembershipReadModel`.
  - `CanChangeFieldPolicy` — implements `check_can_change_field()` logic: varies by field (reporter can set some fields, assignee can reassign, editbugs group required for others). Consults `ProductGroupControlsReadModel` for the `editbugs` flag on the bug's product.
  - `CanConfirmBugPolicy` — checks if the user is in a group with `canconfirm` control for the bug's product, OR if the user is the reporter (Bugzilla allows reporters to confirm their own bugs in some configurations).
  - `CanEnterProductPolicy` — checks the `entry` flag in `ProductGroupControlsReadModel` to determine if a user can file bugs in a product.
  - `CanEditProductPolicy` — checks the `canedit` flag in `ProductGroupControlsReadModel`.
- **Event subscription handlers**: Subscribes to `user.Events.*` and `product.Events.*` to keep projected read models current.

**Does NOT own**: Group definitions, membership management, or group control configuration.

### Event Flow Summary

```
service-user                     service-product                    service-bug
─────────────                    ───────────────                    ───────────
GroupMemberAdded ──────────────────────────────────────────────→ UserGroupMembershipReadModel
GroupMemberRemoved ───────────────────────────────────────────→ UserGroupMembershipReadModel
GroupCreated ─────────────────→ (init default group_control_map)
                                                             
                                 GroupControlsUpdated ────────→ ProductGroupControlsReadModel
                                 GroupControlsMadeMandatory ──→ (retroactive bug_group_map update)
                                                             
                                                                 Layer 2 Policies query:
                                                                 - UserGroupMembershipReadModel
                                                                 - ProductGroupControlsReadModel
```

## Consequences

### What becomes easier

1. **Clear bounded context boundaries** — each service has a well-defined ownership slice with no ambiguity about where permission logic lives.
2. **Independent scaling** — group membership changes (infrequent, admin-driven) don't couple to bug visibility checks (frequent, every bug read/write).
3. **Local authorization decisions** — service-bug never makes synchronous cross-service calls during command handling; all authorization data is available via projected read models.
4. **Testability** — each service's permission logic can be tested in isolation with mocked read model data.
5. **Alignment with Banyan patterns** — the split naturally maps to Layer 1 (permissions on `@Command` contracts) and Layer 2 (`@RequirePolicy` on handlers), with read models projected from cross-service events.

### What becomes harder

1. **Eventual consistency of authorization** — when a user is added to or removed from a group, there is a propagation delay before service-bug's `UserGroupMembershipReadModel` reflects the change. A user might temporarily see (or be denied) bugs based on stale group data.
   - **Mitigation**: Group membership changes are infrequent and typically admin-driven. The read model projection lag is typically sub-second. For critical security group changes, a UI hint ("Changes may take a moment to propagate") is acceptable.
2. **Mandatory group cascading** — when a product makes a group MANDATORY, all existing bugs in that product must be retroactively added to `bug_group_map`. This requires a saga/process manager in service-bug that handles `GroupControlsMadeMandatory` events, iterates all bugs in the product, and emits `BugGroupAdded` events.
   - **Mitigation**: This is a rare admin operation. The saga can run asynchronously with progress tracking.
3. **Read model projection complexity** — service-bug must project and maintain two cross-origin read models (`UserGroupMembershipReadModel` from user events, `ProductGroupControlsReadModel` from product events). These must be kept consistent and queryable.
   - **Mitigation**: The Banyan platform auto-discovers `@ReadModel` decorated classes. Each read model is a standard projection with `@MapFromEvent` decorators.
4. **Group inheritance computation** — Bugzilla supports transitive group-to-group inheritance (`group_group_map` DAG). The `UserGroupMembershipReadModel` must resolve the full flattened membership set, including inherited groups.
   - **Mitigation**: service-user emits pre-resolved membership events (after flattening). Service-bug's read model stores the flattened set, not the raw DAG. The inheritance resolution complexity stays in service-user.

### Trade-offs accepted

- **Stale reads for authorization** — we accept eventual consistency for permission enforcement in exchange for avoiding synchronous cross-service calls. This is appropriate for a bug tracker where permission changes are rare and the worst case (briefly showing/hiding a bug) is low-impact.
- **Duplicate group data** — group membership and group control data exists in both the owning service and the projected read models in service-bug. This is intentional: it enables local, low-latency authorization checks without coupling services at runtime.
- **Three-service coordination for permission changes** — an admin changing group permissions on a product triggers events in two services (service-user for membership, service-product for controls) that are both consumed by service-bug. This is inherent to the three-way split and preferable to a monolithic permission service.

## Permission Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ADMIN CONFIGURES PERMISSIONS                       │
└─────────────────────────────────────────────────────────────────────────────┘

  1. Add user to group               2. Configure group controls on product
     ┌──────────────┐                   ┌───────────────┐
     │ service-user │                   │service-product │
     │              │                   │                │
     │ AddGroupMember                  │ UpdateGroupControls
     │    ↓         │                   │    ↓           │
     │ GroupAggregate                  │ ProductAggregate
     │    ↓         │                   │    ↓           │
     │ GroupMember  │                   │ GroupControls  │
     │ Added event  │                   │ Updated event  │
     └──────┬───────┘                   └───────┬────────┘
            │ broadcast                         │ broadcast
            ▼                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          EVENTS ON MESSAGE BUS                               │
│                                                                              │
│  user.Events.GroupMemberAdded     product.Events.GroupControlsUpdated        │
│  user.Events.GroupMemberRemoved   product.Events.GroupControlsMadeMandatory  │
└──────────────────────────┬───────────────────────────┬───────────────────────┘
                           │                           │
                           ▼                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     service-bug: PROJECTED READ MODELS                       │
│                                                                              │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────┐     │
│  │ UserGroupMembershipReadModel│  │ ProductGroupControlsReadModel      │     │
│  │                             │  │                                    │     │
│  │ userId → [groupId, ...]    │  │ (productId, groupId) → {           │     │
│  │ (flattened with            │  │   entry, membercontrol,            │     │
│  │  inheritance)              │  │   othercontrol, canedit,           │     │
│  │                            │  │   editcomponents, editbugs,        │     │
│  │                            │  │   canconfirm                       │     │
│  │                            │  │ }                                  │     │
│  └──────────────┬──────────────┘  └──────────────┬─────────────────────┘     │
│                 │                                │                          │
│                 └──────────────┬─────────────────┘                          │
│                                ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Layer 2 POLICIES (on command handlers)           │    │
│  │                                                                     │    │
│  │  CanSeeBugPolicy          → Is user in a group on this bug?        │    │
│  │                              Is bug's group in user's groups?      │    │
│  │                              Is user reporter/assignee/CC/QA?      │    │
│  │                                                                     │    │
│  │  CanChangeFieldPolicy     → Does user's group have editbugs        │    │
│  │                              control on this bug's product?         │    │
│  │                              Is user reporter/assignee?             │    │
│  │                                                                     │    │
│  │  CanConfirmBugPolicy      → Does user's group have canconfirm      │    │
│  │                              control on this bug's product?         │    │
│  │                              Is user the reporter?                  │    │
│  │                                                                     │    │
│  │  CanEnterProductPolicy    → Does user's group have entry           │    │
│  │                              control on this product?               │    │
│  │                                                                     │    │
│  │  CanEditProductPolicy     → Does user's group have canedit         │    │
│  │                              control on this product?               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                     USER OPERATES ON A BUG                                   │
└─────────────────────────────────────────────────────────────────────────────┘

  User → API Gateway → service-bug command handler
                           │
                           ├─ Layer 1: @Command({ permissions: ['bugs:update'] })
                           │    (gateway checks user has 'bugs:update' permission)
                           │
                           └─ Layer 2: @RequirePolicy('CanChangeFieldPolicy')
                                │
                                ├─ Query UserGroupMembershipReadModel
                                │    → user is in groups [A, B, C]
                                │
                                ├─ Query ProductGroupControlsReadModel
                                │    → product P: group A has editbugs=true
                                │
                                └─ User is in group A → ALLOW
```

## Alternatives Considered

### Alternative 1: Single `service-user` owns all group permission logic

Group definitions, membership, `group_control_map`, and bug visibility enforcement all remain in `service-user`, matching Bugzilla's `User.pm` structure.

**Rejected because**: This recreates the god-object anti-pattern at the service level. `service-user` would need to know about product IDs (for `group_control_map`), bug IDs (for `can_see_bug`), and would require synchronous calls from service-bug for every bug operation. It violates the bounded context principle — service-user would need intimate knowledge of both the product and bug domains.

### Alternative 2: Dedicated `service-authorization` service

A fourth service owns all permission evaluation, with service-bug making synchronous RPC calls for authorization checks.

**Rejected because**: Every bug command (read or write) would require a synchronous cross-service call, creating a latency bottleneck and a single point of failure. Authorization is not a separate bounded context — it is a cross-cutting concern best handled by each service locally using projected data, aligned with Banyan's Layer 1 + Layer 2 authorization model.

### Alternative 3: `service-product` owns both configuration and enforcement

Since `group_control_map` is per-product, service-product could own both the configuration data and the enforcement logic, serving as an authorization gateway for bug operations.

**Rejected because**: Bug visibility enforcement requires intimate knowledge of bug data (CC list, reporter, assignee, bug groups, reporter_accessible, cclist_accessible flags). Placing this in service-product would require service-product to subscribe to bug events and maintain bug-state read models, creating a circular dependency between service-bug and service-product. Enforcement belongs where the data lives — in service-bug.
