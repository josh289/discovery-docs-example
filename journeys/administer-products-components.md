# Journey Spec: Administer Products and Components

## 1. Overview

- **Journey name**: Administer Products and Components
- **Purpose**: Enable administrators to manage the product hierarchy — create, edit, and deactivate products, components, versions, and milestones. This journey covers the CRUD lifecycle of the classification structure that bugs are filed against.
- **Entry points**:
  - "Admin" nav item in sidebar → Administration page
  - Direct URL: `/admin/products`, `/admin/components`, `/admin/versions`, `/admin/milestones`
  - "Manage products" link from product list page
- **Target user/actor**: Administrators with appropriate admin permissions
- **Expected completion time**: 30 seconds (simple edit) to 3 minutes (new product with components)
- **Steps count**: 1–2 steps per entity (CRUD operations)
- **Flow type**: Dashboard CRUD with modal editors

## Workflow & Rules Mapping

| Journey step | Workflow activity | Decision rule(s) |
|--------------|-------------------|------------------|
| Admin Dashboard | — (query-only, no workflow) | [BR-product-PERM-CreateProduct](../decision-rules.md#br-product-perm-createproduct), [BR-product-PERM-CreateComponent](../decision-rules.md#br-product-perm-createcomponent) |
| Create/Edit Product Modal (Create) | `workflow-product-creation-saga` / `start` → `create-product` → `check-product` → `create-first-version` → `check-version` → `create-default-milestone` → `check-milestone` | [BR-product-001](../decision-rules.md#br-product-001), [BR-product-012](../decision-rules.md#br-product-012), [BR-product-ST-new-Active](../decision-rules.md#br-product-st-new-active), [BR-product-PERM-CreateProduct](../decision-rules.md#br-product-perm-createproduct) |
| Create/Edit Product Modal (Edit) | [no clear mapping — manual review needed] (UpdateProduct is a direct command, not part of a workflow) | [BR-product-001](../decision-rules.md#br-product-001), [BR-product-014](../decision-rules.md#br-product-014), [BR-product-ST-Active-Inactive](../decision-rules.md#br-product-st-active-inactive) |
| Create/Edit Component Modal (Create) | `workflow-product-creation-saga` / `create-product` (saga optional step — initial component) or direct `CreateComponent` command | [BR-product-005](../decision-rules.md#br-product-005), [BR-product-008](../decision-rules.md#br-product-008), [BR-product-009](../decision-rules.md#br-product-009), [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy), [BR-product-PERM-CreateComponent](../decision-rules.md#br-product-perm-createcomponent) |
| Create/Edit Component Modal (Edit) | [no clear mapping — manual review needed] (UpdateComponent is a direct command, not part of a workflow) | [BR-product-005](../decision-rules.md#br-product-005), [BR-product-008](../decision-rules.md#br-product-008), [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) |
| Create/Edit Version Modal (Create) | `workflow-product-creation-saga` / `create-first-version` (saga auto-creates "unspecified") or direct `CreateVersion` command | [BR-product-006](../decision-rules.md#br-product-006), [BR-product-002](../decision-rules.md#br-product-002), [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) |
| Create/Edit Version Modal (Edit) | [no clear mapping — manual review needed] (UpdateVersion is a direct command, not part of a workflow) | [BR-product-006](../decision-rules.md#br-product-006), [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) |
| Create/Edit Milestone Modal (Create) | `workflow-product-creation-saga` / `create-default-milestone` (saga auto-creates "---") or direct `CreateMilestone` command | [BR-product-007](../decision-rules.md#br-product-007), [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) |
| Create/Edit Milestone Modal (Edit/Delete) | [no clear mapping — manual review needed] (UpdateMilestone / DeleteMilestone are direct commands, not part of a workflow) | [BR-product-007](../decision-rules.md#br-product-007), [BR-product-004](../decision-rules.md#br-product-004), [BR-product-013](../decision-rules.md#br-product-013), [BR-product-POLICY-NotDefaultMilestonePolicy](../decision-rules.md#br-product-policy-notdefaultmilestonepolicy), [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) |

## 2. Flow Diagram

```
[Administration Page]
    │
    ├── [Products Tab] ──────────────────────────────────────
    │     ├── [+ New Product] → [Create Product Modal] → [Created]
    │     ├── [Edit Product] → [Edit Product Modal] → [Updated]
    │     └── [Product Row] → Expand to show components
    │           ├── [+ New Component] → [Create Component Modal] → [Created]
    │           └── [Edit Component] → [Edit Component Modal] → [Updated]
    │
    ├── [Versions Tab] ─────────────────────────────────────
    │     ├── [+ New Version] → [Create Version Modal] → [Created]
    │     └── [Edit Version] → [Edit Version Modal] → [Updated]
    │
    └── [Milestones Tab] ───────────────────────────────────
          ├── [+ New Milestone] → [Create Milestone Modal] → [Created]
          └── [Edit Milestone] → [Edit Milestone Modal] → [Updated]
```

## 3. Steps

### Admin Dashboard

**Purpose**: Overview of all products with their components, versions, milestones, and group controls.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Administration                                              │
│                                                             │
│ [Products] [Versions] [Milestones] [Groups] [Flag Types]    │
│                                                             │
│ ┌─ Products ─────────────────────────────────── [+ New] ───┐│
│ │ Name            Components  Active  Default Milestone    ││
│ │ ▸ Firefox       12          ✓       120.1               ││
│ │ ▸ Thunderbird   8           ✓       115.0               ││
│ │ ▸ Bugzilla      5           ✓       5.1                 ││
│ └──────────────────────────────────────────────────────────┘│
│                                                             │
│ (expanded product row)                                      │
│ ┌─ Firefox > Components ──────────────────────── [+ New] ─┐│
│ │ Component      Default Assignee  Default QA  Active     ││
│ │ General        @josh             @alex       ✓          ││
│ │ Layout         @sara             —           ✓          ││
│ │ Networking     @dev3             —           ✓          ││
│ └──────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Query: GetProducts (service-product) — product list with metadata
- Query: GetComponents(productId) (service-product) — component list per product

---

### Create/Edit Product Modal

**Purpose**: Create a new product or edit an existing product's settings.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Create Product                                        [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Name *                                                      │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Firefox                                                   ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ Description                                                 │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Mozilla Firefox web browser                               ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ Default Milestone                                            │
│ ┌───────────────────────────────┐                           │
│ │ 120.1                        ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ ☐ Allow UNCONFIRMED status                                  │
│                                                             │
│ ☐ Require milestone on assignment                           │
│                                                             │
│ ☐ Active                                                    │
│                                                             │
│                                             [Cancel] [Save] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: CreateProduct (service-product) — on create
- Command: UpdateProduct (service-product) — on edit

**Validation**:
- Name is required, must be unique
- Name must not contain special characters that would break URL slugs
- Default milestone must exist for the product
- At least one active product must remain in the system

**State**:
```typescript
interface ProductFormState {
  name: string;
  description: string;
  defaultMilestone: string | null;
  allowsUnconfirmed: boolean;
  mustHaveMilestoneOnAccept: boolean;
  isActive: boolean;
  isSubmitting: boolean;
  validationErrors: Record<string, string>;
}
```

---

### Create/Edit Component Modal

**Purpose**: Create or edit a component within a product. Components define sub-areas of a product with default assignee and QA contact.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Create Component in Firefox                           [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Name *                                                      │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ General                                                   ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ Description                                                 │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ General Firefox functionality                             ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ Default Assignee                                            │
│ ┌───────────────────────────────┐                           │
│ │ Search for a user…          ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ Default QA Contact                                          │
│ ┌───────────────────────────────┐                           │
│ │ Search for a user…          ▼│                           │
│ └───────────────────────────────┘                           │
│                                                             │
│ Initial CC List                                             │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ @dev1 ●  @dev2 ●                                       │   │
│ │───────────────────────────────────────────────────────│   │
│ │ Add user…                                              │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│                                             [Cancel] [Save] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: CreateComponent (service-product)
- Command: UpdateComponent (service-product)
- Query: SearchUsers(query) (service-user) — for assignee/QA/CC typeahead

**Validation**:
- Name is required, must be unique within the product
- Default assignee and QA must be valid users with access to the product
- Component names must not conflict with existing components in the same product

---

### Create/Edit Version Modal

**Purpose**: Add or edit a version within a product.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Create Version in Firefox                             [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Version Value *                                             │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ 121.0                                                     ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ ☐ Active                                                    │
│                                                             │
│                                             [Cancel] [Save] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: CreateVersion (service-product)
- Command: UpdateVersion (service-product)

---

### Create/Edit Milestone Modal

**Purpose**: Add or edit a milestone within a product.

**Wireframe**:
```
┌─────────────────────────────────────────────────────────────┐
│ Create Milestone in Firefox                           [✕]   │
│ ────────────────────────────────────────────                 │
│                                                             │
│ Milestone Value *                                           │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ 121.1                                                     ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ Sort Key                                                    │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ 10                                                        ││
│ └───────────────────────────────────────────────────────────┘│
│                                                             │
│ ☐ Active                                                    │
│                                                             │
│                                             [Cancel] [Save] │
└─────────────────────────────────────────────────────────────┘
```

**Data Requirements**:
- Command: CreateMilestone (service-product)
- Command: UpdateMilestone (service-product)
- Command: DeleteMilestone (service-product) — triggers MilestoneDeleted event, which service-bug listens to

**Validation**:
- Milestone value required, must be unique within product
- Sort key must be a positive integer
- Cannot delete a milestone that is set as the default milestone for the product

## 4. Journey State Machine

```typescript
type AdminJourneyState =
  | { type: 'dashboard'; activeTab: 'products' | 'versions' | 'milestones' | 'groups' | 'flag-types' }
  | { type: 'creating-entity'; entityType: 'product' | 'component' | 'version' | 'milestone' }
  | { type: 'editing-entity'; entityType: string; entityId: string }
  | { type: 'submitting'; entityType: string; entityId?: string }
  | { type: 'confirm-delete'; entityType: string; entityId: string; impactCount: number }
  | { type: 'complete'; action: 'created' | 'updated' | 'deleted'; entityName: string }
  | { type: 'error'; error: Error; operation: string };
```

## 5. Error Recovery

| Error Scenario | Recovery Strategy |
|---------------|-------------------|
| Duplicate name | Inline error: "A {entity} with this name already exists." |
| Referential integrity (deactivate product with open bugs) | Warning dialog: "This product has {n} open bugs. Deactivating will prevent new bug creation." Confirm/Cancel. |
| Delete milestone in use | Error: "This milestone is used by {n} bugs and is the default milestone. Update those bugs first." |
| Invalid user for assignee/QA | Inline error: "This user does not have access to this product." |
| Network failure | Error toast with "Try Again". Form data preserved. |
| Concurrent edit | Error: "This was modified by another admin. Please refresh and try again." |

## 6. Success States

- **Entity created**: Toast: "{entityType} '{name}' created." Modal closes. Table row appears.
- **Entity updated**: Toast: "{entityType} '{name}' updated." Modal closes. Table row updates.
- **Entity deactivated**: Toast: "{entityType} '{name}' deactivated." Row shows strikethrough or inactive badge.
- **Entity deleted**: Toast: "{entityType} '{name}' deleted." Row removed from table.
- **Screen reader**: "{entityType} {name} {action}" for each operation.

## 7. Progress Indication

- Table loading: skeleton rows
- Modal submission: spinner on Save button, modal not dismissible
- Inline editing: brief spinner on the cell, then green "Saved" indicator
- Delete confirmation: spinner on "Delete" button

## 8. Bulk Operations

Administrators can select multiple entities for bulk actions:
- Bulk deactivate products/components
- Bulk reassign components (change default assignee/QA)
- Bulk version/milestone management for a product

Each bulk operation shows a confirmation dialog with impact summary before execution.

## Workflow & Rules Cross-References

### Step: Admin Dashboard

- **Workflow gateway / activity**: — (query-only, no workflow activity)
  [source: no workflow — Admin Dashboard issues `GetProducts` and `GetComponents` queries only]
- **Decision rules**:
  - [BR-product-PERM-CreateProduct](../decision-rules.md#br-product-perm-createproduct) — `products:create` permission required (gateway/JWT claims)
  - [BR-product-PERM-CreateComponent](../decision-rules.md#br-product-perm-createcomponent) — `components:create` permission required
  - [BR-product-PERM-UpdateGroupControls](../decision-rules.md#br-product-perm-updategroupcontrols) — `products:manage-groups` permission required
- **Edge cases formalized**: (none — this step is read-only)

### Step: Create/Edit Product Modal (Create — new product)

- **Workflow gateway / activity**: `workflow-product-creation-saga` / `start` → `create-product` → `check-product` → `create-first-version` → `check-version` → `create-default-milestone` → `check-milestone` → `gateway-optional-group` → `gateway-optional-series` → `finalize-product` → `end-success`
  [source: audit-output/workflows/workflow-product-creation-saga.bpmn.yaml]
- **Decision rules**:
  - [BR-product-001](../decision-rules.md#br-product-001) — Product names must be globally unique, checked case-insensitively
  - [BR-product-012](../decision-rules.md#br-product-012) — Product Creation Saga compensates by deactivating the product if version or milestone creation fails; group creation failure is non-fatal
  - [BR-product-002](../decision-rules.md#br-product-002) — Every product must retain at least one version (auto-created by saga; last cannot be deleted)
  - [BR-product-007](../decision-rules.md#br-product-007) — Milestone values must be unique within their owning product
  - [BR-product-014](../decision-rules.md#br-product-014) — Products cannot be hard-deleted; deactivation is the only removal mechanism
  - [BR-product-ST-new-Active](../decision-rules.md#br-product-st-new-active) — Product creation requires globally unique name and valid classification
  - [BR-product-PERM-CreateProduct](../decision-rules.md#br-product-perm-createproduct) — `products:create` permission required
- **Edge cases formalized**:
  - Duplicate product name → decision-point branch: `create-product` → `check-product` (failure) → `compensate-deactivate` cite: [BR-product-001](../decision-rules.md#br-product-001)
  - Saga step 2 (create-first-version) fails → decision-point branch: `create-first-version` → `check-version` (failure) → `compensate-deactivate` cite: [BR-product-012](../decision-rules.md#br-product-012)
  - Saga step 3 (create-default-milestone) fails → decision-point branch: `create-default-milestone` → `check-milestone` (failure) → `compensate-deactivate` cite: [BR-product-012](../decision-rules.md#br-product-012)

### Step: Create/Edit Product Modal (Edit — update existing product)

- **Workflow gateway / activity**: [no clear mapping — manual review needed] (UpdateProduct is a direct command, not orchestrated by a workflow)
- **Decision rules**:
  - [BR-product-001](../decision-rules.md#br-product-001) — Product names must be globally unique, checked case-insensitively (applies on rename)
  - [BR-product-ST-Inactive-Active](../decision-rules.md#br-product-st-inactive-active) — Product reactivation requires the product to exist and be currently inactive
  - [BR-product-014](../decision-rules.md#br-product-014) — Products cannot be hard-deleted; deactivation is the only removal mechanism (applies when toggling `isActive`)
- **Edge cases formalized**:
  - Duplicate product name on rename → decision-point branch: UpdateProduct command rejected cite: [BR-product-001](../decision-rules.md#br-product-001)
  - Concurrent edit (optimistic concurrency conflict) → decision-point branch: command rejected with version conflict, user must refresh cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule; service-product applies same pattern)

### Step: Create/Edit Product Modal (Deactivate product)

- **Workflow gateway / activity**: [no clear mapping — manual review needed] (DeactivateProduct is a direct command; compensation path in `workflow-product-creation-saga` uses DeactivateProduct internally but that is not triggered by this UI action)
- **Decision rules**:
  - [BR-product-014](../decision-rules.md#br-product-014) — Products cannot be hard-deleted; deactivation (setting `isActive = false`) is the only removal mechanism
  - [BR-product-ST-Active-Inactive](../decision-rules.md#br-product-st-active-inactive) — `Active` → `Inactive` on DeactivateProduct, guard is "Product exists and is currently active"
  - [BR-product-NOTIF-ProductCreated](../decision-rules.md#br-product-notif-productcreated) — Notify service-bug and service-notification on product changes (analogous notify path on `ProductDeactivated`)
- **Edge cases formalized**:
  - Referential integrity (deactivate product with open bugs) → decision-point branch: warning dialog presented, admin confirms deactivation; service-bug continues to preserve existing data cite: [BR-product-014](../decision-rules.md#br-product-014)

### Step: Create/Edit Component Modal (Create — new component)

- **Workflow gateway / activity**: `workflow-product-creation-saga` / `create-product` (saga has an optional initial component step per service-product BR 12), or direct `CreateComponent` command (not part of a workflow)
  [source: audit-output/workflows/workflow-product-creation-saga.bpmn.yaml]
- **Decision rules**:
  - [BR-product-005](../decision-rules.md#br-product-005) — Component names must be unique within their owning product
  - [BR-product-008](../decision-rules.md#br-product-008) — Component `defaultAssigneeUserId` must reference an existing, enabled user
  - [BR-product-009](../decision-rules.md#br-product-009) — `defaultQaContactUserId` only accepted when global `useqacontact` parameter is enabled
  - [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) — User must have `editcomponents` group control for the product
  - [BR-product-PERM-CreateComponent](../decision-rules.md#br-product-perm-createcomponent) — `components:create` permission required
  - [BR-user-003](../decision-rules.md#br-user-003) — User-existence validation via `UserListReadModel` (disabled if `disabledText` non-empty)
- **Edge cases formalized**:
  - Duplicate component name within product → decision-point branch: CreateComponent command rejected cite: [BR-product-005](../decision-rules.md#br-product-005)
  - Invalid user for assignee/QA → decision-point branch: CreateComponent command rejected ("This user does not have access to this product.") cite: [BR-product-008](../decision-rules.md#br-product-008)
  - User not found / user disabled → decision-point branch: CreateComponent command rejected, `defaultAssigneeUserId` does not reference an existing enabled user cite: [BR-product-008](../decision-rules.md#br-product-008)

### Step: Create/Edit Component Modal (Edit — update existing component)

- **Workflow gateway / activity**: [no clear mapping — manual review needed] (UpdateComponent is a direct command, not part of a workflow)
- **Decision rules**:
  - [BR-product-005](../decision-rules.md#br-product-005) — Component names must be unique within their owning product (applies on rename)
  - [BR-product-008](../decision-rules.md#br-product-008) — Component `defaultAssigneeUserId` must reference an existing, enabled user
  - [BR-product-009](../decision-rules.md#br-product-009) — `defaultQaContactUserId` gated by `useqacontact` parameter
  - [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) — User must have `editcomponents` group control for the product
- **Edge cases formalized**:
  - Duplicate component name on rename → decision-point branch: UpdateComponent command rejected cite: [BR-product-005](../decision-rules.md#br-product-005)
  - Invalid user for assignee/QA → decision-point branch: UpdateComponent command rejected cite: [BR-product-008](../decision-rules.md#br-product-008)
  - Concurrent edit → decision-point branch: command rejected with version conflict cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule applies)

### Step: Create/Edit Version Modal (Create — new version)

- **Workflow gateway / activity**: `workflow-product-creation-saga` / `create-first-version` (saga auto-creates "unspecified" version), or direct `CreateVersion` command (not part of a workflow for admin-initiated versions)
  [source: audit-output/workflows/workflow-product-creation-saga.bpmn.yaml]
- **Decision rules**:
  - [BR-product-006](../decision-rules.md#br-product-006) — Version values must be unique within their owning product
  - [BR-product-002](../decision-rules.md#br-product-002) — Every product must retain at least one version (MinimumVersionPolicy on delete)
  - [BR-product-POLICY-MinimumVersionPolicy](../decision-rules.md#br-product-policy-minimumversionpolicy) — Product must have more than one version before a version can be deleted
  - [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) — User must have `editcomponents` group control for the product
- **Edge cases formalized**:
  - Duplicate version value within product → decision-point branch: CreateVersion command rejected cite: [BR-product-006](../decision-rules.md#br-product-006)
  - Concurrent edit → decision-point branch: command rejected with version conflict cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule applies)

### Step: Create/Edit Version Modal (Edit — update existing version)

- **Workflow gateway / activity**: [no clear mapping — manual review needed] (UpdateVersion is a direct command, not part of a workflow)
- **Decision rules**:
  - [BR-product-006](../decision-rules.md#br-product-006) — Version values must be unique within their owning product (applies on rename)
  - [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) — User must have `editcomponents` group control for the product
- **Edge cases formalized**:
  - Duplicate version value on rename → decision-point branch: UpdateVersion command rejected cite: [BR-product-006](../decision-rules.md#br-product-006)
  - Concurrent edit → decision-point branch: command rejected with version conflict cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule applies)

### Step: Create/Edit Milestone Modal (Create — new milestone)

- **Workflow gateway / activity**: `workflow-product-creation-saga` / `create-default-milestone` (saga auto-creates "---" milestone), or direct `CreateMilestone` command (not part of a workflow for admin-initiated milestones)
  [source: audit-output/workflows/workflow-product-creation-saga.bpmn.yaml]
- **Decision rules**:
  - [BR-product-007](../decision-rules.md#br-product-007) — Milestone values must be unique within their owning product
  - [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) — User must have `editcomponents` group control for the product
- **Edge cases formalized**:
  - Duplicate milestone value within product → decision-point branch: CreateMilestone command rejected cite: [BR-product-007](../decision-rules.md#br-product-007)
  - Concurrent edit → decision-point branch: command rejected with version conflict cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule applies)

### Step: Create/Edit Milestone Modal (Edit — update existing milestone)

- **Workflow gateway / activity**: [no clear mapping — manual review needed] (UpdateMilestone is a direct command, not part of a workflow)
- **Decision rules**:
  - [BR-product-007](../decision-rules.md#br-product-007) — Milestone values must be unique within their owning product (applies on rename)
  - [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) — User must have `editcomponents` group control for the product
- **Edge cases formalized**:
  - Duplicate milestone value on rename → decision-point branch: UpdateMilestone command rejected cite: [BR-product-007](../decision-rules.md#br-product-007)
  - Concurrent edit → decision-point branch: command rejected with version conflict cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule applies)

### Step: Delete Milestone

- **Workflow gateway / activity**: [no clear mapping — manual review needed] (DeleteMilestone is a direct command, not part of a workflow)
- **Decision rules**:
  - [BR-product-004](../decision-rules.md#br-product-004) — Product's current default milestone cannot be deleted (NotDefaultMilestonePolicy); a different default must be set first
  - [BR-product-013](../decision-rules.md#br-product-013) — When a non-default milestone is deleted, bugs referencing it must be reassigned to the product's default milestone by service-bug
  - [BR-product-POLICY-NotDefaultMilestonePolicy](../decision-rules.md#br-product-policy-notdefaultmilestonepolicy) — Milestone must not be the product's current default milestone to be deleted
  - [BR-product-POLICY-CanAdminProductPolicy](../decision-rules.md#br-product-policy-canadminproductpolicy) — User must have `editcomponents` group control for the product
  - [BR-product-ST-MilestoneActive-Deleted](../decision-rules.md#br-product-st-milestoneactive-deleted) — Milestone soft-deletion blocked if it is the product's current default
- **Edge cases formalized**:
  - Delete milestone that is the default milestone → decision-point branch: DeleteMilestone command rejected (NotDefaultMilestonePolicy) cite: [BR-product-004](../decision-rules.md#br-product-004)
  - Delete milestone in use by bugs → decision-point branch: DeleteMilestone succeeds, emits `MilestoneDeleted` with `defaultMilestoneId`; service-bug reassigns bugs via `MilestoneDeletedHandler` cite: [BR-product-013](../decision-rules.md#br-product-013)
  - Concurrent edit → decision-point branch: command rejected with version conflict cite: [BR-bug-015](../decision-rules.md#br-bug-015) (analogous optimistic concurrency rule applies)
