# Journey Spec: Manage Attachments and Flags

## 1. Overview

- **Journey name**: Manage Attachments and Flags
- **Purpose**: Handle the full lifecycle of file attachments on bugs (upload, view, download, update metadata, delete) and flag workflows (request review, grant/deny, clear). These are closely related workflows often used together during bug resolution.
- **Entry points**:
  - "Attachments" tab on bug detail page
  - Drag-and-drop file onto bug detail page
  - "Flags" tab on bug detail page
  - Attachment link in comment thread (auto-generated system comments)
- **Target user/actor**: Any authenticated user with product-edit permission (attachments), flag requestees/granters (flags)
- **Expected completion time**: 15 seconds (upload) to 2 minutes (flag request + review cycle)
- **Steps count**: 1–2 steps per action
- **Flow type**: Hub-and-spoke from bug detail (modal/dialog overlays)

## Workflow & Rules Mapping

| Journey step | Workflow activity | Decision rule(s) |
|--------------|-------------------|------------------|
| Upload Attachment | workflow-bug-lifecycle / attachment-added-start → create-attachment | [BR-attachment-001](../decision-rules.md#br-attachment-001), [BR-attachment-002](../decision-rules.md#br-attachment-002), [BR-attachment-003](../decision-rules.md#br-attachment-003), [BR-attachment-005](../decision-rules.md#br-attachment-005), [BR-attachment-017](../decision-rules.md#br-attachment-017), [BR-attachment-PERM-CreateAttachment](../decision-rules.md#br-attachment-perm-createattachment) |
| Upload Attachment | workflow-flag-review-approval / start → validate-flag-type-applicability (when initial flags included) | [BR-attachment-010](../decision-rules.md#br-attachment-010), [BR-attachment-POLICY-FlagTypeApplicabilityPolicy](../decision-rules.md#br-attachment-policy-flagtypeapplicabilitypolicy) |
| View / Preview Attachment | [no clear mapping — manual review needed] | [BR-attachment-POLICY-CanEditPrivateAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditprivateattachmentpolicy), [BR-attachment-017](../decision-rules.md#br-attachment-017) |
| Edit Attachment Metadata | workflow-bug-lifecycle / update-bug-start (analogous field-update branch for attachment metadata) | [BR-attachment-008](../decision-rules.md#br-attachment-008), [BR-attachment-017](../decision-rules.md#br-attachment-017), [BR-attachment-POLICY-CanEditAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditattachmentpolicy), [BR-attachment-PERM-UpdateAttachmentMetadata](../decision-rules.md#br-attachment-perm-updateattachmentmetadata) |
| Delete Attachment | workflow-bug-lifecycle / end-attachment-added (reverse — cleanup path) | [BR-attachment-008](../decision-rules.md#br-attachment-008), [BR-attachment-009](../decision-rules.md#br-attachment-009), [BR-attachment-PERM-DeleteAttachment](../decision-rules.md#br-attachment-perm-deleteattachment), [BR-attachment-ST-Active-Deleted](../decision-rules.md#br-attachment-st-active-deleted) |
| Delete Attachment | workflow-flag-review-approval / start-obsolete → cancel-pending-flags-obsolete (cascade cleanup of flags) | [BR-attachment-007](../decision-rules.md#br-attachment-007), [BR-attachment-ST-Active-Obsolete](../decision-rules.md#br-attachment-st-active-obsolete) |
| Request Flag | workflow-flag-review-approval / start → validate-flag-type-applicability → validate-requestee → create-flag-request | [BR-attachment-010](../decision-rules.md#br-attachment-010), [BR-attachment-011](../decision-rules.md#br-attachment-011), [BR-attachment-012](../decision-rules.md#br-attachment-012), [BR-attachment-013](../decision-rules.md#br-attachment-013), [BR-attachment-015](../decision-rules.md#br-attachment-015), [BR-attachment-016](../decision-rules.md#br-attachment-016) |
| Grant / Deny Flag | workflow-flag-review-approval / wait-for-review-decision → gateway-decision → grant-flag / deny-flag | [BR-attachment-013](../decision-rules.md#br-attachment-013), [BR-attachment-014](../decision-rules.md#br-attachment-014), [BR-attachment-015](../decision-rules.md#br-attachment-015), [BR-attachment-POLICY-CanSetFlagPolicy](../decision-rules.md#br-attachment-policy-cansetflagpolicy) |

## 2. Flow Diagram

```
[Bug Detail → Attachments Tab]
    │
    ├── [Upload Attachment] → [Upload Dialog] → [Uploaded ✓]
    │
    ├── [View Attachment] → [Attachment Detail / Preview]
    │
    ├── [Edit Attachment] → [Edit Metadata Dialog] → [Updated ✓]
    │
    └── [Delete Attachment] → [Confirm Dialog] → [Deleted ✓]

[Bug Detail → Flags Tab]
    │
    ├── [Request Flag] → [Request Dialog] → [Requested ?]
    │
    ├── [Grant Flag (+)] → [Confirm] → [Granted +]
    │
    ├── [Deny Flag (-)] → [Reason Dialog] → [Denied -]
    │
    └── [Clear Flag] → [Confirm] → [Cleared]
```

## 3. Steps

### Upload Attachment

**Purpose**: Attach a file to a bug. Supports all file types with metadata.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Attach File to Bug #4521                              [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ ┌─ Drop files here or click to browse ──────────────────┐   │
│ │                                                       │   │
│ │         📎  Choose File                               │   │
│ │                                                       │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ Description *                                               │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Stack trace for the crash                                ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ Content Type                                                │
│ ○ Auto-detect  ○ Text  ○ Image  ○ Patch  ○ Other: [____]  │
│                                                             │
│ ☐ This is a patch                                           │
│ ☐ Mark as private (insider only)                            │
│                                                             │
│ Flags (optional)                                            │
│ review? → [Search for reviewer…    ▼]                       │
│                                                             │
│                                          [Cancel] [Attach]  │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: CreateAttachment (service-attachment)
  - Inputs: bugId, file (binary), description, mimeType, isPatch, isPrivate, flags[]
- Query: GetFlagTypes(productId, componentId) (service-attachment) — available flags for attachment

**Validation**:
- File is required
- Description is required
- File size within server limit
- MIME type auto-detected or manually specified
- Private flag only available to insider group members
- Flag requestees must be valid users with flag authority

**State**:
```typescript
interface UploadAttachmentState {
  file: File | null;
  description: string;
  contentType: 'auto' | 'text' | 'image' | 'patch' | 'other';
  customMimeType: string;
  isPatch: boolean;
  isPrivate: boolean;
  flagRequests: Array<{ flagType: string; requestee: string }>;
  isUploading: boolean;
  uploadProgress: number;
  validationErrors: Record<string, string>;
}
```

**Transitions**:
| Action | Target | Condition |
|--------|--------|-----------|
| Cancel | Close dialog | Always |
| Attach | Submit | File selected, description non-empty, valid flags |
| Attach (error) | Stay | Show inline error |

---

### View / Preview Attachment

**Purpose**: View attachment details and preview content (images, text, patches).

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Attachment: stacktrace.log                            [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Bug #4521  |  Uploaded by @josh, 1 hour ago                 │
│ Type: text/plain  |  Size: 12 KB  |  Patch: No             │
│                                                             │
│ ┌─ Preview ──────────────────────────────────────────────┐  │
│ │ Exception: NullPointerException                         │  │
│ │   at SessionRestore.init(SessionRestore.java:42)        │  │
│ │   at Browser.launch(Browser.java:128)                   │  │
│ │   at App.main(App.java:15)                              │  │
│ │ ...                                                     │  │
│ └─────────────────────────────────────────────────────────┘  │
│                                                             │
│ Flags: review? → @reviewer                                  │
│                                                             │
│ [Download] [Edit Metadata] [Delete]                         │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetAttachment(attachmentId) (service-attachment)
- Query: GetAttachmentContent(attachmentId) (service-attachment) — file content for preview

---

### Edit Attachment Metadata

**Purpose**: Update an attachment's description, content type, patch flag, or privacy.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Edit Attachment: stacktrace.log                        [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Description *                                               │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Stack trace for the crash (updated)                       ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ Content Type                                                │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ text/plain                                                ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ ☐ This is a patch                                           │
│ ☐ Mark as private (insider only)                            │
│                                                             │
│                                             [Cancel] [Save] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: UpdateAttachment (service-attachment)

---

### Delete Attachment

**Purpose**: Remove an attachment from a bug. Requires confirmation.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Delete Attachment                                     [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Are you sure you want to delete "stacktrace.log"?           │
│                                                             │
│ This will remove the attachment from Bug #4521.             │
│ This action cannot be undone.                               │
│                                                             │
│                                             [Cancel] [Del.] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: DeleteAttachment (service-attachment)

**Validation**:
- Only the attachment creator, bug assignee, or admin can delete
- Confirmation required

---

### Request Flag

**Purpose**: Request a flag (review, approval, needinfo, etc.) on a bug or attachment.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Request Flag on Bug #4521                             [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Flag Type *                                                 │
│ ┌───────────────────────────────┐                           │
│ │ review                      ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ Request to *                                                │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Search for a user…                                       ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│                                             [Cancel] [Req.] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetFlagTypes(productId, componentId) (service-attachment) — available flag types
- Command: SetFlag (service-attachment) — with state `?` (requested)

**Validation**:
- Flag type must be requestable (`isRequestable = true`)
- Requestee must be a valid user
- Requestee must have permission to grant the flag (checked server-side)
- Flag type must be applicable to the product/component (checked via flaginclusions/flagexclusions)

---

### Grant / Deny Flag

**Purpose**: Respond to a flag request. Grant (+) or deny (-).

**Wireframe** (inline on bug detail):
```
┌─────────────────────────────────────────────────────────────┐
│ Respond to Flag Request                                     │
│                                                             │
│ review? (requested by @josh, assigned to you)               │
│                                                             │
│ ● Grant (+)  ○ Deny (-)                                    │
│                                                             │
│ Comment (optional)                                          │
│ ┌───────────────────────────────────────────────────────────┐│
│ │                                                           ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│                                             [Cancel] [Set]  │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: SetFlag (service-attachment) — with state `+` or `-`
- Only the flag requestee (or someone in the grant group) can set the flag

**Validation**:
- User must be in the flag type's grant group
- Once set to `+` or `-`, the flag can be cleared or re-requested but not changed by non-granters

## 4. Journey State Machine

```typescript
type AttachmentFlagJourneyState =
  | { type: 'viewing-list'; bugId: string }
  | { type: 'uploading'; bugId: string }
  | { type: 'viewing-attachment'; attachmentId: string }
  | { type: 'editing-attachment'; attachmentId: string }
  | { type: 'confirming-delete'; attachmentId: string }
  | { type: 'requesting-flag'; bugId: string; flagType: string }
  | { type: 'responding-flag'; flagId: string; response: '+' | '-' }
  | { type: 'submitting'; action: string }
  | { type: 'complete'; action: string; entityName: string }
  | { type: 'error'; error: Error; action: string };
```

## 5. Error Recovery

| Error Scenario | Recovery Strategy |
|---------------|-------------------|
| File too large | Inline error: "File exceeds maximum size limit ({max})." Prevent upload. |
| Unsupported file type | Inline error: "This file type is not allowed." |
| Upload network failure | Error toast. File retained in input. "Try Again" button. |
| Flag type not applicable | Error: "This flag type is not available for this product/component." |
| Flag requestee not authorized | Error: "This user cannot grant the {flagType} flag." |
| Cannot grant flag (not in grant group) | Error: "You do not have permission to set this flag." |
| Concurrent flag change | Error: "This flag was modified by another user. Please refresh." |
| Delete permission denied | Error: "You do not have permission to delete this attachment." |

## 6. Success States

- **Attachment uploaded**: Toast: "Attachment uploaded." System comment appears in thread. Attachment visible in list.
- **Attachment updated**: Toast: "Attachment updated." Metadata refreshes.
- **Attachment deleted**: Toast: "Attachment deleted." Row removed. System comment appears.
- **Flag requested**: Toast: "Flag {name} requested from {requestee}." Flag shows `?` state.
- **Flag granted**: Toast: "Flag {name} granted." Flag shows `+` state with green indicator.
- **Flag denied**: Toast: "Flag {name} denied." Flag shows `-` state with red indicator.
- **Flag cleared**: Toast: "Flag {name} cleared." Flag removed from display.
- **Screen reader**: "{action} {entity}" for each operation.

## 7. Progress Indication

- **Upload**: Progress bar in upload dialog showing bytes transferred
- **Flag set/request**: Spinner on confirm button
- **Delete**: Spinner on confirm button
- **File preview**: Loading indicator for large file previews

## 8. Drag-and-Drop

Bug detail pages support drag-and-drop file upload:
- Drop zone: entire bug detail content area
- Visual feedback: border highlight on drag-over
- Drop triggers upload dialog with file pre-selected
- Multiple files: opens multiple upload dialogs sequentially

## Workflow & Rules Cross-References

### Step: Upload Attachment

- **Workflow gateway / activity**:
  - `workflow-bug-lifecycle / attachment-added-start → create-attachment` (serviceTask) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:137]
  - `workflow-flag-review-approval / start → validate-flag-type-applicability` (serviceTask — when initial flags are included in the upload) [source: audit-output/workflows/workflow-flag-review-approval.bpmn.yaml:55]
- **Decision rules**:
  - [BR-attachment-001](../decision-rules.md#br-attachment-001) — File size soft limit (`maxattachmentsize` KB)
  - [BR-attachment-002](../decision-rules.md#br-attachment-002) — File size hard limit (`maxlocalattachment` MB)
  - [BR-attachment-003](../decision-rules.md#br-attachment-003) — MIME type must match `LEGAL_CONTENT_TYPES` whitelist
  - [BR-attachment-004](../decision-rules.md#br-attachment-004) — MIME auto-detection and patch forcing for octet-stream
  - [BR-attachment-005](../decision-rules.md#br-attachment-005) — Filename sanitization (basename only, length-truncated)
  - [BR-attachment-006](../decision-rules.md#br-attachment-006) — When `isPatch=true`, content type is forced to `text/plain`
  - [BR-attachment-017](../decision-rules.md#br-attachment-017) — Setting `isPrivate=true` requires insider group membership
  - [BR-attachment-010](../decision-rules.md#br-attachment-010) — Flag type applicability (inclusion/exclusion rules)
  - [BR-attachment-PERM-CreateAttachment](../decision-rules.md#br-attachment-perm-createattachment) — `attachments:create` permission required
  - [BR-attachment-ST-None-Active](../decision-rules.md#br-attachment-st-none-active) — Attachment created in Active state on success
- **Edge cases formalized**:
  - "File too large" → decision-point branch: `create-attachment → reject(file_too_large)`; cite: [BR-attachment-001](../decision-rules.md#br-attachment-001), [BR-attachment-002](../decision-rules.md#br-attachment-002)
  - "Unsupported file type" → decision-point branch: `create-attachment → reject(illegal_mime_type)`; cite: [BR-attachment-003](../decision-rules.md#br-attachment-003)
  - "Upload network failure" → decision-point branch: `create-attachment → reject(network_error)`; cite: [UI-only — no business rule]

### Step: View / Preview Attachment

- **Workflow gateway / activity**: [no clear mapping — manual review needed] (view/preview is a read-only query operation with no workflow gate or decision rule)
- **Decision rules**:
  - [BR-attachment-017](../decision-rules.md#br-attachment-017) — Editing/viewing private attachment requires insider group membership
  - [BR-attachment-POLICY-CanEditPrivateAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditprivateattachmentpolicy) — Insider group required when `isPrivate=true`
- **Edge cases formalized**:
  - (No edge cases in Error Recovery table for this step)

### Step: Edit Attachment Metadata

- **Workflow gateway / activity**: `workflow-bug-lifecycle / update-bug-start` (analogous field-update path for attachment metadata) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:155]
- **Decision rules**:
  - [BR-attachment-POLICY-CanEditAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditattachmentpolicy) — Submitter or `editbugs` group membership for the bug's product
  - [BR-attachment-POLICY-CanEditPrivateAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditprivateattachmentpolicy) — Insider group required when `isPrivate=true`
  - [BR-attachment-017](../decision-rules.md#br-attachment-017) — Setting/editing `isPrivate=true` requires insider group membership for the bug's product
  - [BR-attachment-008](../decision-rules.md#br-attachment-008) — A deleted attachment cannot receive any further modifications
  - [BR-attachment-PERM-UpdateAttachmentMetadata](../decision-rules.md#br-attachment-perm-updateattachmentmetadata) — `attachments:update` permission required
- **Edge cases formalized**:
  - "Edit on deleted attachment" → decision-point branch: command → reject(attachment_deleted); cite: [BR-attachment-008](../decision-rules.md#br-attachment-008)

### Step: Delete Attachment

- **Workflow gateway / activity**:
  - `workflow-bug-lifecycle / end-attachment-added` (reverse cleanup) [source: audit-output/workflows/workflow-bug-lifecycle.bpmn.yaml:168]
  - `workflow-flag-review-approval / start-obsolete → cancel-pending-flags-obsolete → end-obsolete-cascade` (flag cascade cleanup) [source: audit-output/workflows/workflow-flag-review-approval.bpmn.yaml:148]
- **Decision rules**:
  - [BR-attachment-POLICY-CanEditAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditattachmentpolicy) — Submitter or `editbugs` group membership for delete
  - [BR-attachment-008](../decision-rules.md#br-attachment-008) — Deleted attachment cannot receive further modifications
  - [BR-attachment-009](../decision-rules.md#br-attachment-009) — On deletion: clear flags, isObsolete=true, mimeType=text/plain, isPatch=false, binary removed
  - [BR-attachment-007](../decision-rules.md#br-attachment-007) — Obsolete cascade: when `isObsolete` transitions to true, every pending `?` flag is canceled (`reason='obsolete-cascade'`)
  - [BR-attachment-ST-Active-Deleted](../decision-rules.md#br-attachment-st-active-deleted) — Active → Deleted transition; user must be submitter or have `attachments:delete`
  - [BR-attachment-ST-Active-Obsolete](../decision-rules.md#br-attachment-st-active-obsolete) — Active → Obsolete transition cascades cancellation of pending `?` flags
  - [BR-attachment-PERM-DeleteAttachment](../decision-rules.md#br-attachment-perm-deleteattachment) — `attachments:delete` permission required
- **Edge cases formalized**:
  - "Delete permission denied" → decision-point branch: `DeleteAttachment → reject(permission_denied)`; cite: [BR-attachment-PERM-DeleteAttachment](../decision-rules.md#br-attachment-perm-deleteattachment), [BR-attachment-POLICY-CanEditAttachmentPolicy](../decision-rules.md#br-attachment-policy-caneditattachmentpolicy)

### Step: Request Flag

- **Workflow gateway / activity**: `workflow-flag-review-approval / start → validate-flag-type-applicability → gateway-applicable → validate-requestee → gateway-requestee-valid → create-flag-request → notify-requestee` [source: audit-output/workflows/workflow-flag-review-approval.bpmn.yaml:22]
- **Decision rules**:
  - [BR-attachment-010](../decision-rules.md#br-attachment-010) — Flag type applicability (inclusion/exclusion matching)
  - [BR-attachment-011](../decision-rules.md#br-attachment-011) — Inactive flag type cannot have new flag instances created
  - [BR-attachment-012](../decision-rules.md#br-attachment-012) — `targetType` must match the operation (bug vs attachment)
  - [BR-attachment-013](../decision-rules.md#br-attachment-013) — Multiplicable flag constraint (one per type per target)
  - [BR-attachment-015](../decision-rules.md#br-attachment-015) — Grant/request group enforcement per requested status
  - [BR-attachment-016](../decision-rules.md#br-attachment-016) — Requestee validation (enabled, can see bug/attachment, can grant)
  - [BR-attachment-POLICY-FlagTypeApplicabilityPolicy](../decision-rules.md#br-attachment-policy-flagtypeapplicabilitypolicy) — Flag type must be active, applicable, and `targetType` must match
  - [BR-attachment-POLICY-RequesteeVisibilityPolicy](../decision-rules.md#br-attachment-policy-requesteevisibilitypolicy) — Requestee visibility/permission validation
  - [BR-attachment-PERM-SetAttachmentFlagRequest](../decision-rules.md#br-attachment-perm-setattachmentflagrequest) — `flags:request` permission required for `?`
- **Edge cases formalized**:
  - "Flag type not applicable" → decision-point branch: `gateway-applicable → end-not-applicable`; cite: [BR-attachment-010](../decision-rules.md#br-attachment-010)
  - "Flag requestee not authorized" → decision-point branch: `gateway-requestee-valid → end-requestee-invalid`; cite: [BR-attachment-016](../decision-rules.md#br-attachment-016)

### Step: Grant / Deny Flag

- **Workflow gateway / activity**: `workflow-flag-review-approval / wait-for-review-decision → gateway-decision → grant-flag / deny-flag → notify-setter-granted / notify-setter-denied → end-granted / end-denied` [source: audit-output/workflows/workflow-flag-review-approval.bpmn.yaml:98]
- **Decision rules**:
  - [BR-attachment-015](../decision-rules.md#br-attachment-015) — Setting `+`/`-` requires `grantGroupId` membership
  - [BR-attachment-013](../decision-rules.md#br-attachment-013) — Multiplicable flag constraint (one per type per target if `isMultiplicable=false`)
  - [BR-attachment-014](../decision-rules.md#br-attachment-014) — On re-request, `setterId` is NOT updated (preserves original requester for routing)
  - [BR-attachment-POLICY-CanSetFlagPolicy](../decision-rules.md#br-attachment-policy-cansetflagpolicy) — Validates group membership per requested status
  - [BR-attachment-ST-FlagRequested-Granted](../decision-rules.md#br-attachment-st-flagrequested-granted) — Flag transitions `?` → `+` when user is in `grantGroupId`
  - [BR-attachment-ST-FlagRequested-Denied](../decision-rules.md#br-attachment-st-flagrequested-denied) — Flag transitions `?` → `-` when user is in `grantGroupId`
  - [BR-attachment-PERM-SetAttachmentFlagSet](../decision-rules.md#br-attachment-perm-setattachmentflagset) — `flags:set` permission required for `+`/`-`/`X`
- **Edge cases formalized**:
  - "Cannot grant flag (not in grant group)" → decision-point branch: `grant-flag → reject(not_in_grant_group)`; cite: [BR-attachment-015](../decision-rules.md#br-attachment-015)
  - "Concurrent flag change" → decision-point branch: `gateway-decision → reject(concurrent_modification)`; cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule applies)
