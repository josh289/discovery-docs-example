# Journey Spec: Search and Filter Bugs

## 1. Overview

- **Journey name**: Search and Filter Bugs
- **Purpose**: Enable users to find bugs using quick search (by ID or keyword), structured filters, and advanced boolean chart queries. Results can be saved as named searches for later reuse.
- **Entry points**:
  - Search bar in header (quick search)
  - "Search" nav item in sidebar → search page
  - `/search` URL
  - Keyboard shortcut `/` (focus search bar)
- **Target user/actor**: Any authenticated user
- **Expected completion time**: 10 seconds (quick search) to 3 minutes (advanced search)
- **Steps count**: 1 page (multi-mode: quick, filter, advanced)
- **Flow type**: Single-page with modes

## Workflow & Rules Mapping

| Journey step | Workflow activity | Decision rule(s) |
|--------------|-------------------|------------------|
| Mode 1: Quick Search (Header Bar) | [no clear mapping — manual review needed] | [BR-search-014](../decision-rules.md#br-search-014), [BR-search-PERM-ExecuteSearch](../decision-rules.md#br-search-perm-executesearch) |
| Mode 2: Filtered Bug List | workflow-bug-lifecycle / validate-workflow-status (status filter dropdown references workflow statuses) | [BR-search-014](../decision-rules.md#br-search-014), [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy), [BR-search-PERM-ExecuteSearch](../decision-rules.md#br-search-perm-executesearch) |
| Mode 3: Advanced Search (Boolean Charts) | [no clear mapping — manual review needed] | [BR-search-014](../decision-rules.md#br-search-014), [BR-search-PERM-ExecuteSearch](../decision-rules.md#br-search-perm-executesearch) |
| Save Search Dialog | [no clear mapping — manual review needed] | [BR-search-002](../decision-rules.md#br-search-002), [BR-search-006](../decision-rules.md#br-search-006), [BR-search-009](../decision-rules.md#br-search-009), [BR-search-PERM-CreateSavedSearch](../decision-rules.md#br-search-perm-createsavedsearch) |

## 2. Flow Diagram

```
[Open Search] → [Quick Search Bar]
                    │
                    ├── Type bug ID → [Navigate to Bug Detail]
                    │
                    └── Type keyword → [Show Results with Filter Bar]
                                            │
                                            ├── Toggle filters → [Refine results]
                                            │
                                            ├── "Advanced…" → [Boolean Chart Builder]
                                            │                       │
                                            │                       └── [Search] → [Results]
                                            │
                                            └── [Save Search…] → [Name Dialog] → [Saved]
```

## 3. Steps

### Mode 1: Quick Search (Header Bar)

**Purpose**: Fast access to a specific bug by ID or keyword search.

**Wireframe**:
```
┌───────────────────────────────────────────────────────────┐
│ ┌──────────────────────────────────────────────┐ [Search]  │
│ │ 🔍 Search bugs by ID, summary, or keyword…  │           │
│ └──────────────────────────────────────────────┘           │
└───────────────────────────────────────────────────────────┘
```

**Behavior**:
- If input matches `#\d+` (bug ID pattern): navigate directly to bug detail
- Otherwise: navigate to search page with query pre-filled, show results

**Data Requirements**:
- Query: SearchBugs({ query, limit: 25 }) (service-bug)

**Validation**: None client-side. All filtering is server-side.

**State**:
```typescript
interface QuickSearchState {
  query: string;
  isSearching: boolean;
  suggestions: BugSummary[];
}
```

---

### Mode 2: Filtered Bug List

**Purpose**: Browse bugs with structured filter dropdowns. This is the default search page experience.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Search                                                      │
│                                                             │
│ ┌─ Quick Search ─────────────────────────────────────────┐  │
│ │ 🔍 Fix crash on launch                               │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                             │
│ Filter Bar                                                  │
│ [Status ▼] [Priority ▼] [Product ▼] [Assignee ▼]           │
│ [Component ▼] [Severity ▼] [Reporter ▼] [Advanced…]        │
│                                                             │
│ 47 results    [Sort: Last Modified ▼]  [View: Table | List] │
│ ──────────────────────────────────────────────────────────  │
│ #ID     Status     Summary                    Assignee  Sev │
│ #4521   ● New      Fix crash on launch        @josh    P1  │
│ #4518   ● Asgn     Memory leak in cache       @sara    P2  │
│ #4515   ● InProg   Update footer links        @alex    P3  │
│ ...                                                         │
│ ──────────────────────────────────────────────────────────  │
│ Showing 1–25 of 47         [Prev] [1] [2] [Next]           │
│                                                             │
│ [Save Search…]                                              │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: SearchBugs({ query, filters, sort, pagination }) (service-bug)
- Query: GetLegalValues('status') (service-bug) — filter dropdown options
- Query: GetProducts (service-product) — product filter
- Query: SearchUsers (service-user) — assignee/reporter filter typeahead

**Validation**:
- All filter values validated server-side
- Results filtered by user's group visibility (CanSeeBugPolicy)

**State**:
```typescript
interface FilteredSearchState {
  query: string;
  filters: {
    status: string[];
    priority: string[];
    product: string[];
    component: string[];
    assignee: string[];
    reporter: string[];
    severity: string[];
  };
  sort: { field: string; direction: 'asc' | 'desc' };
  pagination: { offset: number; limit: number };
  results: BugSummary[];
  totalResults: number;
  isLoading: boolean;
}
```

---

### Mode 3: Advanced Search (Boolean Charts)

**Purpose**: Build complex boolean queries with multiple criteria rows connected by AND/OR logic. Mirrors Bugzilla's classic boolean chart system.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Advanced Search                                              │
│                                                             │
│ ┌─ Search Criteria ────────────────────────────────────────┐│
│ │                                                          ││
│ │ [Summary ▼] [contains ▼] [crash on launch______] [+] [-] ││
│ │   AND                                                     ││
│ │ [Status ▼] [is ▼]      [NEW, ASSIGNED ▼]      [+] [-]   ││
│ │   AND                                                     ││
│ │ [Product ▼] [is ▼]     [Firefox ▼]             [+] [-]   ││
│ │                                                          ││
│ │ [+ Add criteria row]                                     ││
│ │                                                          ││
│ │ ── People ──                                              ││
│ │ Assignee: [@josh ▼]   Reporter: [@sara ▼]                ││
│ │ CC contains: [________]                                   ││
│ │                                                          ││
│ │ ── Dates ──                                               ││
│ │ Created: [from] [2024-01-01] [to] [2024-06-01]           ││
│ │ Modified: [from] [________] [to] [________]               ││
│ └──────────────────────────────────────────────────────────┘│
│                                                             │
│ [Search]  [Save Search…]  [Clear All]                       │
│                                                             │
│ ── Results (47) ──────────────────────────────────────────  │
│ (same results table as filtered view)                       │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: SearchBugs({ booleanChart, people, dates, pagination }) (service-bug) — advanced search DSL
- Query: GetBugFields(productId?) (service-bug) — available fields for criteria rows

**Validation**:
- At least one criteria row must be non-empty
- Date values must be valid YYYY-MM-DD format
- User references must be valid user IDs
- Field/operator combinations must be valid (server-side)

**State**:
```typescript
interface AdvancedSearchState {
  criteriaRows: Array<{
    id: string;
    field: string;
    operator: string;
    value: string | string[];
  }>;
  people: {
    assignee: string | null;
    reporter: string | null;
    ccContains: string | null;
  };
  dates: {
    createdFrom: string | null;
    createdTo: string | null;
    modifiedFrom: string | null;
    modifiedTo: string | null;
  };
  connector: 'AND' | 'OR';
  results: BugSummary[];
  totalResults: number;
  isLoading: boolean;
}
```

---

### Save Search Dialog

**Purpose**: Name and save the current search for reuse in the dashboard.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Save Search                                          [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Search Name *                                               │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Firefox open P1s                                         ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ ☐ Share with team (visible to all users)                    │
│                                                             │
│                                             [Cancel] [Save] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: SaveSearch (service-search) — save with name and query
- Command: ShareSearch (service-search) — if shared

**Validation**:
- Search name is required, must be unique for this user
- Search must have at least one filter/criteria

## 4. Journey State Machine

```typescript
type SearchJourneyState =
  | { type: 'quick-search'; query: string }
  | { type: 'filtered-results'; filters: FilterState; results: BugSummary[] }
  | { type: 'advanced-search'; criteria: AdvancedCriteria }
  | { type: 'loading' }
  | { type: 'results-loaded'; results: BugSummary[]; total: number }
  | { type: 'no-results'; query: string }
  | { type: 'error'; error: Error }
  | { type: 'saving-search'; searchName: string }
  | { type: 'search-saved'; searchId: string };
```

## 5. Error Recovery

| Error Scenario | Recovery Strategy |
|---------------|-------------------|
| No results found | Empty state: "No bugs match the current filters. Try adjusting your search criteria." + "File a Bug" CTA. |
| Search query too broad (server timeout) | Error: "Your search returned too many results. Add more specific criteria." |
| Invalid filter combination | Inline error on the affected row. |
| Save search name conflict | Inline error: "You already have a saved search with this name." |
| Network failure | Error toast with "Try Again" button. |
| Bug ID not found (quick search) | Show: "Bug #{id} not found." Stay on search page. |

## 6. Success States

- **Search results loaded**: Results table populated with count: "Showing 1–25 of 47 bugs"
- **Bug ID found (quick search)**: Navigate to bug detail page
- **Search saved**: Toast: "Search '{name}' saved". Saved search appears in dashboard.
- **Screen reader**: "{count} bugs found" on results load

## 7. Progress Indication

- Search bar: inline spinner during quick search
- Results area: skeleton rows during search
- Filter dropdowns: no loading (options pre-loaded or cached)
- Advanced criteria: debounced search (300ms after last keystroke)

## 8. Sort & Pagination

- Sort options: Last Modified, Bug ID, Summary, Severity, Priority, Assignee
- Sort toggle: column header click cycles asc → desc → none
- Pagination: 25/50/100 per page, configurable
- URL-synced: search state persisted in URL query params for sharing/bookmarking

## Workflow & Rules Cross-References

### Mode 1: Quick Search (Header Bar)

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  — Quick Search is a read-side query (`Quicksearch` or `SearchBugs` against service-search/service-bug) with no corresponding workflow gateway. The search index is populated asynchronously by `index-bug-created` and `index-status-change` nodes in workflow-bug-lifecycle.
- **Decision rules**:
  - [BR-search-014](../decision-rules.md#br-search-014) — Search results are security-filtered per user (reporter, CC, assignee, visible group member)
  - [BR-search-PERM-ExecuteSearch](../decision-rules.md#br-search-perm-executesearch) — `search:execute` JWT permission required
  - [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) — Underlying bug visibility policy (group restrictions, reporterAccessible, cclistAccessible)
  - [unciteable: TODO] (searched: BR-search-015 referenced in slice but not present in master decision-rules.md; ES 1-second refresh staleness is an ADR-004/infrastructure concern, not a master-table rule)
- **Edge cases formalized**:
  - "Bug ID not found (quick search)" → decision-point branch: `Quicksearch → no-results: "Bug #{id} not found."` cite: [BR-search-014](../decision-rules.md#br-search-014) (security filter may suppress bug from results)
  - "Network failure" → decision-point branch: `Quicksearch → error: toast with "Try Again"` cite: [UI-only — no business rule]

### Mode 2: Filtered Bug List

- **Workflow gateway / activity**: `workflow-bug-lifecycle / validate-workflow-status` (serviceTask) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:37]
  — The status filter dropdown presents the statuses defined by the StatusWorkflowConfig (governed by `validate-workflow-status` in the creation flow). Filtered results are served by `ExecuteSearch` on service-search, with the search index populated by `index-bug-created` [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:74] and `index-status-change` [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:263].
- **Decision rules**:
  - [BR-search-014](../decision-rules.md#br-search-014) — Search results are security-filtered per user (group membership, reporter/CC/assignee checks)
  - [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) — Bug visibility policy (group restrictions, reporterAccessible, cclistAccessible)
  - [BR-search-PERM-ExecuteSearch](../decision-rules.md#br-search-perm-executesearch) — `search:execute` JWT permission required
  - [unciteable: TODO] (searched: BR-search-015 in slice but absent from master; staleness is ADR-004 / infrastructure-level, not a formalized master-table rule)
- **Edge cases formalized**:
  - "No results found" → decision-point branch: `ExecuteSearch → no-results: "No bugs match the current filters. Try adjusting your search criteria." + "File a Bug" CTA` cite: [BR-search-014](../decision-rules.md#br-search-014)
  - "Search query too broad (server timeout)" → decision-point branch: `ExecuteSearch → error: "Your search returned too many results. Add more specific criteria."` cite: [UI-only — no business rule]
  - "Invalid filter combination" → decision-point branch: `ExecuteSearch → reject(invalid_filter): inline error on affected row` cite: [BR-search-006](../decision-rules.md#br-search-006) (query canonicalization implies invalid params are rejected/stripped)
  - "Network failure" → decision-point branch: `ExecuteSearch → error: toast with "Try Again"` cite: [UI-only — no business rule]

### Mode 3: Advanced Search (Boolean Charts)

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  — Advanced Search is a read-side query (`ExecuteSearch` with boolean chart DSL against service-search). No workflow gateway governs search query construction. The Elasticsearch index that backs results is asynchronously maintained by service-search event subscriptions.
- **Decision rules**:
  - [BR-search-014](../decision-rules.md#br-search-014) — Security filtering applied to all search results
  - [BR-search-PERM-ExecuteSearch](../decision-rules.md#br-search-perm-executesearch) — `search:execute` JWT permission required
  - [unciteable: TODO] (searched: BR-search-015 in slice but absent from master; staleness is ADR-004 / infrastructure-level, not a formalized master-table rule)
- **Edge cases formalized**:
  - "No results found" → decision-point branch: `ExecuteSearch → no-results: "No bugs match the current filters."` cite: [BR-search-014](../decision-rules.md#br-search-014)
  - "Search query too broad (server timeout)" → decision-point branch: `ExecuteSearch → error: "Your search returned too many results."` cite: [UI-only — no business rule]
  - "Invalid filter combination" → decision-point branch: `ExecuteSearch → reject(invalid_filter): inline error on affected row` cite: [BR-search-006](../decision-rules.md#br-search-006)
  - "Network failure" → decision-point branch: `ExecuteSearch → error: toast with "Try Again"` cite: [UI-only — no business rule]

### Save Search Dialog

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  — Save Search invokes `CreateSavedSearch` (service-search) which is a CRUD command with no corresponding workflow gateway. SavedSearchAggregate is not event-sourced (per ADR-016); it has no discrete state machine beyond simple CRUD transitions.
- **Decision rules**:
  - [unciteable: TODO] (searched: BR-search-001 referenced in slice but not present in master decision-rules.md; saved-search name format/length is governed by SERVICE_SPEC#business-rule-1 only)
  - [BR-search-002](../decision-rules.md#br-search-002) — Saved search name must be unique among all saved searches owned by the same user
  - [BR-search-006](../decision-rules.md#br-search-006) — Query field is cleaned before storage (empty params stripped, keys sorted)
  - [BR-search-009](../decision-rules.md#br-search-009) — Recent searches capped at SAVE_NUM_SEARCHES; oldest auto-deleted on overflow
  - [BR-search-PERM-CreateSavedSearch](../decision-rules.md#br-search-perm-createsavedsearch) — `search:saved-search:create` permission required
- **Edge cases formalized**:
  - "Save search name conflict" → decision-point branch: `CreateSavedSearch → reject(duplicate_name): "You already have a saved search with this name."` cite: [BR-search-002](../decision-rules.md#br-search-002)
  - "Network failure" → decision-point branch: `CreateSavedSearch → error: toast with "Try Again"` cite: [UI-only — no business rule]
