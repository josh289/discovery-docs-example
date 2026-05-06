# Cluster Inventory

## Overview

The `discovery/` directory contains a single legacy application: **Bugzilla**, the open-source Perl-based bug-tracking system (250 source files, ~111 kLOC across `.pm`, `.cgi`, `.pl`). The codebase was imported as a single bulk snapshot on 2026-05-05; the `discovery/bugzilla/` path is excluded from git tracking (`.gitignore`), so every file reports `(unversioned)` for author and zero churn. There are no git commits to draw stability signals from — the heuristic falls back to test proximity and architectural analysis.

This is a **legacy migration** source: the entire Bugzilla Perl codebase serves as input to the Evergreen factory, which re-architects it as Banyan CQRS/Event Sourcing microservices in TypeScript. The bounded contexts identified during discovery (see `output/phase-1-discovery/discovery.md`) define the cluster boundaries below.

## Clusters

- **service-bug** — The central bug-tracking aggregate. Encompasses `Bugzilla::Bug` (5 123 LOC — the god-object), status workflow (`Bugzilla::Status`), bug mail triggering (`Bugzilla::BugMail`), bug URLs, user-last-visit tracking, and the primary CGI entry points for viewing, creating, and processing bugs. This is the largest and most architecturally significant cluster; the entire monolith orbits it.

- **service-user** — User account lifecycle, authentication (multi-method: DB, LDAP, RADIUS, Env), group management, API keys, user settings, and CSRF tokens. Includes the auth stack (`Bugzilla::Auth`, `Bugzilla::Auth::Verify`, `Bugzilla::Auth::Login`) and the admin CGIs for user/group management. Notably, `Bugzilla::User` (3 407 LOC) contains bug-visibility logic that belongs in `service-bug` and product-access logic that belongs in `service-product` (see exploration findings).

- **service-product** — The Classification → Product → Component hierarchy, versions, milestones, and product-level group permission controls. A relatively clean hierarchical domain with well-defined CRUD operations and one notable side effect: mandatory-group cascading retroactively adds all existing bugs to a new mandatory group.

- **service-attachment** — File attachments on bugs, flag types, and the request/grant/deny flag workflow for reviews and approvals. Flags can target both bugs and attachments, creating a cross-cutting concern. The `Bugzilla::Flag` module (1 309 LOC) is the second-largest single module after `Bug` and `User`.

- **service-comment** — Append-only comment thread on bugs, including privacy (insider groups), user-applied tags, and tag weighting. The smallest domain cluster at 724 total LOC. Comments are immutable after creation (body cannot be edited, only privacy toggled).

- **service-search** — The boolean-chart SQL generation engine (`Bugzilla::Search` at 3 561 LOC), quicksearch, saved/shared queries, chart/reporting, and series data. Primarily a query-side concern. The search engine is a pure SQL-generation pipeline with 25+ operators that would be substantially rewritten or replaced in the Banyan migration. Has the only meaningful integration test in the codebase (`xt/search.t`).

- **service-notification** — Email notification pipeline, scheduled reports (whine system), and background job queue (TheSchwartz). A pure event consumer that computes recipients, renders per-user templated emails, and manages cron-driven scheduled reports. Depends on domain events from bug, comment, attachment, and user services.

## Source Files

### service-bug

| Path | Last-Modified | LOC | Primary Author | Stability |
|---|---|---|---|---|
| discovery/bugzilla/Bugzilla/Bug.pm | 2026-05-05 | 5123 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla.pm | 2026-05-05 | 1106 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/BugMail.pm | 2026-05-05 | 658 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/process_bug.cgi | 2026-05-05 | 446 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/post_bug.cgi | 2026-05-05 | 246 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Status.pm | 2026-05-05 | 341 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/BugUrl.pm | 2026-05-05 | 218 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/show_bug.cgi | 2026-05-05 | 141 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/BugUserLastVisit.pm | 2026-05-05 | 95 | (unversioned) | unknown (no nearby tests, 0 churn) |

### service-user

| Path | Last-Modified | LOC | Primary Author | Stability |
|---|---|---|---|---|
| discovery/bugzilla/Bugzilla/User.pm | 2026-05-05 | 3407 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editusers.cgi | 2026-05-05 | 775 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Group.pm | 2026-05-05 | 732 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Token.pm | 2026-05-05 | 692 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/userprefs.cgi | 2026-05-05 | 680 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Auth.pm | 2026-05-05 | 550 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editgroups.cgi | 2026-05-05 | 455 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/User/Setting.pm | 2026-05-05 | 449 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Auth/Verify.pm | 2026-05-05 | 264 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/User/APIKey.pm | 2026-05-05 | 157 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Auth/Login.pm | 2026-05-05 | 137 | (unversioned) | unknown (no nearby tests, 0 churn) |

### service-product

| Path | Last-Modified | LOC | Primary Author | Stability |
|---|---|---|---|---|
| discovery/bugzilla/Bugzilla/Product.pm | 2026-05-05 | 1185 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Component.pm | 2026-05-05 | 686 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editproducts.cgi | 2026-05-05 | 443 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Milestone.pm | 2026-05-05 | 398 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Version.pm | 2026-05-05 | 372 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Classification.pm | 2026-05-05 | 308 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editcomponents.cgi | 2026-05-05 | 249 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editclassifications.cgi | 2026-05-05 | 238 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editmilestones.cgi | 2026-05-05 | 222 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editversions.cgi | 2026-05-05 | 211 | (unversioned) | unknown (no nearby tests, 0 churn) |

### service-attachment

| Path | Last-Modified | LOC | Primary Author | Stability |
|---|---|---|---|---|
| discovery/bugzilla/Bugzilla/Flag.pm | 2026-05-05 | 1309 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Attachment.pm | 2026-05-05 | 1055 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/attachment.cgi | 2026-05-05 | 849 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/FlagType.pm | 2026-05-05 | 797 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/editflagtypes.cgi | 2026-05-05 | 579 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Attachment/PatchReader.pm | 2026-05-05 | 332 | (unversioned) | unknown (no nearby tests, 0 churn) |

### service-comment

| Path | Last-Modified | LOC | Primary Author | Stability |
|---|---|---|---|---|
| discovery/bugzilla/Bugzilla/Comment.pm | 2026-05-05 | 646 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Comment/TagWeights.pm | 2026-05-05 | 78 | (unversioned) | unknown (no nearby tests, 0 churn) |

### service-search

| Path | Last-Modified | LOC | Primary Author | Stability |
|---|---|---|---|---|
| discovery/bugzilla/Bugzilla/Search.pm | 2026-05-05 | 3561 | (unversioned) | well-tested (sibling test `xt/search.t`, 0 churn) |
| discovery/bugzilla/buglist.cgi | 2026-05-05 | 1165 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Search/Quicksearch.pm | 2026-05-05 | 717 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Search/Saved.pm | 2026-05-05 | 416 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Chart.pm | 2026-05-05 | 475 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/query.cgi | 2026-05-05 | 328 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Series.pm | 2026-05-05 | 330 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/chart.cgi | 2026-05-05 | 363 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Search/Clause.pm | 2026-05-05 | 162 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Search/Recent.pm | 2026-05-05 | 194 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/reports.cgi | 2026-05-05 | 226 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Search/ClauseGroup.pm | 2026-05-05 | 116 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Search/Condition.pm | 2026-05-05 | 107 | (unversioned) | unknown (no nearby tests, 0 churn) |

### service-notification

| Path | Last-Modified | LOC | Primary Author | Stability |
|---|---|---|---|---|
| discovery/bugzilla/whine.pl | 2026-05-05 | 673 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/JobQueue/Runner.pm | 2026-05-05 | 271 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Mailer.pm | 2026-05-05 | 273 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/JobQueue.pm | 2026-05-05 | 215 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Whine/Schedule.pm | 2026-05-05 | 169 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Whine/Query.pm | 2026-05-05 | 133 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Whine.pm | 2026-05-05 | 121 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Job/Mailer.pm | 2026-05-05 | 48 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/Bugzilla/Job/BugMail.pm | 2026-05-05 | 32 | (unversioned) | unknown (no nearby tests, 0 churn) |
| discovery/bugzilla/jobqueue.pl | 2026-05-05 | 76 | (unversioned) | unknown (no nearby tests, 0 churn) |

## External Dependencies

### Python (docs/requirements.txt)

- `sphinx` ==5.3.0 — documentation build tool
- `sphinx_rtd_theme` ==1.3.0 — Read the Docs theme
- `readthedocs-sphinx-search` ==0.3.2 — search extension

### Perl (Bugzilla/Install/Requirements.pm — REQUIRED_MODULES)

- `CGI.pm` ≥ 3.51 — web framework / CGI handling
- `Digest-SHA` ≥ 0 — SHA digest (any version)
- `TimeDate` (Date::Format) ≥ 2.23 — date formatting
- `DateTime` ≥ 0.75 — datetime objects
- `DateTime-TimeZone` ≥ 1.64 — timezone support
- `DBI` ≥ 1.54 (1.614 on Perl ≥ 5.13.3) — database driver interface
- `DBIx-Connector` ≥ 0.56 — DBI connection management
- `Moo` ≥ 2.003004 — lightweight object system
- `Template-Toolkit` ≥ 3.008 — template engine
- `Email-Sender` ≥ 2.600 — email sending
- `Email-Address-XS` ≥ 1.05 — email address parsing
- `Email-MIME` ≥ 1.904 — MIME email construction
- `URI` ≥ 1.55 — URI handling
- `List-MoreUtils` ≥ 0.32 — list utilities
- `Math-Random-ISAAC` ≥ 1.0.1 — cryptographically secure RNG
- `JSON-XS` ≥ 2.01 — JSON serialization

### Perl (Bugzilla/Install/Requirements.pm — OPTIONAL_MODULES)

- `GD` ≥ 1.20 — graphical reports / charts
- `Chart` (Chart::Lines) ≥ 2.4.1 — chart generation
- `Template-GD` (Template::Plugin::GD::Image) ≥ 0 — GD integration for templates
- `GDTextUtil` (GD::Text) ≥ 0 — text rendering for charts
- `GDGraph` (GD::Graph) ≥ 0 — graph plotting
- `MIME-tools` (MIME::Parser) ≥ 5.406 — old-bug-move import
- `libwww-perl` (LWP::UserAgent) ≥ 0 — HTTP client (upgrade notifications)
- `XML-Twig` (XML::Twig) ≥ 0 — XML import/export
- `PatchReader` ≥ 0.9.6 — patch viewer
- `perl-ldap` (Net::LDAP) ≥ 0 — LDAP authentication
- `Authen-SASL` ≥ 0 — SASL authentication (SMTP)
- `Net-SMTP-SSL` ≥ 1.01 — SMTP over SSL
- `RadiusPerl` (Authen::Radius) ≥ 0 — RADIUS authentication
- `SOAP-Lite` ≥ 0.712 — XMLRPC transport
- `XMLRPC-Lite` ≥ 0.712 — XMLRPC protocol
- `JSON-RPC` ≥ 0 — JSONRPC / REST protocol
- `Test-Taint` ≥ 1.06 — taint-mode testing (API tests)
- `HTML-Parser` ≥ 3.40 (3.67 on Perl ≥ 5.13.3) — HTML parsing
- `HTML-Scrubber` ≥ 0 — HTML sanitization
- `Encode` ≥ 2.21 — character encoding detection
- `Encode-Detect` ≥ 0 — charset detection
- `Email-Reply` ≥ 0 — inbound email parsing
- `HTML-FormatText-WithLinks` (HTML::FormatText::WithLinks) ≥ 0.13 — HTML-to-text for inbound email
- `TheSchwartz` ≥ 1.07 — background job queue
- `Daemon-Generic` ≥ 0 — jobqueue daemonization
- `mod_perl` (mod_perl2) ≥ 1.999022 — Apache embedded Perl
- `Apache-SizeLimit` (Apache2::SizeLimit) ≥ 0.96 — Apache process size management
- `File-MimeInfo` (File::MimeInfo::Magic) ≥ 0 — MIME type detection
- `IO-stringy` (IO::Scalar) ≥ 0 — in-memory file handles
- `Cache-Memcached` ≥ 0 — memcached client

## Hidden Dependencies and Side Effects

- **`Bugzilla::Bug` — god-object coupling**: `Bug.pm` (5 123 LOC) serves as both data-access object and command processor for all bug mutations. It directly calls `Bugzilla::BugMail`, `Bugzilla::Flag`, `Bugzilla::Attachment`, `Bugzilla::User`, and `Bugzilla::Hook` within mutation methods. Every side effect (email, fulltext index update, extension hooks) fires synchronously within the same DB transaction. *(see output/phase-2-exploration/slices/bug-core-domain.md "cross-table side effects")*

- **`Bugzilla::Bug` — `see_also` recursive behavior**: Adding a local bug URL automatically adds the reverse link on the target bug — a cross-aggregate side effect within the monolith. *(see output/phase-2-exploration/slices/bug-core-domain.md line 322)*

- **`Bugzilla::User` — misplaced domain logic**: `User.pm` (3 407 LOC) contains `can_see_bug()` and `visible_bugs()` (belongs in `service-bug`), `can_see_product()` and `can_enter_product()` (belongs in `service-product`), and `wants_bug_mail()` (belongs in `service-notification`). *(see output/phase-2-exploration/exploration.md "Critical decomposition note")*

- **Memcached as global shared state**: Multiple modules (`Bugzilla::Memcached`, `Bugzilla::Status`, `Bugzilla::Config`) cache computed values in a shared memcached instance with manual invalidation. Status workflow states are cached with keys like `status_bug_state_open`; invalidation occurs on status create/delete. *(see output/phase-2-exploration/slices/bug-status-workflow.md "Cache invalidation")*

- **Extension hook system — shared mutable state**: `Bugzilla::Hook::process()` passes a shared mutable `$args` hashref to all extensions in load order. Extensions can modify it, and downstream extensions see those modifications — cooperative mutation with no ordering guarantees. *(see output/phase-2-exploration/slices/extension-system.md "Shared mutable state")*

- **Product creation multi-step saga**: Creating a product triggers: insert product row → auto-create first version → auto-create default milestone → optionally create bug group → optionally create charting series → fire `product_end_of_create` hook → clear memcached config/user caches. *(see output/phase-2-exploration/slices/product-hierarchy.md "Product creation side-effects")*

- **Mandatory group cascading**: When a group becomes MANDATORY for a product, all existing bugs are retroactively added to that group — a cross-aggregate side effect (product change → bug mutation). *(see output/phase-2-exploration/slices/product-hierarchy.md line 289)*

- **Comment privacy toggle triggers fulltext re-index**: Changing `isprivate` on a comment calls `$self->bug->_sync_fulltext(update_comments => 1)`, re-indexing the parent bug's fulltext search index synchronously. *(see output/phase-2-exploration/slices/comments.md line 49)*

- **Password hash auto-upgrade on login**: The DB verifier auto-upgrades password hashes (old algorithm → SHA-256 with salt) on successful login — a write side effect hidden inside an authentication read path. *(see output/phase-2-exploration/slices/user-accounts-auth.md line 374)*

- **Environment variable auth (`Login::Env`)**: The `auth_env_id`, `auth_env_email`, `auth_env_realname` environment variables are read at authentication time for SSO proxy integration. In Banyan, this would need gateway-level extraction. *(see output/phase-2-exploration/slices/user-accounts-auth.md line 77)*

- **Attachment/Flag changes logged to `bugs_activity`**: Both attachment and flag mutations write to the shared `bugs_activity` table (prefixed with `attachments.` or `flagtypes.name`), then flush memcached. The activity log is owned by the bug domain but written from attachment/flag code. *(see output/phase-2-exploration/slices/attachments-flags.md line 184)*

- **Extension `disabled` file check at load time**: A `disabled` file in the extension directory causes Bugzilla to skip the extension entirely during module loading — a filesystem-based configuration side effect at import time. *(see output/phase-2-exploration/slices/extension-system.md line 29)*

- **`localconfig` read once at startup**: Configuration secrets (DB credentials, etc.) are read from `localconfig` once at startup and cached for the process lifetime. Banyan services will need secret rotation support. *(see output/phase-2-exploration/slices/platform-infrastructure.md line 247)*

## Stability Ratings

```json
{
  "service-bug": "unknown",
  "service-user": "unknown",
  "service-product": "unknown",
  "service-attachment": "unknown",
  "service-comment": "unknown",
  "service-search": "unknown",
  "service-notification": "unknown"
}
```
