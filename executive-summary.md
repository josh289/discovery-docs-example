# Executive Summary

We identified **7 bounded contexts** [source: audit-output/cluster-inventory.md] across a ~111 kLOC Perl monolith (Bugzilla) targeted for re-architecture as Evergreen CQRS/Event Sourcing microservices. The audit catalogued **492 decision rules** spanning validation, business-policy, state-transition, permission, and notification categories [source: audit-output/decision-rules.md]. The phased statement of work estimates **324 story points (~13.5 dev-months at 3 developers)** to complete the migration, sequenced as foundation services first, then core domain, then query/notification leaves, then data-migration cutover [source: audit-output/sow.md]. Six of seven clusters carry `unknown` stability ratings — the codebase was imported as a single bulk snapshot with zero git history — meaning every service carries the risk profile of untested code [source: audit-output/cluster-inventory.md].

## Three Surprises

- **Event naming inconsistencies already exist in the architecture docs** — four mismatches (e.g., `BugProductChanged` vs `BugMoved`) where `@EventHandlerDecorator` references events by string, so a typo silently creates a dead subscription handler that never fires [source: audit-output/risk-register.md#R-004].
- **Scheduled report ownership is split across two services with duplicate permission rules** — both service-notification and service-search define independent `CreateScheduledReport`/`DeleteScheduledReport` commands gated by different permission strings (`notifications:schedule:*` vs `search:report:*`), with no reconciliation [source: audit-output/decision-rules.md#duplicates-and-conflicts].
- **`bugs:admin_workflow` — the highest-privilege bug permission — has no Layer 2 domain policy**; any user with that JWT claim can modify the global status workflow without a business-logic check, a gap not present in any other high-privilege operation [source: audit-output/permission-matrix.md#gaps--inconsistencies].

## Recommended Path Forward

Follow the four-phase strangler-fig sequence from the SOW: stand up service-user and service-product first (zero inbound dependencies), then the core bug/comment/attachment triad, then the search and notification leaf services, then freeze-cut-transform-load the data migration with a REST compatibility shim preserving all 40+ legacy endpoints [source: audit-output/sow.md]. The phased gates between stages provide explicit rollback checkpoints before any irreversible state change.

### Alternative Considered

A big-bang rewrite of the entire monolith in a single release was rejected because it eliminates the rollback checkpoints that the phased strangler-fig approach preserves at every boundary [source: audit-output/sow.md#Rollback-Boundaries]. With all seven clusters at `unknown` stability and no test coverage signals, a single irreversible cutover would compound the data-migration risk (R-010: non-resumable FETL with no rollback) across every service simultaneously [source: audit-output/risk-register.md#R-010].
