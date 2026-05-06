# Journey Spec: Dashboard and Personal Workflow

## 1. Overview

- **Journey name**: Dashboard and Personal Workflow
- **Purpose**: The landing page experience — provide users with a personalized overview of their bugs, saved searches, recent activity, and quick access to key workflows. This is the "home base" of the application.
- **Entry points**:
  - Application root URL `/`
  - Logo click in sidebar
  - "Dashboard" nav item in sidebar
- **Target user/actor**: Any authenticated user
- **Expected completion time**: Ongoing (persistent page, users return frequently)
- **Steps count**: 1 page with multiple interactive widgets
- **Flow type**: Dashboard hub (non-linear, widget-based)

## Workflow & Rules Mapping

| Journey step | Workflow activity | Decision rule(s) |
|--------------|-------------------|------------------|
| Dashboard Page (aggregate widget load) | [no clear mapping — manual review needed] | [no clear mapping — manual review needed] |
| Widget: My Assigned Bugs | [no clear mapping — manual review needed] (read-side: `service-bug.Queries.SearchBugs` filtered by assignee) | [no clear mapping — manual review needed] |
| Widget: My Reported Bugs | [no clear mapping — manual review needed] (read-side: `service-bug.Queries.SearchBugs` filtered by reporter) | [no clear mapping — manual review needed] |
| Widget: Saved Searches | [no clear mapping — manual review needed] (read-side: `service-search.Queries.ListSavedSearches` + `service-search.Queries.ExecuteSearch`) | [no clear mapping — manual review needed] |
| Widget: Recent Activity | [no clear mapping — manual review needed] (read-side: `service-bug.Queries.GetBugHistory` / `service-comment.Queries.GetBugComments`; consumed projections from `workflow-bug-lifecycle / index-status-change`, `workflow-bug-lifecycle / notify-bug-created`, `workflow-bug-lifecycle / notify-status-changed`) | [no clear mapping — manual review needed] |
| Widget: Watched Bugs | [no clear mapping — manual review needed] (read-side: `service-bug.Queries.SearchBugs` with watched filter) | [no clear mapping — manual review needed] |

> **Note**: This is a read-side dashboard journey. All steps are query-based widget loads that consume projections built from write-side workflow events. No step triggers a workflow gateway directly. The workflow-bug-lifecycle produces the data that populates bug list widgets and activity feeds (via `index-bug-created`, `index-status-change`, `notify-*` nodes), but the dashboard itself only queries read models and the search index. No decision rules are evaluated during dashboard rendering.

## 2. Flow Diagram

```
[Dashboard]
    │
    ├── My Assigned Bugs widget → [Click bug] → Bug Detail
    │
    ├── My Reported Bugs widget → [Click bug] → Bug Detail
    │
    ├── Saved Searches widget → [Click search] → Search Results
    │                      └── [Manage searches] → Saved Searches Settings
    │
    ├── Watched Bugs widget → [Click bug] → Bug Detail
    │
    └── Recent Activity widget → [Click bug] → Bug Detail
```

## 3. Steps

### Dashboard Page

**Purpose**: Personalized overview with widgets showing assigned bugs, saved searches, and recent activity.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Dashboard                                  [@josh ▾]        │
│                                                             │
│ ┌─ My Assigned Bugs (12) ──────────────── [View all →] ──┐ │
│ │                                                         │ │
│ │ #4521  ● New      Fix crash on launch        P1  2h ago │ │
│ │ #4518  ● Asgn     Memory leak in cache       P2  1d ago │ │
│ │ #4515  ● InProg   Update footer links        P3  3d ago │ │
│ │ #4501  ● Resolved Fix login redirect   FIXED  —  1w ago │ │
│ │                                                         │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─ Saved Searches ────────────────── [Manage searches →] ─┐│
│ │                                                          ││
│ │ 🔍 Firefox open P1s (34)                                ││
│ │ 🔍 My reported bugs (8)                                 ││
│ │ 🔍 Untriaged (156)                                      ││
│ │ 🔍 Overdue deadlines (5)                                ││
│ │                                                          ││
│ └──────────────────────────────────────────────────────────┘│
│                                                             │
│ ┌─ Recent Activity ───────────────────────────────────────┐ │
│ │                                                          │ │
│ │ @sara commented on #4521 — 2 hours ago                  │ │
│ │   "I can reproduce this on macOS 14.1 as well."         │ │
│ │                                                          │ │
│ │ @alex transitioned #4510 to RESOLVED/FIXED — 4h ago     │ │
│ │                                                          │ │
│ │ @josh attached file to #4518 — yesterday                │ │
│ │                                                          │ │
│ │ @sara created #4521 — 2 days ago                        │ │
│ │                                                          │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─ Watched Bugs (3) ──────────────────── [View all →] ──┐  │
│ │                                                         │  │
│ │ #4499  ● InProg  Migrate to new API          @dev3      │  │
│ │ #4480  ● New     Investigate perf regression @dev1      │  │
│ │                                                         │  │
│ └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:

| Widget | Query | Service |
|--------|-------|---------|
| My Assigned Bugs | Query: SearchBugs({ assignee: currentUser, status: open, sort: 'lastModified:desc', limit: 5 }) | service-bug |
| My Reported Bugs | Query: SearchBugs({ reporter: currentUser, sort: 'lastModified:desc', limit: 5 }) | service-bug |
| Saved Searches | Query: GetSavedSearches(userId) | service-search |
| Saved Search Results | Query: SearchBugs(savedSearch.query) per search | service-bug |
| Recent Activity | Query: GetRecentActivity(userId, { limit: 10 }) | service-bug |
| Watched Bugs | Query: GetWatchedBugs(userId) | service-bug |

**State**:
```typescript
interface DashboardState {
  assignedBugs: BugSummary[];
  reportedBugs: BugSummary[];
  savedSearches: SavedSearch[];
  savedSearchCounts: Record<string, number>;
  recentActivity: ActivityEntry[];
  watchedBugs: BugSummary[];
  isLoading: Record<WidgetName, boolean>;
  errors: Record<WidgetName, string | null>;
}
```

**Widget Interactions**:

| Widget | Interaction | Target |
|--------|------------|--------|
| My Assigned Bugs | Click bug row | Bug Detail page |
| My Assigned Bugs | "View all" | Search page with assignee=me, status=open filters |
| Saved Searches | Click search | Search page with saved query |
| Saved Searches | "Manage searches" | Preferences > Saved Searches section |
| Recent Activity | Click activity entry | Bug Detail page |
| Watched Bugs | Click bug row | Bug Detail page |
| Watched Bugs | "View all" | Search page with watched filter |

---

### Widget: My Assigned Bugs

**Purpose**: Show the current user's assigned bugs, ordered by last modified, with quick visual indicators for urgency.

**Behavior**:
- Shows top 5 bugs by default, expandable to 10
- Each row shows: ID, status badge, summary (truncated), priority, last-modified relative time
- Priority color-coded (P1 = red dot, P2 = orange, etc.)
- "View all →" link navigates to filtered search

---

### Widget: Saved Searches

**Purpose**: Quick access to the user's saved searches with live bug counts.

**Behavior**:
- Each saved search shows: name, count of matching bugs (refreshed on dashboard load)
- Click executes the saved search (navigates to search results page)
- "Manage searches" link goes to user preferences
- Shared searches marked with a share icon

---

### Widget: Recent Activity

**Purpose**: Show a timeline of recent changes to bugs the user is involved with (assigned, reported, watching, CC'd).

**Behavior**:
- Shows last 10 activity entries
- Each entry shows: user avatar, action description, relative time
- Activity types: comment added, status changed, field updated, attachment added, bug created
- Click any entry to navigate to the relevant bug

---

### Widget: Watched Bugs

**Purpose**: Show bugs the user is watching (explicitly added to watch list).

**Behavior**:
- Shows bug ID, status, summary, assignee
- "Unwatch" action available inline (small ✕ button)
- "View all" navigates to search page with watched filter

## 4. Journey State Machine

```typescript
type DashboardJourneyState =
  | { type: 'loading' }
  | { type: 'loaded'; data: DashboardState }
  | { type: 'widget-error'; widget: string; error: Error }
  | { type: 'refreshing'; widget?: string }
  | { type: 'navigating'; target: string };
```

## 5. Error Recovery

| Error Scenario | Recovery Strategy |
|---------------|-------------------|
| Widget load failure | Show error state within the widget card: "Failed to load." + "Try Again" button. Other widgets unaffected. |
| Saved search count stale | Show count in muted text. Refresh counts on dashboard load. Manual refresh button available. |
| No assigned bugs | Empty state: "No bugs assigned to you. Browse open bugs or check your saved searches." |
| No saved searches | Empty state: "No saved searches. Run a search and save it for quick access." + "Search" CTA |
| No recent activity | Empty state: "No recent activity to show." |
| Full page load failure | Full-page error: "Failed to load dashboard." + "Try Again" button. |

## 6. Success States

- **Dashboard loaded**: All widgets populate with data
- **Widget refresh**: Individual widgets can be refreshed independently (pull-to-refresh gesture on mobile, refresh icon on desktop)
- **Auto-refresh**: Dashboard auto-refreshes every 5 minutes when the page is active

## 7. Progress Indication

- **Page load**: Skeleton cards for each widget (not spinners)
- **Widget refresh**: Inline spinner within the widget header
- **Saved search counts**: Show "…" while counting, then actual number

## 8. Personalization

The dashboard respects user preferences:
- Widget visibility: Users can show/hide widgets from preferences
- Widget order: Users can reorder widgets via drag-and-drop
- Default sort for "My Assigned Bugs": configurable (priority, last modified, severity)
- Saved search display: configurable (show counts or not)

## Workflow & Rules Cross-References

### Dashboard Page (aggregate widget load)

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  - The dashboard page aggregates data from multiple read-side queries across `service-bug`, `service-search`, and `service-comment`. No single workflow node governs this step. Data consumed is the product of prior write-side events (BugCreated, BugStatusTransitioned, CommentCreated, etc.) projected into read models and the Elasticsearch index.
- **Decision rules**:
  - [no clear mapping — manual review needed]
- **Edge cases formalized**:
  - `Full page load failure` → decision-point branch: `loading → widget-error` — full-page error rendering with "Try Again" recovery
    cite: [UI-only — no business rule]

### Widget: My Assigned Bugs

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  - Read-side query: `service-bug.Queries.SearchBugs({ assignee: currentUser, status: open, sort: 'lastModified:desc', limit: 5 })`. The displayed data originates from `BugListReadModel` projected from `bug.Events.BugCreated`, `bug.Events.BugUpdated`, and `bug.Events.BugStatusTransitioned` events. The search index that supports the "View all" navigation is maintained by `workflow-bug-lifecycle / index-bug-created` and `workflow-bug-lifecycle / index-status-change` nodes.
  - [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `index-bug-created` node, `index-status-change` node]
- **Decision rules**:
  - [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) — Search result security filtering: user sees only bugs where they are reporter, assignee, in CC, or in a visible group
  - [BR-search-014](../decision-rules.md#br-search-014) — Search results security-filtered per user
- **Edge cases formalized**:
  - `Widget load failure` → decision-point branch: `loading → widget-error` for this widget; other widgets unaffected
    cite: [UI-only — no business rule]
  - `No assigned bugs` → decision-point branch: empty state rendering ("No bugs assigned to you. Browse open bugs or check your saved searches.")
    cite: [UI-only — no business rule]

### Widget: My Reported Bugs

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  - Read-side query: `service-bug.Queries.SearchBugs({ reporter: currentUser, sort: 'lastModified:desc', limit: 5 })`. Same read-model lineage as My Assigned Bugs — data projected from bug lifecycle events.
- **Decision rules**:
  - [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) — Search result security filtering applies
  - [BR-search-014](../decision-rules.md#br-search-014) — Search results security-filtered per user
- **Edge cases formalized**:
  - `Widget load failure` → decision-point branch: `loading → widget-error` for this widget; other widgets unaffected
    cite: [UI-only — no business rule]
  - `No reported bugs` → decision-point branch: empty state rendering (analogous to "No assigned bugs" pattern)
    cite: [UI-only — no business rule]

### Widget: Saved Searches

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  - Read-side queries: `service-search.Queries.ListSavedSearches(userId)` to load saved search definitions, then `service-search.Queries.ExecuteSearch(savedSearch.query)` per search to compute counts. The Elasticsearch index backing ExecuteSearch is populated by `workflow-bug-lifecycle / index-bug-created` and `workflow-bug-lifecycle / index-status-change` nodes.
  - [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `index-bug-created` node, `index-status-change` node]
- **Decision rules**:
  - [BR-search-014](../decision-rules.md#br-search-014) — Search result security filtering (Elasticsearch bug security filter per user group membership)
  - [BR-search-POLICY-SearchVisibilityPolicy](../decision-rules.md#br-search-policy-searchvisibilitypolicy) — User must be owner or member of shared group to access a saved search
  - [BR-search-008](../decision-rules.md#br-search-008) — User may link a saved search to footer only if owner or member of shared group
- **Edge cases formalized**:
  - `Widget load failure` → decision-point branch: `loading → widget-error` for this widget; other widgets unaffected
    cite: [UI-only — no business rule]
  - `Saved search count stale` → decision-point branch: display count in muted text; refresh on dashboard load; manual refresh button available
    cite: [UI-only — no business rule]
  - `No saved searches` → decision-point branch: empty state rendering ("No saved searches. Run a search and save it for quick access." + "Search" CTA)
    cite: [UI-only — no business rule]

### Widget: Recent Activity

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  - Read-side query: `service-bug.Queries.GetBugHistory` / `service-bug.Queries.GetRecentActivity(userId, { limit: 10 })`. Activity entries are projected from `bug.Events.BugCreated`, `bug.Events.BugUpdated`, `bug.Events.BugStatusTransitioned` (via `BugActivityReadModel` in service-bug) and `comment.Events.CommentCreated` (via `CommentListReadModel` in service-comment). The underlying data traces back to multiple workflow-bug-lifecycle nodes:
    - `workflow-bug-lifecycle / create-bug-aggregate` → BugCreated → activity entry "bug created"
      [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `create-bug-aggregate` node]
    - `workflow-bug-lifecycle / apply-status-transition` → BugStatusTransitioned → activity entry "status changed"
      [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `apply-status-transition` node]
    - `workflow-bug-lifecycle / create-comment` → CommentCreated → activity entry "comment added"
      [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `create-comment` node]
    - `workflow-bug-lifecycle / update-bug` → BugUpdated → activity entry "field updated"
      [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `update-bug` node]
    - `workflow-bug-lifecycle / create-attachment` → AttachmentCreated → activity entry "attachment added"
      [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `create-attachment` node]
- **Decision rules**:
  - [BR-bug-014](../decision-rules.md#br-bug-014) — Time-tracking fields require dedicated `bugs:edit_timetracking` permission
  - [BR-notification-007](../decision-rules.md#br-notification-007) — Time-tracking fields stripped from notifications/diff for users lacking `bugs:timetracker` permission
  - [BR-comment-007](../decision-rules.md#br-comment-007) — Private comments hidden from non-insiders unless caller has `comments:view:private`
  - [BR-comment-POLICY-CanSeePrivateCommentsPolicy](../decision-rules.md#br-comment-policy-canseeprivatecommentspolicy) — Filters private comments from query results
- **Edge cases formalized**:
  - `Widget load failure` → decision-point branch: `loading → widget-error` for this widget; other widgets unaffected
    cite: [UI-only — no business rule]
  - `No recent activity` → decision-point branch: empty state rendering ("No recent activity to show.")
    cite: [UI-only — no business rule]

### Widget: Watched Bugs

- **Workflow gateway / activity**: [no clear mapping — manual review needed]
  - Read-side query: `service-bug.Queries.SearchBugs` or `GetWatchedBugs(userId)` filtered by watch list. Displayed bug data is projected from bug lifecycle events and indexed via `workflow-bug-lifecycle / index-bug-created` and `workflow-bug-lifecycle / index-status-change` nodes.
  - [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml — `index-bug-created` node, `index-status-change` node]
- **Decision rules**:
  - [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) — Search result security filtering applies
  - [BR-notification-PERM-GetNotificationPreferences](../decision-rules.md#br-notification-perm-getnotificationpreferences) — `notifications:preferences:read` permission required to access watcher list
  - [BR-search-014](../decision-rules.md#br-search-014) — Search results security-filtered per user
- **Edge cases formalized**:
  - `Widget load failure` → decision-point branch: `loading → widget-error` for this widget; other widgets unaffected
    cite: [UI-only — no business rule]
  - `No watched bugs` → decision-point branch: empty state rendering (analogous to other empty-state patterns)
    cite: [UI-only — no business rule]
