# ADR-008: Surrogate userId as Aggregate ID

## Status

Accepted

## Context

Bugzilla uses `login_name` (an email address) as the primary user identifier. The `profiles` table has an auto-incrementing `userid` integer column, but throughout the codebase — authentication, session cookies, API tokens, group membership, bug assignments — the email-based `login_name` is the canonical identity reference.

Bugzilla also allows users to change their email address via a token-based confirmation flow (`Bugzilla::Token` issues paired `emailold`/`emailnew` tokens). This is not an edge case; it is a supported account lifecycle operation backed by confirmation emails to both old and new addresses.

If `service-user` were to adopt `login_name` (email) as the Banyan aggregate ID, email changes would require:

1. Creating a new aggregate under the new email key.
2. Replaying or transferring all historical events from the old aggregate stream to the new one.
3. Updating every cross-service reference (bugs assigned to the user, comments authored, group memberships, attachment creators, flag setters, CC list entries) from the old email to the new one.
4. Emitting compensating events across all downstream services to update their read models.

This is prohibitively complex and fragile in an event-sourced system where aggregate IDs are immutable stream keys. It also conflicts with the Banyan platform's convention that aggregate IDs are stable, opaque identifiers not derived from mutable business data.

The `UserAggregate` sketch from exploration identifies these commands and events:

```
UserAggregate (id = userId)
  commands: CreateUser, UpdateUserProfile, DisableUser, EnableUser, ChangePassword, ChangeEmail
  events: UserCreated, UserProfileUpdated, UserDisabled, UserEnabled, PasswordChanged, EmailChanged
```

The `ChangeEmail` command and `EmailChanged` event already imply that email is a mutable field, not an identity key.

## Decision

Use a surrogate UUID as the `UserAggregate` ID (`userId`). The `login_name`/email is a mutable field on the aggregate with a case-insensitive unique constraint enforced at the command handler level.

### Concrete design

1. **Aggregate ID**: `userId` — a UUID v4 generated at account creation time, passed as the aggregate stream key.
2. **Email field**: `loginName` — a mutable string field on the `UserAggregate` state, validated for uniqueness on `CreateUser` and `ChangeEmail` commands.
3. **Unique email enforcement**: The `CreateUserCommandHandler` and `ChangeEmailCommandHandler` query a `UserByEmailReadModel` (projected from `UserCreated` and `EmailChanged` events) to verify no existing user holds the target email. This check also guards against emails currently in a pending email-change token (matching Bugzilla's `is_available_username()` behavior).
4. **Cross-service references**: All events emitted by `service-user` and consumed by other services carry `userId` (UUID), never `loginName`. Downstream read models store `userId` as the stable foreign key.
5. **Email change flow**: The `ChangeEmail` command updates the `loginName` field on the existing aggregate and emits `EmailChanged(userId, oldEmail, newEmail)`. No new aggregate is created; no event replay is needed. The token-based confirmation flow (email old + new addresses) becomes a saga/process in `service-user` that gates the `ChangeEmail` command on two confirmation steps.

### Read models supporting this decision

| Read Model | Purpose |
|---|---|
| `UserProfileReadModel` | Stores `userId`, `loginName`, `realName`, `isEnabled`, etc. Projected from all user events. |
| `UserByEmailReadModel` | Maps `loginName` → `userId` for uniqueness checks and login lookups. Projected from `UserCreated` and `EmailChanged`. Indexed on `loginName` (case-insensitive). |

## Consequences

### What becomes easier

- **Email changes are a single field update**. The `ChangeEmail` command mutates `loginName` on the existing aggregate and emits one event. No aggregate recreation, no event replay, no cross-service reference updates.
- **Cross-service references are stable**. Bugs, comments, attachments, flags, and group memberships reference `userId` (UUID), which never changes. An email rename does not cascade to any downstream service.
- **Authentication decoupling**. Login by email requires one extra lookup (`UserByEmailReadModel` → `userId`), but this cleanly separates the authentication concern (find user by credential) from the identity concern (stable aggregate ID).
- **Future external identity support**. LDAP, RADIUS, or SSO integrations can map `externId` to the same stable `userId` without colliding with email-based login. Bugzilla's `extern_id` field maps naturally to a secondary lookup.
- **Event sourcing alignment**. UUID aggregate IDs are the standard pattern in Banyan CQRS. No special handling for "identity" aggregates.

### What becomes harder

- **Login requires a lookup query**. Authentication by email requires querying `UserByEmailReadModel` to resolve the email to a `userId` before loading the aggregate. This is a standard pattern (every auth system does this) but is an extra read operation compared to Bugzilla's direct email-keyed lookup.
- **Email uniqueness is eventually consistent**. The `UserByEmailReadModel` is projected from events. In a high-concurrency scenario, two `CreateUser` commands with the same email could both pass the uniqueness check before either's event is projected. Mitigation: the command handler can perform an additional aggregate-level check or use a unique constraint on the read model projection table.
- **Migration mapping**. Data migration from Bugzilla's integer `userid` to UUID requires a mapping step. The existing `profiles.userid` integer maps to the new UUID `userId`, with a migration table retained for audit.
- **API compatibility**. Bugzilla's REST API accepts email-based user references in many endpoints (e.g., `assigned_to`, `cc`, `qa_contact`). The Banyan API gateway or query handlers must resolve email references to `userId` before routing commands.

### Trade-offs accepted

- The extra lookup on every login is negligible (single-row index read on a read model table) and is the standard pattern for any system that decouples identity from credentials.
- Email uniqueness enforcement moves from a database `UNIQUE` constraint to a command-handler-level check backed by a projected read model. This is slightly weaker than a hard DB constraint but is acceptable given the low collision probability for email addresses and the ability to add a compensating event if a duplicate is detected post-hoc.

## Alternatives Considered

### Alternative 1: Email as aggregate ID

**Description**: Use `loginName` (email) directly as the aggregate stream key, matching Bugzilla's current model.

**Rejected because**: Email changes would require creating a new aggregate under the new email key and transferring or replaying all events from the old stream. Cross-service references that store the aggregate ID (email) would all need updating. This violates the event-sourcing invariant that aggregate IDs are immutable and introduces a cascading update problem across all services that reference users.

### Alternative 2: Integer sequential ID (Bugzilla-compatible)

**Description**: Use Bugzilla's auto-incrementing integer `userid` as the aggregate ID, preserving backward compatibility.

**Rejected because**: UUIDs are the Banyan platform convention for aggregate IDs. Sequential integers require coordination across instances (no auto-increment in a distributed event store) and leak implementation details (predictable IDs enable enumeration attacks). UUID v4 provides global uniqueness without coordination.

### Alternative 3: Composite key (userId + email)

**Description**: Use a composite key combining a surrogate ID with the email for extra safety.

**Rejected because**: Composite keys add complexity to every cross-service reference and event payload with no real benefit. The surrogate UUID alone provides stable identity; the email is just a mutable field. No downstream service needs both values in the key.
