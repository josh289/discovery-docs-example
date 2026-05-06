# Journey Spec: Bug Detail Review

## 1. Overview

- **Journey name**: Bug Detail Review
- **Purpose**: The primary daily workflow for developers, QA, and project managers — viewing a bug's full details, reading the comment thread, adding comments, editing fields in-place, managing attachments, viewing activity history, and interacting with dependencies and flags.
- **Entry points**:
  - Click bug row in any bug list
  - Click bug ID link anywhere in the application
  - Direct URL: `/bugs/{bugId}`
  - Search result click
  - Notification link (email, in-app)
- **Target user/actor**: Any authenticated user with `bugs:update` or read access to the bug
- **Expected completion time**: 1–10 minutes (read + interact)
- **Steps count**: 1 page with tabbed sub-sections (non-linear, free navigation)
- **Flow type**: Dashboard/hub (non-linear, user navigates freely between tabs)

## Workflow & Rules Mapping

| Journey step | Workflow activity | Decision rule(s) |
|--------------|-------------------|------------------|
| Tab: Details (Default) — Load bug | [no clear mapping — manual review needed] (read-only query: GetBug) | [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) |
| Tab: Details (Default) — In-place field edit | workflow-bug-lifecycle / update-bug-start → update-bug → notify-bug-updated | [BR-bug-015](../decision-rules.md#br-bug-015), [BR-bug-016](../decision-rules.md#br-bug-016), [BR-bug-POLICY-CanChangeFieldPolicy](../decision-rules.md#br-bug-policy-canchangefieldpolicy) |
| Tab: Details (Default) — Status transition | workflow-bug-lifecycle / status-transition-start → validate-transition → gateway-transition-type → apply-status-transition | [BR-bug-003](../decision-rules.md#br-bug-003), [BR-bug-004](../decision-rules.md#br-bug-004), [BR-bug-008](../decision-rules.md#br-bug-008), [BR-bug-018](../decision-rules.md#br-bug-018), [BR-bug-022](../decision-rules.md#br-bug-022), [BR-bug-POLICY-ValidStatusTransitionPolicy](../decision-rules.md#br-bug-policy-validstatustransitionpolicy), [BR-bug-POLICY-NoOpenBlockersPolicy](../decision-rules.md#br-bug-policy-noopenblockerspolicy) |
| Tab: Details (Default) — Assignee change | workflow-bug-lifecycle / update-bug | [BR-bug-POLICY-CanChangeFieldPolicy](../decision-rules.md#br-bug-policy-canchangefieldpolicy), [BR-bug-NOTIF-BugAssigned](../decision-rules.md#br-bug-notif-bugassigned) |
| Tab: Details (Default) — CC list change | workflow-bug-lifecycle / update-bug | [BR-bug-POLICY-CanChangeFieldPolicy](../decision-rules.md#br-bug-policy-canchangefieldpolicy) |
| Tab: Details (Default) — Time tracking edit | workflow-bug-lifecycle / update-bug | [BR-bug-014](../decision-rules.md#br-bug-014), [BR-bug-PERM-UpdateTimetracking](../decision-rules.md#br-bug-perm-updatetimetracking) |
| Tab: Activity | [no clear mapping — manual review needed] (read-only query: GetBugHistory) | [no clear mapping — manual review needed] |
| Tab: Attachments — View list | [no clear mapping — manual review needed] (read-only query: GetBugAttachments) | [BR-attachment-POLICY-CanEditPrivateAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditprivateattachmentpolicy) |
| Tab: Attachments — Upload | workflow-bug-lifecycle / attachment-added-start → create-attachment → notify-attachment-created → create-attachment-system-comment | [BR-attachment-001](../decision-rules.md#br-attachment-001), [BR-attachment-002](../decision-rules.md#br-attachment-002), [BR-attachment-003](../decision-rules.md#br-attachment-003), [BR-attachment-005](../decision-rules.md#br-attachment-005), [BR-attachment-PERM-CreateAttachment](../decision-rules.md#br-attachment-perm-createattachment) |
| Tab: Attachments — Edit metadata | workflow-bug-lifecycle / create-attachment (UpdateAttachmentMetadata) | [BR-attachment-POLICY-CanEditAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditattachmentpolicy), [BR-attachment-PERM-UpdateAttachmentMetadata](../decision-rules.md#br-attachment-perm-updateattachmentmetadata) |
| Tab: Attachments — Delete | workflow-bug-lifecycle / create-attachment (DeleteAttachment) | [BR-attachment-008](../decision-rules.md#br-attachment-008), [BR-attachment-009](../decision-rules.md#br-attachment-009), [BR-attachment-PERM-DeleteAttachment](../decision-rules.md#br-attachment-perm-deleteattachment) |
| Tab: Dependencies — View | [no clear mapping — manual review needed] (read-only query: GetBug) | [no clear mapping — manual review needed] |
| Tab: Dependencies — Add dependency | workflow-bug-lifecycle / dependency-add-start → add-dependency → notify-dependency-added | [BR-bug-006](../decision-rules.md#br-bug-006), [BR-bug-007](../decision-rules.md#br-bug-007), [BR-bug-POLICY-DependencyLoopPolicy](../decision-rules.md#br-bug-policy-dependencylooppolicy), [BR-bug-POLICY-BugVisibilityPolicy](../decision-rules.md#br-bug-policy-bugvisibilitypolicy) |
| Tab: Dependencies — Remove dependency | workflow-bug-lifecycle / add-dependency (RemoveBugDependency) | [BR-bug-PERM-UpdateBug](../decision-rules.md#br-bug-perm-updatebug) |
| Tab: Flags — View | [no clear mapping — manual review needed] (read-only queries: GetBugFlags, GetFlagTypes) | [no clear mapping — manual review needed] |
| Tab: Flags — Set / request flag | workflow-bug-lifecycle / create-attachment (SetBugFlag) | [BR-attachment-010](../decision-rules.md#br-attachment-010), [BR-attachment-011](../decision-rules.md#br-attachment-011), [BR-attachment-013](../decision-rules.md#br-attachment-013), [BR-attachment-015](../decision-rules.md#br-attachment-015), [BR-attachment-POLICY-FlagTypeApplicabilityPolicy](../decision-rules.md#br-attachment-policy-flagtypeapplicabilitypolicy), [BR-attachment-POLICY-CanSetFlagPolicy](../decision-rules.md#br-attachment-policy-cansetflagpolicy) |
| Tab: Flags — Clear flag | workflow-bug-lifecycle / create-attachment (ClearBugFlag) | [BR-attachment-015](../decision-rules.md#br-attachment-015), [BR-attachment-POLICY-CanSetFlagPolicy](../decision-rules.md#br-attachment-policy-cansetflagpolicy) |
| Comment Thread — View comments | [no clear mapping — manual review needed] (read-only query: GetBugComments) | [BR-comment-007](../decision-rules.md#br-comment-007), [BR-comment-POLICY-CanSeePrivateCommentsPolicy](../decision-rules.md#br-comment-policy-canseeprivatecommentspolicy) |
| Comment Thread — Add comment | workflow-bug-lifecycle / comment-added-start → create-comment → notify-comment-created → project-comment-work-time | [BR-comment-003](../decision-rules.md#br-comment-003), [BR-comment-006](../decision-rules.md#br-comment-006), [BR-comment-014](../decision-rules.md#br-comment-014), [BR-comment-015](../decision-rules.md#br-comment-015), [BR-comment-POLICY-CanCommentOnBugPolicy](../decision-rules.md#br-comment-policy-cancommentonbugpolicy) |

## 2. Flow Diagram

```
[Navigate to Bug] → [Load Bug Detail]
                        │
                        ├── [Tab: Details] → In-place field editing
                        │
                        ├── [Tab: Activity] → Activity timeline (read-only)
                        │
                        ├── [Tab: Attachments] → Upload/download/manage attachments
                        │
                        ├── [Tab: Dependencies] → Add/remove dependency links
                        │
                        ├── [Tab: Flags] → Request/set/clear flags
                        │
                        └── [Comment Thread] (always visible) → Add comment
```

## 3. Steps

### Tab: Details (Default)

**Purpose**: View and edit all bug fields. In-place editing for mutable fields.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ ← Back   Products > Firefox > General                       │
│                                                             │
│ Bug #4521  ● New              [Edit] [Transition ▼]        │
│ Fix crash on launch during session restore                  │
├──────────────────────────────────┬──────────────────────────┤
│                                  │ Metadata Sidebar         │
│  [Details] [Activity] [Att.]     │                          │
│  [Dep.] [Flags]                  │ Product:    Firefox  [✎] │
│                                  │ Component:  General  [✎] │
│  Description:                    │ Version:    120.0    [✎] │
│  When launching Firefox          │ Severity:   ● normal [✎] │
│  with session restore, a         │ Priority:   P3       [✎] │
│  crash occurs in the tab         │ Platform:   All      [✎] │
│  restoration code…               │ OS:         All      [✎] │
│                                  │ Assignee:   @josh    [✎] │
│  ── Comment Thread (5) ──────── │ Reporter:   @sara       │
│                                  │ QA:         @alex    [✎] │
│  @sara 2 hours ago              │ Milestone:  120.1     [✎] │
│  I can reproduce this on        │ Deadline:   2024-01-15[✎]│
│  macOS 14.1 as well.            │                          │
│                                  │ Depends on: #4510    [✎] │
│  @josh 1 hour ago               │ Blocks:     #4530    [✎] │
│  Stack trace attached.          │ CC: @dev1, @dev2     [✎] │
│  Looks like a null pointer.     │ Keywords: crash, reg  [✎] │
│                                  │ Whiteboard: [sprint-12]  │
│                                  │ URL: https://…           │
│                                  │ See Also: #4499          │
│                                  │                          │
│  ┌─ Add Comment ──────────────┐ │ Flags:                    │
│  │                            │ │  needinfo? @reviewer   [✎] │
│  │ [Markdown supported]       │ │                          │
│  │                            │ │ ── Time Tracking ──      │
│  │ [ ] Private comment        │ │ Estimated: 8h         [✎] │
│  │ [Hours worked: ___]        │ │ Remaining: 5h         [✎] │
│  │                            │ │ Deadline:  2024-01-15 [✎] │
│  │      [Add Comment]         │ │                          │
│  └────────────────────────────┘ │                          │
└──────────────────────────────────┴──────────────────────────┘
```

**Data Requirements**:
- Query: GetBug(bugId) (service-bug) — full bug detail
- Query: GetBugComments(bugId) (service-comment) — comment thread
- Query: GetBugHistory(bugId) (service-bug) — for activity tab
- Query: GetBugUserLastVisit(bugId) (service-bug) — mark "new since last visit"

**In-place Field Editing**:
Each metadata field with a `[✎]` icon can be clicked to enter edit mode. The field becomes an input/select inline. On blur or Enter, the change is submitted.

- Command: UpdateBug (service-bug) — for most field changes
- Command: AssignBug (service-bug) — for assignee changes
- Command: AddCc / RemoveCc (service-bug) — for CC list changes
- Command: AddBugDependency / RemoveBugDependency (service-bug) — for dependency changes
- Command: UpdateTimetracking (service-bug) — for time fields (requires `bugs:edit_timetracking`)

**Validation**:
- Optimistic concurrency: `expectedVersion` must match current version. On conflict, show error and refresh.
- Field-specific validation as per service-bug business rules.

**State**:
```typescript
interface BugDetailState {
  bug: BugDetail | null;
  comments: Comment[];
  isLoading: boolean;
  error: string | null;
  editingField: string | null;
  editValue: any;
  isSubmittingEdit: boolean;
  lastVisit: Date | null;
}
```

---

### Tab: Activity

**Purpose**: View the full chronological activity timeline for this bug. Read-only.

**Wireframe**:
```
┌───────────────────────────────────────────────────────────┐
│ [Details] [Activity (23)] [Attachments (2)] [Dep.] [Flags]│
│                                                           │
│ Activity Log                                              │
│                                                           │
│ ● @sara — 2 hours ago                                    │
│   changed Status from UNCONFIRMED to NEW                  │
│                                                           │
│ ● @josh — 1 hour ago                                     │
│   changed Assignee from — to @josh                       │
│   changed Priority from P3 to P1                         │
│                                                           │
│ ● @admin — 3 hours ago                                   │
│   added group restriction: firefox-security               │
│                                                           │
│ ● @sara — 4 hours ago                                    │
│   created the bug                                        │
│                                                           │
│ ── Show older activity ──                                 │
└───────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetBugHistory(bugId, { dateRange? }) (service-bug)

**Validation**: None (read-only). Time-tracking entries hidden for users without `bugs:edit_timetracking`.

---

### Tab: Attachments

**Purpose**: View, upload, and manage file attachments on the bug.

**Wireframe**:
```
┌───────────────────────────────────────────────────────────┐
│ [Details] [Activity] [Attachments (2)] [Dep.] [Flags]     │
│                                                           │
│ ┌─ Upload Attachment ─────────────────────────────────┐   │
│ │ [Choose File]  Description: [_____________________] │   │
│ │ ☐ Is patch    ☐ Private                             │   │
│ │                                    [Upload Attachment]│   │
│ └─────────────────────────────────────────────────────┘   │
│                                                           │
│ 📎 stacktrace.log (12 KB) — @josh, 1 hour ago            │
│    text/plain  |  Not a patch  |  [View] [Download] [🗑]  │
│                                                           │
│ 📎 screenshot.png (340 KB) — @sara, 4 hours ago          │
│    image/png   |  Not a patch  |  [View] [Download] [🗑]  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetAttachments(bugId) (service-attachment)
- Command: CreateAttachment (service-attachment) — upload
- Command: UpdateAttachment (service-attachment) — edit metadata
- Command: DeleteAttachment (service-attachment) — remove

---

### Tab: Dependencies

**Purpose**: View and manage the dependency graph (depends on / blocks).

**Wireframe**:
```
┌───────────────────────────────────────────────────────────┐
│ [Details] [Activity] [Attachments] [Dependencies] [Flags] │
│                                                           │
│ ┌─ Depends On ────────────────────────────────────────┐   │
│ │ ↓ #4510 ● In Progress  Memory leak in cache         │   │
│ │   [Remove]                                           │   │
│ │                                                      │   │
│ │ [+ Add dependency]                                   │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                           │
│ ┌─ Blocks ────────────────────────────────────────────┐   │
│ │ ↑ #4530 ● New  Update footer links                  │   │
│ │   [Remove]                                           │   │
│ └──────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetBug(bugId) — includes dependency references
- Command: AddBugDependency (service-bug) — add dependency
- Command: RemoveBugDependency (service-bug) — remove dependency

**Validation**:
- No self-reference (bugId ≠ dependsOnBugId)
- No circular dependencies (server-side check)
- Target bug must exist and be visible

---

### Tab: Flags

**Purpose**: View, request, set, and clear flags on the bug.

**Wireframe**:
```
┌───────────────────────────────────────────────────────────┐
│ [Details] [Activity] [Attachments] [Dependencies] [Flags] │
│                                                           │
│ Flags on this bug:                                        │
│                                                           │
│ needinfo  @reviewer  ?  [Set +] [Set -] [Clear]           │
│ review    —          —  [Request ?] [Clear]                │
│ approval  @pm        +  [Clear]                            │
│                                                           │
│ [+ Request New Flag]                                      │
└───────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetFlags(bugId) (service-attachment) — flag instances
- Query: GetFlagTypes(productId, componentId) (service-attachment) — available flag types
- Command: SetFlag (service-attachment) — set/request flag
- Command: ClearFlag (service-attachment) — remove flag

---

### Comment Thread (Always Visible)

**Purpose**: Read existing comments and add new ones. The comment form is always at the bottom of the main content area.

**Data Requirements**:
- Query: GetBugComments(bugId) (service-comment)
- Command: CreateComment (service-comment) — add comment

**Validation**:
- Comment body: required, max 65,535 characters
- Private comment: only insiders can set `isPrivate = true`
- `workTime`: only shown if user has `bugs:edit_timetracking` permission

**State**:
```typescript
interface CommentFormState {
  body: string;
  isPrivate: boolean;
  workTime: number | null;
  isSubmitting: boolean;
  previewMode: boolean;
}
```

## 4. Journey State Machine

```typescript
type BugDetailJourneyState =
  | { type: 'loading'; bugId: string }
  | { type: 'loaded'; bug: BugDetail; comments: Comment[]; activeTab: TabId }
  | { type: 'editing-field'; fieldName: string; value: any }
  | { type: 'submitting-edit'; fieldName: string }
  | { type: 'submitting-comment' }
  | { type: 'not-found'; bugId: string }
  | { type: 'forbidden'; bugId: string }
  | { type: 'error'; error: Error };
```

## 5. Error Recovery

| Error Scenario | Recovery Strategy |
|---------------|-------------------|
| Bug not found | Full-page error: "Bug #{id} not found." Link back to dashboard. |
| No visibility (group restriction) | "You do not have permission to view this bug." Link to request access. |
| Version conflict on field edit | Inline error: "This bug was modified by another user. Your changes were not saved." Refresh button. |
| Field validation failure | Inline error below the field. Field stays in edit mode. |
| Comment submission failure | Error toast. Comment text preserved in the form. "Try Again" button. |
| Attachment upload failure | Error toast with reason (file too large, type not allowed). File input stays. |
| Dependency cycle detected | Inline error: "Adding this dependency would create a cycle." Dependency not added. |
| Network failure | Full-page error state with "Try Again" button. |

## 6. Success States

- **Field edit**: Inline green "Saved" confirmation that fades after 2 seconds. Activity timeline updates.
- **Comment added**: Comment appears at bottom of thread. Form clears. Toast: "Comment added".
- **Attachment uploaded**: File appears in attachments tab. Toast: "Attachment uploaded".
- **Dependency added**: Dependency appears in list. Toast: "Dependency added".
- **Flag set**: Flag state updates. Toast: "Flag {name} set to {state}".

## 7. Progress Indication

- Page-level: skeleton screen while loading bug data
- Tab switching: no loading state (data pre-loaded or cached)
- In-place field edit: spinner on the field being saved
- Comment submission: spinner on "Add Comment" button
- Attachment upload: progress bar on upload

## 8. Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `e` | Edit mode (focus first editable field) |
| `c` | Focus comment textarea |
| `a` | Open attachments tab |
| `Esc` | Cancel current edit / close dialog |
| `←` / `→` | Navigate between tabs |

## Workflow & Rules Cross-References

### Tab: Details (Default) — Load bug
- **Workflow gateway / activity**: [no clear mapping — manual review needed] (read-only query path; bug detail page load issues GetBug, GetBugComments, GetBugHistory, GetBugUserLastVisit queries — not modeled as workflow nodes)
- **Decision rules**:
  - [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) — Bug visibility: user must be in at least one restricted group, in CC list, reporter, or have `bugs:update` for the product
- **Edge cases formalized**:
  - "Bug not found" → decision-point branch: `GetBug → not-found state` cite: [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy)
  - "No visibility (group restriction)" → decision-point branch: `GetBug → forbidden state (CanSeeBugPolicy group check)` cite: [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy)

### Tab: Details (Default) — In-place field edit
- **Workflow gateway / activity**: `workflow-bug-lifecycle / update-bug-start → update-bug → notify-bug-updated` (serviceTask chain for field changes) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:161]
- **Decision rules**:
  - [BR-bug-015](../decision-rules.md#br-bug-015) — Optimistic concurrency (expectedVersion check) on all mutations
  - [BR-bug-016](../decision-rules.md#br-bug-016) — Field validation in topological dependency order (VALIDATOR_DEPENDENCIES)
  - [BR-bug-POLICY-CanChangeFieldPolicy](../decision-rules.md#br-bug-policy-canchangefieldpolicy) — Field-level access control
  - [BR-bug-002](../decision-rules.md#br-bug-002) — Reporter is immutable after creation
- **Edge cases formalized**:
  - "Version conflict on field edit" → decision-point branch: `update-bug → reject(version_conflict)` cite: [BR-bug-015](../decision-rules.md#br-bug-015)
  - "Field validation failure" → decision-point branch: `update-bug → reject(field_validation_error)` cite: [BR-bug-016](../decision-rules.md#br-bug-016)

### Tab: Details (Default) — Status transition
- **Workflow gateway / activity**: `workflow-bug-lifecycle / status-transition-start → validate-transition → gateway-transition-type → check-open-blockers (if FIXED) → apply-status-transition` (serviceTask chain with gateway for resolution type branching) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:187]
- **Decision rules**:
  - [BR-bug-003](../decision-rules.md#br-bug-003) — Status transitions must follow the configurable workflow directed graph edges
  - [BR-bug-POLICY-ValidStatusTransitionPolicy](../decision-rules.md#br-bug-policy-validstatustransitionpolicy) — Verifies edge exists and is active, plus requireComment / allows_unconfirmed
  - [BR-bug-008](../decision-rules.md#br-bug-008) — No resolving as FIXED with open blockers
  - [BR-bug-POLICY-NoOpenBlockersPolicy](../decision-rules.md#br-bug-policy-noopenblockerspolicy) — When resolving FIXED, all blockers must be in closed status
  - [BR-bug-018](../decision-rules.md#br-bug-018) — Status transitions may require a comment (requireComment edge flag)
  - [BR-bug-022](../decision-rules.md#br-bug-022) — Transition to ASSIGNED/IN_PROGRESS requires non-default milestone if `musthavemilestoneonaccept`
- **Edge cases formalized**:
  - "Invalid transition (edge missing)" → decision-point branch: `validate-transition → reject(invalid_transition)`; cite: [BR-bug-003](../decision-rules.md#br-bug-003)
  - "Open blockers when resolving FIXED" → decision-point branch: `check-open-blockers → reject(still_unresolved_bugs)`; cite: [BR-bug-008](../decision-rules.md#br-bug-008)

### Tab: Details (Default) — Assignee change
- **Workflow gateway / activity**: `workflow-bug-lifecycle / update-bug` (AssignBug command; emits BugAssigned) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:165]
- **Decision rules**:
  - [BR-bug-POLICY-CanChangeFieldPolicy](../decision-rules.md#br-bug-policy-canchangefieldpolicy) — Only assignee can reassign; field-level access control
  - [BR-bug-NOTIF-BugAssigned](../decision-rules.md#br-bug-notif-bugassigned) — Notify service-notification on bug assignment
  - [BR-bug-POLICY-StrictIsolationPolicy](../decision-rules.md#br-bug-policy-strictisolationpolicy) — Under strict isolation, new assignee must have permission to edit the bug's product
- **Edge cases formalized**:
  - "Assignee invalid under strict isolation" → decision-point branch: `update-bug → reject(strict_isolation_violation)`; cite: [BR-bug-POLICY-StrictIsolationPolicy](../decision-rules.md#br-bug-policy-strictisolationpolicy)

### Tab: Details (Default) — CC list change
- **Workflow gateway / activity**: `workflow-bug-lifecycle / update-bug` (AddCc / RemoveCc commands) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:165]
- **Decision rules**:
  - [BR-bug-POLICY-CanChangeFieldPolicy](../decision-rules.md#br-bug-policy-canchangefieldpolicy) — Field-level permission for CC list
  - [BR-bug-POLICY-StrictIsolationPolicy](../decision-rules.md#br-bug-policy-strictisolationpolicy) — Under strict isolation, new CC user must have permission to edit the bug's product
- **Edge cases formalized**:
  - "CC user invalid under strict isolation" → decision-point branch: `update-bug → reject(strict_isolation_violation)`; cite: [BR-bug-POLICY-StrictIsolationPolicy](../decision-rules.md#br-bug-policy-strictisolationpolicy)

### Tab: Details (Default) — Time tracking edit
- **Workflow gateway / activity**: `workflow-bug-lifecycle / update-bug` (UpdateTimetracking command) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:165]
- **Decision rules**:
  - [BR-bug-014](../decision-rules.md#br-bug-014) — Time-tracking fields (estimatedTime, remainingTime, deadline) require dedicated `bugs:edit_timetracking` permission
  - [BR-bug-PERM-UpdateTimetracking](../decision-rules.md#br-bug-perm-updatetimetracking) — `bugs:edit_timetracking` JWT permission required
  - [BR-bug-013](../decision-rules.md#br-bug-013) — Remaining time auto-zeroed on close or duplicate
- **Edge cases formalized**:
  - "Lacks time-tracking permission" → decision-point branch: `update-bug → reject(insufficient_permission_for_timetracking)`; cite: [BR-bug-014](../decision-rules.md#br-bug-014)

### Tab: Activity
- **Workflow gateway / activity**: [no clear mapping — manual review needed] (read-only query: GetBugHistory; no workflow node models history retrieval)
- **Decision rules**:
  - [no clear mapping — manual review needed]
- **Edge cases formalized**:
  - [no clear mapping — manual review needed] (read-only tab with no error recovery rows)

### Tab: Attachments — View list
- **Workflow gateway / activity**: [no clear mapping — manual review needed] (read-only query: GetBugAttachments; no workflow node)
- **Decision rules**:
  - [no clear mapping — manual review needed]
- **Edge cases formalized**:
  - [no clear mapping — manual review needed] (read-only view with no error rows)

### Tab: Attachments — Upload
- **Workflow gateway / activity**: `workflow-bug-lifecycle / attachment-added-start → create-attachment → notify-attachment-created → create-attachment-system-comment` (serviceTask chain for attachment creation) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:128]
- **Decision rules**:
  - [BR-attachment-001](../decision-rules.md#br-attachment-001) — File size soft limit (`maxattachmentsize`)
  - [BR-attachment-002](../decision-rules.md#br-attachment-002) — File size hard limit (`maxlocalattachment`)
  - [BR-attachment-003](../decision-rules.md#br-attachment-003) — MIME type must match `LEGAL_CONTENT_TYPES` whitelist
  - [BR-attachment-004](../decision-rules.md#br-attachment-004) — MIME sniffing for octet-stream → patch detection
  - [BR-attachment-005](../decision-rules.md#br-attachment-005) — Filename sanitization (basename only, length-truncated)
  - [BR-attachment-PERM-CreateAttachment](../decision-rules.md#br-attachment-perm-createattachment) — `attachments:create` permission required
  - [BR-bug-POLICY-CanSeeBugPolicy](../decision-rules.md#br-bug-policy-canseebugpolicy) — Bug visibility + product-edit permission check
- **Edge cases formalized**:
  - "Attachment upload failure (file too large, type not allowed)" → decision-point branch: `create-attachment → reject(file_validation_error)`; cite: [BR-attachment-001](../decision-rules.md#br-attachment-001), [BR-attachment-002](../decision-rules.md#br-attachment-002), [BR-attachment-003](../decision-rules.md#br-attachment-003)

### Tab: Attachments — Edit metadata
- **Workflow gateway / activity**: `workflow-bug-lifecycle / create-attachment` (UpdateAttachmentMetadata command on existing aggregate) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:132]
- **Decision rules**:
  - [BR-attachment-POLICY-CanEditAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditattachmentpolicy) — Submitter or editbugs group; for private, must also be in insider group
  - [BR-attachment-017](../decision-rules.md#br-attachment-017) — Setting/editing `isPrivate=true` requires insider group membership
  - [BR-attachment-PERM-UpdateAttachmentMetadata](../decision-rules.md#br-attachment-perm-updateattachmentmetadata) — `attachments:update` permission required
- **Edge cases formalized**:
  - "Lacks insider group for private toggle" → decision-point branch: `create-attachment → reject(not_insider)`; cite: [BR-attachment-017](../decision-rules.md#br-attachment-017)

### Tab: Attachments — Delete
- **Workflow gateway / activity**: `workflow-bug-lifecycle / create-attachment` (DeleteAttachment command on existing aggregate) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:132]
- **Decision rules**:
  - [BR-attachment-008](../decision-rules.md#br-attachment-008) — Deleted attachment cannot receive further modifications
  - [BR-attachment-009](../decision-rules.md#br-attachment-009) — On deletion: clear flags, isObsolete=true, mimeType=text/plain, isPatch=false, binary removed
  - [BR-attachment-PERM-DeleteAttachment](../decision-rules.md#br-attachment-perm-deleteattachment) — `attachments:delete` permission required
  - [BR-attachment-ST-Active-Deleted](../decision-rules.md#br-attachment-st-active-deleted) — Active → Deleted transition; user must be submitter or have `attachments:delete`
- **Edge cases formalized**:
  - "Already-deleted attachment further modification attempt" → decision-point branch: command → reject(attachment_deleted); cite: [BR-attachment-008](../decision-rules.md#br-attachment-008)

### Tab: Dependencies — View
- **Workflow gateway / activity**: [no clear mapping — manual review needed] (read-only; dependency data returned as part of GetBug query)
- **Decision rules**:
  - [no clear mapping — manual review needed]
- **Edge cases formalized**:
  - [no clear mapping — manual review needed] (read-only view)

### Tab: Dependencies — Add dependency
- **Workflow gateway / activity**: `workflow-bug-lifecycle / dependency-add-start → add-dependency → notify-dependency-added` (serviceTask chain for dependency creation) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:301]
- **Decision rules**:
  - [BR-bug-006](../decision-rules.md#br-bug-006) — A bug cannot depend on itself
  - [BR-bug-007](../decision-rules.md#br-bug-007) — Adding a dependency cannot create a cycle
  - [BR-bug-POLICY-DependencyLoopPolicy](../decision-rules.md#br-bug-policy-dependencylooppolicy) — Rejects dependency addition if it would create a cycle
  - [BR-bug-POLICY-BugVisibilityPolicy](../decision-rules.md#br-bug-policy-bugvisibilitypolicy) — Referenced bug must be visible to current user
- **Edge cases formalized**:
  - "Dependency cycle detected" → decision-point branch: `add-dependency → reject(dependency_loop_multi)`; cite: [BR-bug-007](../decision-rules.md#br-bug-007)
  - "Self-reference" → decision-point branch: `add-dependency → reject(self_reference)`; cite: [BR-bug-006](../decision-rules.md#br-bug-006)
  - "Target bug invisible" → decision-point branch: `add-dependency → reject(target_invisible)`; cite: [BR-bug-POLICY-BugVisibilityPolicy](../decision-rules.md#br-bug-policy-bugvisibilitypolicy)

### Tab: Dependencies — Remove dependency
- **Workflow gateway / activity**: `workflow-bug-lifecycle / add-dependency` (RemoveBugDependency command; reverse operation on same flow) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:305]
- **Decision rules**:
  - [BR-bug-PERM-UpdateBug](../decision-rules.md#br-bug-perm-updatebug) — `bugs:update` permission required for dependency removal (mutation command)
- **Edge cases formalized**:
  - [no clear mapping — manual review needed] (no specific error row in Error Recovery table for dependency removal)

### Tab: Flags — View
- **Workflow gateway / activity**: [no clear mapping — manual review needed] (read-only queries: GetBugFlags, GetFlagTypes; no workflow node)
- **Decision rules**:
  - [no clear mapping — manual review needed]
- **Edge cases formalized**:
  - [no clear mapping — manual review needed] (read-only view)

### Tab: Flags — Set / request flag
- **Workflow gateway / activity**: `workflow-bug-lifecycle / create-attachment` (SetBugFlag command; flag operations are part of the attachment/flag service) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:132]
- **Decision rules**:
  - [BR-attachment-010](../decision-rules.md#br-attachment-010) — Flag type applicability via inclusion/exclusion rules
  - [BR-attachment-011](../decision-rules.md#br-attachment-011) — Inactive flag type cannot have new flag instances created
  - [BR-attachment-012](../decision-rules.md#br-attachment-012) — `targetType` must match command (bug vs attachment)
  - [BR-attachment-013](../decision-rules.md#br-attachment-013) — Non-multiplicable flag constraint (one flag per type per target)
  - [BR-attachment-015](../decision-rules.md#br-attachment-015) — Grant/request group membership rules per status
  - [BR-attachment-016](../decision-rules.md#br-attachment-016) — Requestee validation
  - [BR-attachment-POLICY-FlagTypeApplicabilityPolicy](../decision-rules.md#br-attachment-policy-flagtypeapplicabilitypolicy) — Flag type must be active, applicable, and `targetType` must match
  - [BR-attachment-POLICY-CanSetFlagPolicy](../decision-rules.md#br-attachment-policy-cansetflagpolicy) — Group membership per requested status
  - [BR-attachment-PERM-SetAttachmentFlagRequest](../decision-rules.md#br-attachment-perm-setattachmentflagrequest) — `flags:request` permission for `?` status
- **Edge cases formalized**:
  - "Flag type not applicable" → decision-point branch: `SetBugFlag → reject(flag_type_not_applicable)`; cite: [BR-attachment-010](../decision-rules.md#br-attachment-010)
  - "Multiplicable violation" → decision-point branch: `SetBugFlag → reject(multiplicable_violation)`; cite: [BR-attachment-013](../decision-rules.md#br-attachment-013)
  - "Insufficient group membership" → decision-point branch: `SetBugFlag → reject(insufficient_group_membership)`; cite: [BR-attachment-015](../decision-rules.md#br-attachment-015)

### Tab: Flags — Clear flag
- **Workflow gateway / activity**: `workflow-bug-lifecycle / create-attachment` (ClearBugFlag command) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:132]
- **Decision rules**:
  - [BR-attachment-015](../decision-rules.md#br-attachment-015) — Clearing (`X`) allowed for original setter or any grant/request group member; null group = anyone
  - [BR-attachment-POLICY-CanSetFlagPolicy](../decision-rules.md#br-attachment-policy-cansetflagpolicy) — Validates clearing permissions
  - [BR-attachment-PERM-SetAttachmentFlagSet](../decision-rules.md#br-attachment-perm-setattachmentflagset) — `flags:set` permission required for X status
- **Edge cases formalized**:
  - "Insufficient permission to clear" → decision-point branch: `ClearBugFlag → reject(insufficient_group_membership)`; cite: [BR-attachment-015](../decision-rules.md#br-attachment-015)

### Comment Thread — View comments
- **Workflow gateway / activity**: [no clear mapping — manual review needed] (read-only query: GetBugComments; no workflow node)
- **Decision rules**:
  - [BR-comment-007](../decision-rules.md#br-comment-007) — Private comments hidden from non-insiders unless caller has `comments:view:private`
  - [BR-comment-POLICY-CanSeePrivateCommentsPolicy](../decision-rules.md#br-comment-policy-canseeprivatecommentspolicy) — Filters private comments from query results
  - [BR-comment-PERM-GetBugComments](../decision-rules.md#br-comment-perm-getbugcomments) — `comments:read` permission required
- **Edge cases formalized**:
  - "Private comments stripped for non-insiders" → decision-point branch: `GetBugComments → filter private`; cite: [BR-comment-007](../decision-rules.md#br-comment-007)

### Comment Thread — Add comment
- **Workflow gateway / activity**: `workflow-bug-lifecycle / comment-added-start → create-comment → notify-comment-created → project-comment-work-time` (serviceTask chain for comment creation) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:96]
- **Decision rules**:
  - [BR-comment-003](../decision-rules.md#br-comment-003) — Comment body: 1–65,535 characters after normalization
  - [BR-comment-006](../decision-rules.md#br-comment-006) — Private comment requires `is_insider` flag
  - [BR-comment-014](../decision-rules.md#br-comment-014) — Referenced bug must exist
  - [BR-comment-015](../decision-rules.md#br-comment-015) — User must have product-edit permission via CanCommentOnBugPolicy
  - [BR-comment-POLICY-CanCommentOnBugPolicy](../decision-rules.md#br-comment-policy-cancommentonbugpolicy) — Validates bug existence and product-edit permission
  - [BR-comment-POLICY-IsInsiderPolicy](../decision-rules.md#br-comment-policy-isinsiderpolicy) — Insider flag required for privacy operations
  - [BR-comment-PERM-CreateComment](../decision-rules.md#br-comment-perm-createcomment) — `comments:create` permission required
- **Edge cases formalized**:
  - "Comment submission failure (body too long/empty)" → decision-point branch: `create-comment → reject(body_too_long or body_empty)`; cite: [BR-comment-003](../decision-rules.md#br-comment-003)
  - "Private comment without insider status" → decision-point branch: `create-comment → reject(not_insider)`; cite: [BR-comment-006](../decision-rules.md#br-comment-006)
  - "Bug not found" → decision-point branch: `create-comment → reject(bug_not_found)`; cite: [BR-comment-014](../decision-rules.md#br-comment-014)
