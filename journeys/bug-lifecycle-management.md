# Journey Spec: Bug Lifecycle Management

## 1. Overview

- **Journey name**: Bug Lifecycle Management (Status Transition)
- **Purpose**: Manage the status transitions of a bug through its lifecycle — from UNCONFIRMED to NEW to ASSIGNED to IN_PROGRESS to RESOLVED/VERIFIED/CLOSED, including marking as duplicate, reopening, and changing resolution.
- **Entry points**:
  - "Transition Status" dropdown on bug detail page
  - Quick-action buttons on bug detail page (contextual: "Resolve", "Reopen", "Confirm", "Assign to Me")
  - Bulk transition from bug list (multi-select → action bar)
- **Target user/actor**: Any authenticated user with `bugs:update` permission
- **Expected completion time**: 10–60 seconds per transition
- **Steps count**: 1–3 steps (branching, depends on transition type)
- **Flow type**: Branching (different paths for resolve, duplicate, reopen, simple transition)

## Workflow & Rules Mapping

| Journey step | Workflow activity | Decision rule(s) |
|--------------|-------------------|------------------|
| Step 1: Select Target Status | workflow-bug-lifecycle / status-transition-start → validate-transition | BR-bug-003 [unciteable: TODO], BR-bug-005 [unciteable: TODO], BR-bug-022 [unciteable: TODO] |
| Step 2a: Select Resolution (for closed statuses) | workflow-bug-lifecycle / gateway-transition-type → check-open-blockers → apply-status-transition | BR-bug-004 [unciteable: TODO], BR-bug-008 [unciteable: TODO] |
| Step 2b: Enter Duplicate Bug ID (for Mark as Duplicate) | workflow-bug-lifecycle / mark-duplicate-start → mark-duplicate | BR-bug-017 [unciteable: TODO] |
| Step 2c: Add Required Comment | workflow-bug-lifecycle / validate-transition | BR-bug-018 [unciteable: TODO] |
| Submit: Confirm Transition | workflow-bug-lifecycle / apply-status-transition → gateway-post-transition → notify-status-changed / notify-resolved / index-status-change → end-status-transitioned | BR-bug-003 [unciteable: TODO], BR-bug-004 [unciteable: TODO], BR-bug-008 [unciteable: TODO], BR-bug-012 [unciteable: TODO], BR-bug-013 [unciteable: TODO] |

## 2. Flow Diagram

```
[Trigger Transition] → [Step 1: Select Target Status]
                           │
                           ├── Simple transition (no comment required) → [Confirm] → [Done]
                           │
                           ├── Transition requiring comment → [Step 2: Add Comment] → [Confirm] → [Done]
                           │
                           ├── Resolve (to closed status) → [Step 2: Select Resolution] → [Step 3: Add Comment (opt)] → [Confirm] → [Done]
                           │
                           └── Mark as Duplicate → [Step 2: Enter Duplicate Bug ID] → [Step 3: Add Comment (opt)] → [Confirm] → [Done]
```

## 3. Steps

### Step 1: Select Target Status

**Purpose**: Choose the new status for the bug. Only valid transitions for the current status are shown.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Transition Bug #4521                                        │
│ Current status: ● New                                       │
│                                                             │
│ Transition to:                                              │
│                                                             │
│ ○ ASSIGNED         Assign to a developer                    │
│ ○ IN_PROGRESS      Start working                            │
│ ● RESOLVED         Apply a resolution                       │
│ ○ CLOSED           Final close                              │
│ ○ UNCONFIRMED      Un-confirm                               │
│                                                             │
│ ── Special Actions ──                                       │
│ ○ Mark as Duplicate                                        │
│                                                             │
│                                             [Cancel] [Next] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetValidStatusTransitions(currentStatus) (service-bug) — available target statuses with requireComment flags
- Query: GetBug(bugId) (service-bug) — current bug state (status, resolution, assignee)

**Validation**:
- Target status must be a valid transition from current status (enforced server-side via ValidStatusTransitionPolicy)
- Transition edge must be active
- If product has `musthavemilestoneonaccept` and target is ASSIGNED/IN_PROGRESS, milestone must be set
- If `allows_unconfirmed` is false on the product, UNCONFIRMED is not available

**State**:
```typescript
interface Step1State {
  currentStatus: string;
  targetStatus: string | null;
  isDuplicate: boolean;
  validTransitions: Array<{
    status: string;
    label: string;
    requireComment: boolean;
  }>;
  requiresResolution: boolean;
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Cancel | Exit (return to bug detail) | Always |
| Select simple status | Confirm (inline) | No resolution or comment required |
| Select closed status | Step 2 (Resolution) | `is_open = false` for target |
| Select "Mark as Duplicate" | Step 2 (Duplicate) | Always |
| Select status requiring comment | Step 2 (Comment) | `requireComment = true` |
| Next | Appropriate next step | Target status selected |

---

### Step 2a: Select Resolution (for closed statuses)

**Purpose**: Choose a resolution value. Required for all closed statuses.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Transition Bug #4521 → RESOLVED                             │
│                                                             │
│ Resolution *                                                │
│ ○ FIXED        Issue is fixed                               │
│ ○ INVALID      Not a valid bug                              │
│ ● WONTFIX      Won't be fixed                               │
│ ○ DUPLICATE    Duplicate of another bug                     │
│ ○ WORKSFORME   Cannot reproduce                             │
│ ○ INCOMPLETE   Incomplete information                       │
│ ○ MOVED        Moved elsewhere                              │
│                                                             │
│ Comment (optional)                                          │
│ ┌───────────────────────────────────────────────────────────┐
│ │                                                           │
│ └───────────────────────────────────────────────────────────┘
│                                                             │
│                                             [Cancel] [Next] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetLegalValues('resolution') (service-bug) — available resolution values
- Command: TransitionBugStatus (service-bug) — on confirm

**Validation**:
- Resolution is required when transitioning to a closed status
- Resolution must be from the configured values list
- Resolution FIXED: all dependency bugs must be in closed status (NoOpenBlockersPolicy)
- If `requireComment` flag is true on the transition edge, comment is required

**State**:
```typescript
interface Step2aState {
  resolution: string | null;
  comment: string;
  resolutions: string[];
  hasOpenBlockers: boolean;
  openBlockerIds: string[];
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Cancel | Exit | Always |
| Back | Step 1 | Always |
| Confirm | Submit | Resolution selected and valid |

---

### Step 2b: Enter Duplicate Bug ID (for Mark as Duplicate)

**Purpose**: Specify which bug this is a duplicate of.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Mark Bug #4521 as Duplicate                                 │
│                                                             │
│ Duplicate of Bug ID *                                       │
│ ┌───────────────────────────────────────────────────────────┐
│ │ #_____                                                   │
│ └───────────────────────────────────────────────────────────┘
│ Enter the bug number or alias of the original bug           │
│                                                             │
│ Comment (optional)                                          │
│ ┌───────────────────────────────────────────────────────────┐
│ │                                                           │
│ └───────────────────────────────────────────────────────────┘
│                                                             │
│                                             [Cancel] [Next] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetBug(duplicateOfBugId) (service-bug) — validate target bug exists and is visible
- Command: MarkAsDuplicate (service-bug) — on confirm

**Validation**:
- Duplicate target bug ID is required
- Target bug must exist (server-side: BugVisibilityPolicy)
- Target bug must be visible to current user
- Cannot mark a bug as duplicate of itself
- Status will be set to `duplicate_or_move_bug_status` (configurable, usually RESOLVED)

**State**:
```typescript
interface Step2bState {
  duplicateOfBugId: string | null;
  duplicateOfBugSummary: string | null;
  comment: string;
  isValidating: boolean;
  validationError: string | null;
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Cancel | Exit | Always |
| Back | Step 1 | Always |
| Confirm | Submit | Valid target bug ID |

---

### Step 2c: Add Required Comment

**Purpose**: Add a comment required by the workflow configuration for this transition.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Transition Bug #4521 → {targetStatus}                       │
│                                                             │
│ Comment * (required for this transition)                    │
│ ┌───────────────────────────────────────────────────────────┐
│ │                                                           │
│ │                                                           │
│ └───────────────────────────────────────────────────────────┘
│                                                             │
│                                             [Cancel] [Next] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: TransitionBugStatus (service-bug) — on confirm (with comment)

**Validation**:
- Comment body must not be empty if `requireComment = true` on the transition edge
- Comment body max 65,535 characters

**State**:
```typescript
interface Step2cState {
  comment: string;
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Cancel | Exit | Always |
| Back | Step 1 | Always |
| Confirm | Submit | Comment is non-empty |

---

### Submit: Confirm Transition

**Purpose**: Execute the status transition Command and update the UI.

**Commands dispatched**:
- Command: TransitionBugStatus (service-bug) — for standard transitions
- Command: MarkAsDuplicate (service-bug) — for duplicate marking
- If comment provided: Command: CreateComment (service-comment) — comment accompanies the transition

**Wireframe** (inline on bug detail page):
```
┌─────────────────────────────────────────────────────────────┐
│ Bug #4521  ● Resolved / Fixed       [Edit] [Transition ▼]  │
│ Fix crash on launch during session restore                  │
│                                                             │
│  ✓ Status changed from NEW to RESOLVED / FIXED              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. Journey State Machine

```typescript
type TransitionJourneyState =
  | { type: 'selecting-status'; bugId: string; currentStatus: string; validTransitions: Transition[] }
  | { type: 'selecting-resolution'; bugId: string; targetStatus: string }
  | { type: 'entering-duplicate'; bugId: string }
  | { type: 'adding-required-comment'; bugId: string; targetStatus: string }
  | { type: 'submitting'; transition: PendingTransition }
  | { type: 'complete'; oldStatus: string; newStatus: string; resolution?: string }
  | { type: 'error'; error: TransitionError; step: string };
```

## 5. Error Recovery

| Error Scenario | Recovery Strategy |
|---------------|-------------------|
| Invalid transition (edge missing) | Hide invalid options. If somehow submitted, show error: "This transition is not valid from {currentStatus}." |
| Resolution missing on close | Highlight resolution field as required. Prevent submission. |
| Open blockers on FIXED resolution | Show blocking bug IDs with links. User must resolve blockers first or choose different resolution. |
| Duplicate target not found | Inline error: "Bug #{id} not found." Allow re-entry. |
| Duplicate target invisible | Inline error: "You do not have permission to view that bug." |
| Self-duplicate | Inline error: "A bug cannot be marked as a duplicate of itself." |
| Version conflict (concurrent edit) | Show conflict banner: "This bug was modified by another user. Please refresh and try again." |
| Comment required but empty | Highlight comment field. Show "A comment is required for this transition." |
| Product musthavemilestoneonaccept | Show: "A target milestone is required before assigning this bug." Link to set milestone. |
| Network failure | Error toast with "Try Again" button. |

## 6. Success State

- **Inline confirmation** on bug detail page:
  - Status badge updates to new status
  - Resolution badge appears (if applicable)
  - Activity timeline entry: "{user} changed status from {old} to {new}"
  - If resolved: toast notification: "Bug #4521 transitioned to RESOLVED / FIXED"
  - If reopened: toast notification: "Bug #4521 reopened"
  - If assigned: assignee field updates, toast: "Bug #4521 assigned to {user}"
- **Screen reader**: "Bug #{id} status changed to {status}"
- **No navigation change** — user stays on bug detail page

## 7. Progress Indication

- Modal/dialog-based flow — no traditional step indicator needed
- Submitting state: spinner on confirm button, modal not dismissible
- Quick transitions (single-click: "Resolve as Fixed", "Reopen", "Confirm") skip the wizard and show only a brief spinner then success toast
- For multi-step transitions, show breadcrumb in modal header: "Select Status → Resolution → Confirm"

## 8. Quick Actions (Shortcuts)

For power users, common transitions are available as single-click buttons on the bug detail page:

| Current Status | Quick Action | Effect |
|---------------|-------------|--------|
| UNCONFIRMED | "Confirm" | → NEW |
| NEW | "Assign to Me" | Command: AssignBug + → ASSIGNED |
| ASSIGNED | "Start Working" | → IN_PROGRESS |
| IN_PROGRESS | "Resolve as Fixed" | → RESOLVED / FIXED (prompts for comment) |
| RESOLVED | "Reopen" | → REOPENED |
| VERIFIED | "Reopen" | → REOPENED |
| CLOSED | "Reopen" | → REOPENED |
| Any open status | "Mark as Duplicate" | Opens duplicate flow |

## Workflow & Rules Cross-References

### Step 1: Select Target Status

- **Workflow gateway / activity**: `workflow-bug-lifecycle / status-transition-start → validate-transition`
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:187]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:191]
- **Decision rules**:
  - BR-bug-003: Status transitions must follow the workflow graph — every transition must correspond to an active directed edge in StatusWorkflowConfig [unciteable: TODO]
  - BR-bug-005: UNCONFIRMED availability is product-scoped — product with `allows_unconfirmed = false` rejects UNCONFIRMED transitions [unciteable: TODO]
  - BR-bug-022: Must have milestone on accept when product requires it — transitioning to ASSIGNED/IN_PROGRESS requires non-default target milestone if product flag is set [unciteable: TODO]
- **Edge cases formalized**:
  - `Invalid transition (edge missing)` → decision-point branch: `validate-transition → reject with invalid transition error`; cite: BR-bug-003 [unciteable: TODO]
  - `Product musthavemilestoneonaccept` → decision-point branch: `validate-transition → reject with milestone-required error`; cite: BR-bug-022 [unciteable: TODO]
  - `allows_unconfirmed is false` → decision-point branch: `validate-transition → hide UNCONFIRMED option / reject UNCONFIRMED transition`; cite: BR-bug-005 [unciteable: TODO]

### Step 2a: Select Resolution (for closed statuses)

- **Workflow gateway / activity**: `workflow-bug-lifecycle / gateway-transition-type → check-open-blockers → apply-status-transition`
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:200]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:211]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:221]
- **Decision rules**:
  - BR-bug-004: Resolution is required for closed statuses and cleared on reopen [unciteable: TODO]
  - BR-bug-008: No resolving as FIXED with open blockers — NoOpenBlockersPolicy queries BugDependencyReadModel for open blockers [unciteable: TODO]
- **Edge cases formalized**:
  - `Resolution missing on close` → decision-point branch: `gateway-transition-type → reject with resolution-required validation`; cite: BR-bug-004 [unciteable: TODO]
  - `Open blockers on FIXED resolution` → decision-point branch: `check-open-blockers → reject with still_unresolved_bugs error`; cite: BR-bug-008 [unciteable: TODO]

### Step 2b: Enter Duplicate Bug ID (for Mark as Duplicate)

- **Workflow gateway / activity**: `workflow-bug-lifecycle / mark-duplicate-start → mark-duplicate`
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:271]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:275]
- **Decision rules**:
  - BR-bug-017: Duplicate target must exist and be visible — CanSeeBugPolicy / BugVisibilityPolicy enforces target bug visibility [unciteable: TODO]
- **Edge cases formalized**:
  - `Duplicate target not found` → decision-point branch: `mark-duplicate → reject with bug-not-found error`; cite: BR-bug-017 [unciteable: TODO]
  - `Duplicate target invisible` → decision-point branch: `mark-duplicate → reject with visibility error`; cite: BR-bug-017 [unciteable: TODO]
  - `Self-duplicate` → decision-point branch: `mark-duplicate → reject with self-reference error`; cite: BR-bug-017 [unciteable: TODO]

### Step 2c: Add Required Comment

- **Workflow gateway / activity**: `workflow-bug-lifecycle / validate-transition`
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:191]
- **Decision rules**:
  - BR-bug-018: Transitions may require comments — if workflow edge has `requireComment: true`, the command must include a comment [unciteable: TODO]
- **Edge cases formalized**:
  - `Comment required but empty` → decision-point branch: `validate-transition → reject with comment-required validation`; cite: BR-bug-018 [unciteable: TODO]

### Submit: Confirm Transition

- **Workflow gateway / activity**: `workflow-bug-lifecycle / apply-status-transition → gateway-post-transition → notify-status-changed / notify-resolved / index-status-change → end-status-transitioned`
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:221]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:231]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:242]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:251]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:263]
  [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:341]
- **Decision rules**:
  - BR-bug-003: Status transitions must follow the workflow graph [unciteable: TODO]
  - BR-bug-004: Resolution required for closed statuses, cleared on reopen [unciteable: TODO]
  - BR-bug-008: No resolving as FIXED with open blockers [unciteable: TODO]
  - BR-bug-012: Ever confirmed is derived and monotonic — once a bug leaves UNCONFIRMED, everConfirmed is permanently true [unciteable: TODO]
  - BR-bug-013: Remaining time is zeroed on close or duplicate [unciteable: TODO]
- **Edge cases formalized**:
  - `Version conflict (concurrent edit)` → decision-point branch: `apply-status-transition → reject with version mismatch error`; cite: BR-bug-015 (Optimistic concurrency on all mutations) [unciteable: TODO]
  - `Network failure` → decision-point branch: client-side only — retry with "Try Again" button; cite: [unciteable: TODO]
