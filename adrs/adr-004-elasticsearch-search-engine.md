# ADR-004: Elasticsearch for Bug Search

## Status

Accepted

## Context

Bugzilla's search subsystem is the most code-intensive domain in the codebase. `Bugzilla::Search` (~111 KB, 3,561 lines) is a monolithic SQL-query-generation engine that accepts structured search criteria — boolean charts, quicksearch strings, and custom field/operator/value triples — and translates them into deeply nested SQL WHERE clauses with dynamic JOIN management across 15+ tables.

The current engine supports:

- **25+ operators** — equality, comparison, substring, regex, word matching, changed-tracking, emptiness, and fulltext
- **30+ field-specific overrides** — user resolution with pronoun substitution, multi-select subselects, comment privacy filtering, idle-time JOINs, and custom field type handling
- **Boolean chart AST** — charts ANDed, rows ANDed, columns ORed, with negation and nesting via `OP`/`CP` delimiters
- **Security filtering** — every query injects `bug_group_map` JOINs and constructs WHERE terms checking group membership, reporter/assignee/CC identity, and accessibility flags
- **Two-phase query optimization** — first fetches matching `bug_id`s with all JOINs, then fetches requested columns for those IDs only

This decision must be made now because the entire `service-search` architecture — directory layout, contracts, query handlers, read model projections, and Docker infrastructure — depends on the underlying query engine technology. Deferring this choice blocks specification and scaffolding for the search service.

### Why PostgreSQL Fulltext Is Insufficient

PostgreSQL's `tsvector`/`tsquery` fulltext search was evaluated and rejected:

1. **No relevance scoring** — Bugzilla's `matches`/`notmatches` operators rely on ranked relevance across summaries and comments. PostgreSQL fulltext ranking is limited and does not support BM25 scoring out of the box.
2. **No faceting or aggregations** — Chart/reports (series data, time-series counts, category breakdowns) require multi-bucket aggregations that are trivial in Elasticsearch but require complex SQL window functions in PostgreSQL.
3. **Boolean chart semantics** — The deeply nested AND/OR/NOT tree with per-field operator dispatch maps to Elasticsearch's `bool.must`/`bool.should`/`bool.must_not` DSL naturally. Expressing the same semantics in SQL requires the exact dynamic JOIN complexity that makes the current Perl codebase unmaintainable.
4. **Cross-field search** — Quicksearch (`#word`, `@user`, `field:value`, priority shortcuts) searches across multiple fields simultaneously. Elasticsearch's multi_match and cross-fields analyzers handle this natively; PostgreSQL requires UNIONed queries or complex OR expressions.

### Why a TypeScript SQL Port Perpetuates Technical Debt

Porting `Bugzilla::Search`'s SQL generation directly to TypeScript (via Knex/TypeORM) would preserve exact query semantics but:

1. Keeps the service coupled to a single relational database schema
2. Requires porting the entire operator-field override dispatch table (30+ overrides)
3. Requires porting dynamic JOIN management (`_combine_joins`, `_extract_then_to`, `_translate_join`)
4. Requires porting `build_subselect()` subquery optimization
5. Does not solve the faceting/aggregation requirement for charts
6. Limits query performance at scale — complex multi-JOIN queries on large bug tables degrade non-linearly

## Decision

We will use **Elasticsearch** as the search engine for `service-search`, deployed as a cluster node in the Docker stack.

### Architecture

```
service-search
  ├── TypeScript query builder modules
  │     ├── quicksearch-parser.ts    — ports Bugzilla::Search::Quicksearch
  │     ├── boolean-chart-ast.ts     — ports Clause/Condition/ClauseGroup tree
  │     ├── operator-map.ts          — maps 25+ operators → Elasticsearch DSL
  │     └── security-filter.ts       — group-based visibility filtering
  ├── Elasticsearch client
  │     └── index mapping for bugs (searchable fields, analyzers)
  ├── Event subscriptions
  │     └── bug.Events.* → near-real-time index updates
  └── Docker service definition
        └── Elasticsearch 8.x node in docker-compose.yml
```

### Key Design Elements

1. **TypeScript query builder** — The boolean chart AST (`Clause`/`Condition`/`ClauseGroup`) is ported as TypeScript modules. Each `Condition(field, operator, value)` is translated to an Elasticsearch filter clause. `Clause` joiners map directly:
   - `AND` → `bool.must`
   - `OR` → `bool.should` (with `minimum_should_match: 1`)
   - Negation → `bool.must_not`

2. **Quicksearch parser** — The quicksearch mini-language (`#word`, `@user`, `field:value`, `P1-P5`, boolean combinators) is ported as a standalone TypeScript parser that emits the same Elasticsearch query DSL as the boolean chart system.

3. **Security filtering** — Bug group visibility is encoded as per-document field-level metadata in the Elasticsearch index. Queries inject group membership filters at query time, equivalent to the current `_standard_where` / `_standard_joins` behavior.

4. **Index sync via events** — `service-search` subscribes to bug domain events (`BugCreated`, `BugUpdated`, `BugResolved`, etc.) and updates the Elasticsearch index near-real-time. This replaces the current `bugs_fulltext` table and the two-phase query optimization.

5. **Saved searches remain CRUD** — Saved searches store the query parameters (not SQL), re-executed against Elasticsearch on demand. This aligns with the clarification recommendation (Q16: simple CRUD for saved searches).

6. **Chart aggregations** — Reporting charts use Elasticsearch aggregations for series data (time-series counts by status, resolution, assignee, etc.), replacing the current `series_data` table and scheduled count jobs.

### Banyan Integration

- **CQRS alignment**: `service-search` is a pure query/projection service. It consumes events from other services and maintains a denormalized Elasticsearch index. It does not own aggregates.
- **Event subscriptions**: Subscribes to `bug.Events.*`, `product.Events.ProductRenamed`, `component.Events.ComponentRenamed` for index updates and saved query field-value propagation.
- **Authorization**: Layer 1 permissions (`search:execute`, `search:saved-search:create`, etc.) on query/command contracts. Layer 2 policies (`SearchVisibilityPolicy` for shared searches, bug security filtering in the query layer).
- **Cross-service scope**: `service-search` maintains its own denormalized read models (Elasticsearch index) projected from bug, user, product, component, attachment, and comment events, rather than making synchronous cross-service queries.

## Consequences

### What Becomes Easier

- **Boolean chart queries** — The AST-to-Elasticsearch mapping is straightforward and eliminates dynamic JOIN management entirely
- **Fulltext search** — BM25 relevance scoring, analyzers, and multi-field search are native to Elasticsearch
- **Faceted search and aggregations** — Chart/report data, dashboard counts, and filter facet values are trivial aggregations
- **Scalability** — Elasticsearch horizontally scales query and indexing load independently from PostgreSQL
- **Index maintenance** — Near-real-time sync from events replaces the `bugs_fulltext` table and scheduled indexing jobs
- **Operator extensibility** — New search operators map to Elasticsearch query types without schema changes

### What Becomes Harder

- **Infrastructure complexity** — An Elasticsearch node must be added to the Docker stack, increasing memory requirements (~2 GB minimum for a development cluster) and operational surface area
- **Index mapping design** — The initial index mapping must carefully define which fields are searchable, which analyzers to use (standard, keyword, n-gram for quicksearch), and how to handle multi-value fields (CC lists, keywords, flags)
- **Eventual consistency of search results** — The Elasticsearch index is updated asynchronously from events; there is a small window where search results may not reflect the latest bug state. This is acceptable for a bug tracker but must be documented
- **Testing** — Integration tests for search require an Elasticsearch instance; the Docker test stack must include it
- **Data migration** — Existing bug data must be bulk-indexed into Elasticsearch during migration, requiring a one-time reindex job

### Trade-offs Accepted

- **Infrastructure dependency** — Elasticsearch is a required service; `service-search` cannot function without it. This is acceptable given that search is a core feature and Elasticsearch is the industry standard for this use case
- **Near-real-time vs. real-time** — Elasticsearch's refresh interval (default 1 second) means search results may be slightly stale. This matches Bugzilla's current behavior where `bugs_fulltext` updates are batched
- **Denormalized data** — The Elasticsearch index duplicates data owned by other services (bug fields, user names, product names). This is inherent to CQRS read models and is the correct tradeoff for query performance

## Alternatives Considered

### Alternative 1: PostgreSQL Fulltext with tsvector

Port the search engine as a TypeScript SQL generator targeting PostgreSQL `tsvector`/`tsquery` indexes.

**Rejected because**: No relevance scoring, no native aggregations/faceting, does not eliminate the dynamic JOIN complexity, and multi-field boolean chart queries remain as complex as the current Perl implementation. Viable only for a minimal search MVP but does not support the full Bugzilla search feature set.

### Alternative 2: TypeScript SQL Port (Knex/TypeORM)

Port `Bugzilla::Search`'s SQL generation directly to TypeScript using a query builder library, preserving the operator dispatch table and dynamic JOIN management.

**Rejected because**: Perpetuates the architectural problems of the current codebase (deep SQL coupling, dynamic JOINs, subselect optimization). Does not address the faceting/aggregation requirement. Considered as a future fallback if Elasticsearch proves operationally burdensome, but not the primary strategy.

### Alternative 3: Hybrid (Simple Searches via PostgreSQL, Advanced via Elasticsearch)

Simple queries (single-field equality, basic lists) go through PostgreSQL read models; advanced boolean charts and fulltext go through Elasticsearch.

**Rejected because**: Introduces two query paths with different consistency models and result formats, increasing code complexity without proportional benefit. The boolean chart AST is the common representation; splitting its execution across two engines adds branching complexity. The current `Search.data()` two-phase optimization is an implementation detail that Elasticsearch eliminates entirely — there is no performance reason to keep the PostgreSQL path.

### Alternative 4: Meilisearch

A lighter-weight search engine with a simpler API than Elasticsearch.

**Rejected because**: Meilisearch lacks the advanced query DSL needed for boolean chart translation (no `bool.must`/`bool.should` composition at the level Elasticsearch provides), has limited aggregation support for charts/reports, and has a smaller ecosystem. Its primary advantage (ease of setup) does not outweigh the feature gap for this use case.
