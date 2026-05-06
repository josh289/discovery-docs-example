# Journey Spec: File a Bug

## 1. Overview

- **Journey name**: File a Bug
- **Purpose**: Guide a user through creating a new bug report with all required and optional fields, ensuring product/component/version validation is enforced at each step.
- **Entry points**: 
  - "File a Bug" button in header (global)
  - "File a Bug" nav item in sidebar
  - [+ New Bug] button on product/component list pages
  - Quick keyboard shortcut `n` (when not in an input field)
- **Target user/actor**: Any authenticated user with `bugs:create` permission
- **Expected completion time**: 2–5 minutes
- **Steps count**: 4 steps (linear, with optional sections)
- **Flow type**: Linear wizard with collapsible optional sections

## Workflow & Rules Mapping

| Journey step | Workflow activity | Decision rule(s) |
|--------------|-------------------|------------------|
| Step 1: Classification | workflow-bug-lifecycle / start → validate-product-access → validate-workflow-status | [BR-bug-PERM-CreateBug](../decision-rules.md#br-bug-perm-createbug), [BR-product-POLICY-ProductAccessPolicy](../decision-rules.md#br-product-policy-productaccesspolicy), [BR-bug-016](../decision-rules.md#br-bug-016), [BR-bug-005](../decision-rules.md#br-bug-005) |
| Step 2: Details | workflow-bug-lifecycle / create-bug-aggregate (field validation sub-step) | [BR-bug-001](../decision-rules.md#br-bug-001), [BR-bug-016](../decision-rules.md#br-bug-016) |
| Step 3: People & Tags | workflow-bug-lifecycle / create-bug-aggregate (people/CC/keyword sub-step) | [BR-bug-011](../decision-rules.md#br-bug-011), [BR-bug-POLICY-StrictIsolationPolicy](../decision-rules.md#br-bug-policy-strictisolationpolicy), [BR-bug-016](../decision-rules.md#br-bug-016) |
| Step 4: Review & Submit | workflow-bug-lifecycle / create-bug-aggregate → gateway-post-creation → notify-bug-created → end-created | [BR-bug-001](../decision-rules.md#br-bug-001), [BR-bug-005](../decision-rules.md#br-bug-005), [BR-bug-015](../decision-rules.md#br-bug-015), [BR-bug-016](../decision-rules.md#br-bug-016), [BR-bug-NOTIF-BugCreated](../decision-rules.md#br-bug-notif-bugcreated) |

## 2. Flow Diagram

```
[Start] → [Step 1: Classification] → [Step 2: Details] → [Step 3: People & Tags] → [Step 4: Review & Submit] → [Done: Bug Created]
                                                    ↘ (skip optional) ↗
```

## 3. Steps

### Step 1: Classification

**Purpose**: Select the product, component, and version for the bug. These fields are required and cascade — component options filter by the selected product.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Step 1 of 4: Classification                                 │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Product *                                                   │
│ ┌───────────────────────────────┐                           │
│ │ Select a product…           ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ Component *                                                 │
│ ┌───────────────────────────────┐                           │
│ │ Select a component…         ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ Version *                                                   │
│ ┌───────────────────────────────┐                           │
│ │ Select a version…           ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│                                                             │
│                              [Cancel]          [Next →]     │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetProducts (service-product) — populate product dropdown
- Query: GetComponents(productId) (service-product) — populate component dropdown, filtered by product
- Query: GetVersions(productId) (service-product) — populate version dropdown, filtered by product

**Validation**:
- Product is required (client-side, immediate on blur)
- Component is required; disabled until product is selected
- Version is required; disabled until product is selected
- Server-side: product must be active, component must belong to selected product, version must belong to selected product

**State**:
```typescript
interface Step1State {
  productId: string | null;
  componentId: string | null;
  versionId: string | null;
  products: Product[];
  components: Component[];
  versions: Version[];
  isLoadingComponents: boolean;
  isLoadingVersions: boolean;
  validationErrors: Record<string, string>;
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Cancel | Exit journey | Always (confirm dialog if data entered) |
| Product selected | Stay | Load components + versions |
| Next | Step 2 | All three fields filled and valid |
| Next (invalid) | Stay | Show validation errors |

---

### Step 2: Details

**Purpose**: Enter the bug summary, description, severity, priority, and other classification details.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Step 2 of 4: Details                                        │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Summary *                                                   │
│ ┌───────────────────────────────────────────────────────────┐
│ │ Enter a short summary of the bug…                         │
│ └───────────────────────────────────────────────────────────┘
│                                                             │
│ Severity                                                    │
│ ┌───────────────────────────────┐                           │
│ │ normal                      ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ Priority                                                    │
│ ┌───────────────────────────────┐                           │
│ │ P3                          ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ ── Optional Fields ────────────────────────────────────     │
│                                                             │
│ Operating System                                             │
│ ┌───────────────────────────────┐                           │
│ │ All                         ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ Hardware / Platform                                          │
│ ┌───────────────────────────────┐                           │
│ │ All                         ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ URL                                                         │
│ ┌───────────────────────────────────────────────────────────┐
│ │ https://…                                                 │
│ └───────────────────────────────────────────────────────────┘
│                                                             │
│                                                             │
│ [← Back]                                     [Next →]      │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetLegalValues('severity', productId) (service-bug) — legal severity values
- Query: GetLegalValues('priority', productId) (service-bug) — legal priority values
- Query: GetLegalValues('op_sys', productId) (service-bug) — OS values
- Query: GetLegalValues('rep_platform', productId) (service-bug) — platform values

**Validation**:
- Summary is required, max MAX_FREETEXT_LENGTH characters (client-side character counter)
- Summary must not be empty or whitespace-only
- URL must be valid format if provided (client-side)
- Severity, priority, OS, platform must be from legal values list if provided

**State**:
```typescript
interface Step2State {
  summary: string;
  severity: string;
  priority: string;
  operatingSystem: string;
  platform: string;
  url: string;
  legalValues: {
    severities: string[];
    priorities: string[];
    operatingSystems: string[];
    platforms: string[];
  };
  validationErrors: Record<string, string>;
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Back | Step 1 | Always (preserve entered data) |
| Next | Step 3 | Summary is non-empty and within length limit |
| Next (invalid) | Stay | Show validation errors |

---

### Step 3: People & Tags

**Purpose**: Configure people (assignee, QA contact, CC list), keywords, and group visibility. All fields are optional.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Step 3 of 4: People & Tags                                  │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Assignee                                                    │
│ ┌───────────────────────────────┐                           │
│ │ Search for a user…          ▼│  ← default from component │
│ └───────────────────────────────┘                           │
│                                                             │
│ QA Contact                                                  │
│ ┌───────────────────────────────┐                           │
│ │ Search for a user…          ▼│  ← default from component │
│ └───────────────────────────────┘                           │
│                                                             │
│ CC List                                                     │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ @alice ●  @bob ●                                       │   │
│ │───────────────────────────────────────────────────────│   │
│ │ Type a name or email to add…                          │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ ── Keywords & Groups ──────────────────────────────────     │
│                                                             │
│ Keywords                                                    │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ crash ●  regression ●                                  │   │
│ │───────────────────────────────────────────────────────│   │
│ │ Add keyword…                                          │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ Group Restrictions                                          │
│ ☐ firefox-security                                         │
│ ☐ core-security                                             │
│                                                             │
│                                                             │
│ [← Back]                            [Skip]     [Next →]    │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetComponentDefaults(componentId) (service-product) — pre-fill default assignee, QA contact, initial CC
- Query: SearchUsers(query) (service-user) — user search/typeahead for assignee, QA, CC fields
- Query: GetKeywords (service-bug) — keyword suggestions
- Query: GetGroups (service-user) — group list for restriction checkboxes

**Validation**:
- Assignee must be a valid user (server-side)
- QA contact must be a valid user (server-side)
- CC list entries must be valid users (server-side)
- Keywords must be from the defined keyword list
- Groups must be applicable to the selected product

**State**:
```typescript
interface Step3State {
  assignedTo: string | null;
  qaContact: string | null;
  ccList: string[];
  keywords: string[];
  groupIds: string[];
  userSearchResults: User[];
  availableKeywords: Keyword[];
  availableGroups: Group[];
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Back | Step 2 | Always (preserve entered data) |
| Skip | Step 4 | All fields optional — proceed with defaults |
| Next | Step 4 | All entered values valid (no blocking errors) |

---

### Step 4: Review & Submit

**Purpose**: Review all entered data before submitting. Show the complete bug report as it will appear. Allow the user to go back and change any field.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Step 4 of 4: Review & Submit                                │
│ ────────────────────────────────────────────                 │
│                                                             │
│ ┌─ Classification ───────────────────────── [Edit ▸] ─────┐ │
│ │  Product:    Firefox                                     │ │
│ │  Component:  General                                     │ │
│ │  Version:    120.0                                       │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─ Details ────────────────────────────────── [Edit ▸] ───┐ │
│ │  Summary:    Fix crash on launch during session restore  │ │
│ │  Severity:   normal                                      │ │
│ │  Priority:   P3                                          │ │
│ │  OS:         All                                         │ │
│ │  Platform:   All                                         │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─ People ─────────────────────────────────── [Edit ▸] ───┐ │
│ │  Assignee:   @josh (default)                             │ │
│ │  QA:         @alex (default)                             │ │
│ │  CC:         @alice, @bob                                │ │
│ │  Keywords:   crash, regression                           │ │
│ │  Groups:     (none)                                      │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                             │
│                                                             │
│ [← Back]                                 [Submit Bug]      │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: CreateBug (service-bug) — on submit
  - Inputs: summary, productId, componentId, versionId, status (auto-determined), severity, priority, platform, operatingSystem, assignedTo, qaContact, url, ccList, keywords, groupIds

**Validation**:
- All Step 1–3 validations re-checked server-side
- Summary must not be empty
- Product/component/version must be active and valid
- Initial status determined by product's `allows_unconfirmed` flag:
  - If `allows_unconfirmed = true` → status = UNCONFIRMED
  - If `allows_unconfirmed = false` → status = NEW

**State**:
```typescript
interface Step4State {
  isSubmitting: boolean;
  submitError: string | null;
  initialStatus: 'UNCONFIRMED' | 'NEW';
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Back | Step 3 | Always |
| Edit (section) | Respective step | Jump to the step for that section |
| Submit Bug | Complete | CreateBug succeeds |
| Submit Bug (error) | Stay | Show error, allow retry |

## 4. Journey State Machine

```typescript
type FileBugJourneyState =
  | { type: 'step'; stepIndex: 0; data: Step1State }
  | { type: 'step'; stepIndex: 1; data: Step2State }
  | { type: 'step'; stepIndex: 2; data: Step2State }
  | { type: 'step'; stepIndex: 3; data: Step4State }
  | { type: 'submitting'; data: AllStepsData }
  | { type: 'complete'; bugId: string; bugNumber: number }
  | { type: 'error'; error: Error; data: AllStepsData; lastStep: number };
```

## 5. Error Recovery

| Error Scenario | Recovery Strategy |
|---------------|-------------------|
| Summary too long | Inline character counter with red highlight at limit. Truncation not allowed. |
| Product/component/version inactive | Show error inline. User must select a different value. |
| Server validation failure on submit | Display error banner at top of review step. Highlight the failing field's section. Allow edit. |
| Network failure on submit | Show error toast with "Try Again" button. Data preserved in form state. |
| Session expiry mid-journey | Persist form data to localStorage. On return, restore from localStorage and show "Resume your bug report" banner. |
| Back navigation | All entered data is preserved. No data loss on back/forward. |
| Cascading select reset | Changing product clears component and version selections. Warn user before clearing. |

## 6. Success State

On successful bug creation:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ✓ Bug #4521 created                                        │
│                                                             │
│  Fix crash on launch during session restore                 │
│  Status: ● Unconfirmed                                      │
│                                                             │
│  [View Bug #4521]  [File Another Bug]                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

- Toast notification: "Bug #4521 created"
- Primary CTA: "View Bug #4521" → navigates to bug detail page
- Secondary CTA: "File Another Bug" → resets wizard with same product/component pre-selected
- Screen reader announcement: "Bug #{id} created successfully"

## 7. Progress Indication

- **Step indicator**: Numbered steps with labels at the top of the form
  ```
  ① Classification  ─── ② Details  ─── ③ People & Tags  ─── ④ Review
  ```
- Completed steps show a checkmark: ✓
- Current step is highlighted with accent color
- Steps are NOT clickable for direct navigation (must use Back/Next)
- Progress bar below step indicator fills proportionally

## 8. Mobile Considerations

- Full-width layout, single column
- Select dropdowns use native mobile pickers
- CC list and keyword tags wrap naturally
- "Review" step uses accordion sections instead of side-by-side cards
- Sticky bottom bar with Back/Next buttons

## Workflow & Rules Cross-References

### Step 1: Classification

- **Workflow gateway / activity**:
  - `workflow-bug-lifecycle / start` — user submits "File a Bug" form with product, component, version [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:10]
  - `workflow-bug-lifecycle / validate-product-access` — validate user can enter product (group_control_map), resolve allowsUnconfirmed flag and component defaults [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:17]
  - `workflow-bug-lifecycle / validate-workflow-status` — resolve initial status from StatusWorkflowConfig [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:24]

- **Decision rules**:
  - [BR-product-ST-Active-Inactive](../decision-rules.md#br-product-st-active-inactive) — Product must be active (`isActive = true`); enforced by service-product `CheckProductAccess` query and service-bug `ProductInfoReadModel`
  - [BR-bug-016](../decision-rules.md#br-bug-016) — Component must belong to selected product; enforced by service-bug field validation in dependency order
  - [BR-bug-016](../decision-rules.md#br-bug-016) — Version must belong to selected product; enforced by service-bug field validation in dependency order
  - [BR-bug-PERM-CreateBug](../decision-rules.md#br-bug-perm-createbug) — User must have `bugs:create` permission (service-bug Authorization Layer 1)
  - [BR-product-POLICY-ProductAccessPolicy](../decision-rules.md#br-product-policy-productaccesspolicy) — Product group access check: if entry groups are configured, user must be a member of at least one

- **Edge cases formalized**:
  - Product inactive → decision-point branch: `validate-product-access` → reject (user must select different product); cite: [BR-product-ST-Active-Inactive](../decision-rules.md#br-product-st-active-inactive)
  - Component does not belong to selected product → decision-point branch: `validate-product-access` → reject (cascading select resets component/version); cite: [BR-bug-016](../decision-rules.md#br-bug-016)
  - Version does not belong to selected product → decision-point branch: `validate-product-access` → reject (cascading select resets component/version); cite: [BR-bug-016](../decision-rules.md#br-bug-016)
  - User lacks group access to product → decision-point branch: `validate-product-access` → reject (product not shown or access denied); cite: [BR-product-POLICY-ProductAccessPolicy](../decision-rules.md#br-product-policy-productaccesspolicy)
  - Changing product clears component and version selections (cascading select reset) → decision-point branch: client-side reset with user warning; cite: [UI-only — no business rule]

### Step 2: Details

- **Workflow gateway / activity**:
  - `workflow-bug-lifecycle / create-bug-aggregate` — fields validated in topological dependency order (VALIDATOR_DEPENDENCIES) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:31]

- **Decision rules**:
  - [BR-bug-001](../decision-rules.md#br-bug-001) — Summary is required and length-bounded (max MAX_FREETEXT_LENGTH characters)
  - [BR-bug-001](../decision-rules.md#br-bug-001) — Summary must not be empty or whitespace-only
  - [BR-bug-016](../decision-rules.md#br-bug-016) — Severity, priority, OS, platform must be from legal values list if provided (validator dependency order)
  - [UI-only — no business rule] — URL must be valid format if provided (client-side validation only)

- **Edge cases formalized**:
  - Summary too long → decision-point branch: `create-bug-aggregate` → reject (inline character counter with red highlight, truncation not allowed); cite: [BR-bug-001](../decision-rules.md#br-bug-001)
  - Summary empty or whitespace-only → decision-point branch: `create-bug-aggregate` → reject (show validation error); cite: [BR-bug-001](../decision-rules.md#br-bug-001)
  - Severity/priority/OS/platform value not in legal values list → decision-point branch: `create-bug-aggregate` → reject (show validation error); cite: [BR-bug-016](../decision-rules.md#br-bug-016)
  - URL format invalid → decision-point branch: client-side validation block (show validation error); cite: [UI-only — no business rule]

### Step 3: People & Tags

- **Workflow gateway / activity**:
  - `workflow-bug-lifecycle / create-bug-aggregate` — Component initial_cc auto-added to CC list; default assignee/QA resolved from component defaults [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:31]

- **Decision rules**:
  - [BR-bug-016](../decision-rules.md#br-bug-016) — Assignee must be a valid, enabled user (server-side, validator dependency order)
  - [BR-product-009](../decision-rules.md#br-product-009) — QA contact accepted only when global `useqacontact` parameter is enabled and user must reference an existing enabled user
  - [BR-bug-016](../decision-rules.md#br-bug-016) — CC list entries must be valid users; component default CC auto-added
  - [BR-bug-016](../decision-rules.md#br-bug-016) — Keywords must be from the defined keyword list (validator dependency order)
  - [BR-bug-011](../decision-rules.md#br-bug-011) — Groups marked mandatory in the product's group_control_map cannot be removed; group applicability enforced
  - [BR-bug-POLICY-StrictIsolationPolicy](../decision-rules.md#br-bug-policy-strictisolationpolicy) — Under strict isolation, user being added to a role must have permission to edit the bug's product

- **Edge cases formalized**:
  - Assignee user not found or disabled → decision-point branch: `create-bug-aggregate` → reject (show error, user must select different assignee); cite: [BR-bug-016](../decision-rules.md#br-bug-016)
  - QA contact user not found or disabled → decision-point branch: `create-bug-aggregate` → reject (show error); cite: [BR-product-009](../decision-rules.md#br-product-009)
  - CC list entry not a valid user → decision-point branch: `create-bug-aggregate` → reject (show error); cite: [BR-bug-016](../decision-rules.md#br-bug-016)
  - Keyword not in defined keyword list → decision-point branch: `create-bug-aggregate` → reject (show error); cite: [BR-bug-016](../decision-rules.md#br-bug-016)
  - Group not applicable to selected product → decision-point branch: `create-bug-aggregate` → reject (show error); cite: [BR-bug-011](../decision-rules.md#br-bug-011)

### Step 4: Review & Submit

- **Workflow gateway / activity**:
  - `workflow-bug-lifecycle / create-bug-aggregate` — CreateBug command: creates BugAggregate, emits BugCreated [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:31]
  - `workflow-bug-lifecycle / validate-workflow-status` — Initial status: UNCONFIRMED if product `allows_unconfirmed`, else NEW [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:24]
  - `workflow-bug-lifecycle / gateway-post-creation` — Post-creation fan-out: notifications, search index, comment scope, attachment scope [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:39]
  - `workflow-bug-lifecycle / notify-bug-created` — Send "new bug" email to CC, assignee, QA, reporter [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:47]
  - `workflow-bug-lifecycle / index-bug-created` — Index new bug in search engine [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:56]
  - `workflow-bug-lifecycle / seed-comment-scope` — Initialize comment scope for new bug [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:61]
  - `workflow-bug-lifecycle / seed-attachment-scope` — Initialize attachment scope for new bug [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:66]
  - `workflow-bug-lifecycle / end-created` — Bug created; lifecycle proceeds through status transitions [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:116]

- **Decision rules**:
  - [BR-bug-016](../decision-rules.md#br-bug-016) — All Step 1–3 validations re-checked server-side via field validator dependency order
  - [BR-bug-001](../decision-rules.md#br-bug-001) — Summary must not be empty (length-bounded)
  - [BR-bug-016](../decision-rules.md#br-bug-016) — Product/component/version must be active and valid (validator dependency order)
  - [BR-bug-005](../decision-rules.md#br-bug-005) — Initial status determined by product's `allows_unconfirmed` flag: UNCONFIRMED if true, NEW if false
  - [BR-bug-015](../decision-rules.md#br-bug-015) — Optimistic concurrency (expectedVersion check) on all mutations
  - [BR-bug-NOTIF-BugCreated](../decision-rules.md#br-bug-notif-bugcreated) — Notify service-comment, service-notification, service-search on bug creation; component default CC auto-added

- **Edge cases formalized**:
  - Server validation failure on submit → decision-point branch: `create-bug-aggregate` → reject (display error banner at top of review step, highlight failing field's section, allow edit); cite: [BR-bug-016](../decision-rules.md#br-bug-016)
  - Network failure on submit → decision-point branch: client-side retry (show error toast with "Try Again" button, data preserved in form state); cite: [UI-only — no business rule]
  - Session expiry mid-journey → decision-point branch: client-side recovery (persist form data to localStorage, restore on return with "Resume your bug report" banner); cite: [UI-only — no business rule]
  - Product `allows_unconfirmed = true` → initial status = UNCONFIRMED; decision-point branch: `validate-workflow-status` → UNCONFIRMED; cite: [BR-bug-005](../decision-rules.md#br-bug-005)
  - Product `allows_unconfirmed = false` → initial status = NEW; decision-point branch: `validate-workflow-status` → NEW; cite: [BR-bug-005](../decision-rules.md#br-bug-005)
