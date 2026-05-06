# C4 Level 1 — System Context

## Diagram

```mermaid
C4Context
    title System Context — Evergreen Bug Tracker (Bugzilla Migration)

    Person(reporter, "Bug Reporter", "Files bugs, adds comments, attaches files, searches")
    Person(triager, "Triager / Maintainer", "Resolves, assigns, sets status, reviews flags")
    Person(admin, "System Admin", "Manages users, groups, products, flag types, scheduled reports")

    System(banyan, "Evergreen Bug Tracker", "Seven CQRS/Event Sourcing microservices migrated from Bugzilla Perl monolith")

    System_Ext(smtp, "SMTP Server", "Outbound email delivery for notifications and Whine scheduled reports")
    System_Ext(s3, "Object Storage (S3/MinIO)", "Binary attachment storage; only metadata is event-sourced")
    System_Ext(legacy, "Legacy Bugzilla Import", "Source Perl monolith (250 files, ~111 kLOC); one-time bulk snapshot")
    System_Ext(idp, "Identity Provider (LDAP/RADIUS)", "External authentication sources for multi-method login")

    Rel(reporter, banyan, "files bugs, comments, attachments, searches", "HTTPS")
    Rel(triager, banyan, "triages, resolves, reviews flags, assigns", "HTTPS")
    Rel(admin, banyan, "manages users, groups, products, flag types", "HTTPS")

    Rel(banyan, smtp, "sends notification emails", "SMTP")
    Rel(banyan, s3, "stores and retrieves attachment binaries", "S3 API")
    Rel(banyan, legacy, "one-time data migration", "Import")
    Rel(banyan, idp, "validates credentials", "LDAP/RADIUS")
```

## Notes

- The system comprises seven bounded contexts decomposed from the Bugzilla monolith: bug, user, product, comment, attachment, search, and notification [source: output/phase-4-architecture/context-map.md:20].
- All inter-service communication uses the Published Language of domain events on RabbitMQ (OHS/PL pattern) [source: output/phase-4-architecture/context-map.md:89].
- Service-notification is a terminal consumer that sends emails via SMTP; it has no outbound domain events [source: output/phase-4-architecture/interaction-map.md:345].
- Binary attachments are stored in S3/MinIO while only metadata is event-sourced inside the AttachmentAggregate [source: output/phase-4-architecture/services/service-attachment.md:440].
- Multi-method authentication (DB, LDAP, RADIUS) is inherited from the Bugzilla `Bugzilla::Auth` stack [source: audit-output/cluster-inventory.md:55].

## Citations

- [source: output/phase-4-architecture/context-map.md:20]
- [source: output/phase-4-architecture/context-map.md:89]
- [source: output/phase-4-architecture/interaction-map.md:345]
- [source: output/phase-4-architecture/services/service-attachment.md:440]
- [source: audit-output/cluster-inventory.md:55]
