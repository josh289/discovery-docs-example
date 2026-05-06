# Permission Matrix

> Audit item #8. Synthesized from all SERVICE_SPEC.md Authorization sections
> + ADR-006 + interaction-map.md. All citations are inline.

---

## Roles Inventory

| Role | Source | Current Implementation | Cross-Service State Required |
|------|--------|----------------------|------------------------------|
| **reporter** | Implicit from `CanSeeBugPolicy` (reporterAccessible) and `CanChangeFieldPolicy` ("reporters can change severity and priority on their own bugs") | Bug-row attribute: `user.id === bug.reporterId` | None — reporterId is immutable, stored in BugAggregate |
| **assignee** | Implicit from `CanChangeFieldPolicy` ("Only the current assignee can reassign") and `CanSeeBugPolicy` | Bug-row attribute: `user.id === bug.assignedTo` | None — assignedTo stored in BugAggregate |
| **qa_contact** | Implicit from `CanChangeFieldPolicy` ("Only the QA contact can change the QA contact") and `CanCommentOnBugPolicy` ("QA contact") | Bug-row attribute: `user.id === bug.qaContact` | None — stored in BugAggregate |
| **cc_member** | Implicit from `CanSeeBugPolicy` (ccListAccessible) and `CanCommentOnBugPolicy` ("in the CC list") | Bug-row collection membership: `user.id ∈ bug.ccList` | None — CC list stored in BugAggregate |
| **editbugs_group_member** | ADR-006 (editbugs control flag), `CanChangeFieldPolicy`, `CanCommentOnBugPolicy`, `CanSeeBugPolicy` | Group membership check: user's groups intersect bug-product's groups where `editbugs=true` | `UserGroupMembershipReadModel` (from `user.Events.GroupMemberAdded/Removed`) + `ProductGroupControlsReadModel` (from `product.Events.GroupControlsUpdated`) |
| **canconfirm_group_member** | ADR-006 (canconfirm control flag), `CanConfirmBugPolicy` | Group membership check: user's groups have `canconfirm=true` for bug's product | `UserGroupMembershipReadModel` + `ProductGroupControlsReadModel` |
| **canedit_group_member** | ADR-006 (canedit control flag), `RemoveCc` command description | Group membership check: user's groups have `canedit=true` for bug's product | `UserGroupMembershipReadModel` + `ProductGroupControlsReadModel` |
| **bugs_admin** | `UpdateStatusWorkflowConfig` actor ("Administrators with `bugs:admin_workflow` permission only") | JWT claim: `bugs:admin_workflow` permission string | None — enforced at gateway (Layer 1) |
| **timetracking_user** | `UpdateTimetracking` actor ("Users with `bugs:edit_timetracking` permission only") | JWT claim: `bugs:edit_timetracking` permission string | None — enforced at gateway (Layer 1) |
| **insider** | `IsInsiderPolicy` and `CanSeePrivateCommentsPolicy` in service-comment spec | Group membership: user is member of the insider group; also linked to `comments:view:private` permission | `service-user` read models or `comments:view:private` JWT claim [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:207] |
| **comment_tagger** | `IsCommentTaggerPolicy` | Group membership: user is member of `comment_taggers_group` site parameter | `service-user` read models for group membership [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:208] |
| **comment_admin** | `CanSuppressCommentPolicy` and `SuppressComment` command | JWT claim: `comments:suppress` permission string | None — enforced at gateway (Layer 1) |
| **authenticated_user** | All queries marked `(authenticated)` and all commands requiring any permission string | JWT authentication (valid token) | None — standard JWT validation |
| **entry_group_member** | `CanEnterProductPolicy` in ADR-006 | Group membership: user's groups have `entry=true` for the target product | `UserGroupMembershipReadModel` + `ProductGroupControlsReadModel` |
| **anonymous** | Not referenced in current specs — all operations require `(authenticated)` at minimum | N/A — no unauthenticated access path exists | N/A |

---

## Master Matrix

| Role | Resource | Action | Layer 1 (permission string) | Layer 2 (policy / check) | Source |
|------|----------|--------|----------------------------|--------------------------|--------|
| authenticated_user | bug | create | `bugs:create` | `CanEnterProductPolicy` (entry group check for target product) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:236] |
| reporter | bug | update (severity, priority) | `bugs:update` | `CanChangeFieldPolicy` — reporters may set severity and priority on their own bugs only | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:237] |
| assignee | bug | update (reassign) | `bugs:update` | `CanChangeFieldPolicy` — only the current assignee can reassign | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276] |
| qa_contact | bug | update (QA contact) | `bugs:update` | `CanChangeFieldPolicy` — only the QA contact can change the QA contact | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276] |
| editbugs_group_member | bug | update (most fields) | `bugs:update` | `CanChangeFieldPolicy` — users with `bugs:update` for the product can change most fields | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276] |
| authenticated_user | bug | transition status | `bugs:transition_status` | `ValidStatusTransitionPolicy` + `NoOpenBlockersPolicy` (for FIXED resolution) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:273] |
| authenticated_user | bug | set resolution | `bugs:transition_status` | `CanChangeFieldPolicy` | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:274] |
| authenticated_user | bug | assign | `bugs:update` | `CanChangeFieldPolicy` + `StrictIsolationPolicy` (when strict isolation enabled) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:240,277] |
| authenticated_user | bug | add dependency | `bugs:update` | `DependencyLoopPolicy` + `BugVisibilityPolicy` (target bug must be visible) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:241,278] |
| authenticated_user | bug | remove dependency | `bugs:update` | `BugVisibilityPolicy` | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:242,278] |
| authenticated_user | bug | add CC | `bugs:update` | `StrictIsolationPolicy` (when strict isolation enabled) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:244,277] |
| authenticated_user | bug | remove CC (self) | `bugs:update` | — (always allowed for self) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:245] |
| canedit_group_member | bug | remove CC (others) | `bugs:update` | `CanEditProductPolicy` (canedit flag check) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:53] |
| authenticated_user | bug | add keyword | `bugs:update` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:246] |
| authenticated_user | bug | remove keyword | `bugs:update` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:247] |
| authenticated_user | bug | set alias | `bugs:update` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:248] |
| authenticated_user | bug | mark as duplicate | `bugs:transition_status` | `BugVisibilityPolicy` (target bug must be visible) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:286,278] |
| authenticated_user | bug | change product | `bugs:update` | `CanChangeFieldPolicy` + field revalidation | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:250,276] |
| authenticated_user | bug | add group restriction | `bugs:update` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:251] |
| authenticated_user | bug | remove group restriction | `bugs:update` | — (mandatory groups cannot be removed per business rule #11) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:252] |
| authenticated_user | bug | add see-also | `bugs:update` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:253] |
| authenticated_user | bug | remove see-also | `bugs:update` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:254] |
| authenticated_user | bug | update custom field | `bugs:update` | `MandatoryFieldPolicy` (rejects clearing mandatory fields) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:255,278] |
| timetracking_user | bug | update time-tracking | `bugs:edit_timetracking` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:256] |
| bugs_admin | workflow-config | update | `bugs:admin_workflow` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:257] |
| authenticated_user | bug | read (get) | `(authenticated)` | `CanSeeBugPolicy` | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:258,279] |
| authenticated_user | bug | search | `(authenticated)` | `CanSeeBugPolicy` (results filtered by visibility) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:259,279] |
| authenticated_user | bug | read history | `(authenticated)` | `CanSeeBugPolicy` + time-tracking entries hidden without `bugs:edit_timetracking` | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:260] |
| authenticated_user | bug | read fields | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:261] |
| authenticated_user | bug | read legal values | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:262] |
| authenticated_user | bug | read valid transitions | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:263] |
| authenticated_user | bug | read creation statuses | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:264] |
| authenticated_user | bug | read last visit | `(authenticated)` | — (requesting user only) | [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:265] |
| reporter, assignee, qa_contact, cc_member, editbugs_group_member | comment | create | `comments:create` | `CanCommentOnBugPolicy` | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:200,206] |
| insider | comment | set privacy | `comments:update:privacy` | `IsInsiderPolicy` | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:201,207] |
| comment_tagger | comment | add tag | `comments:tag:add` | `IsCommentTaggerPolicy` | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:202,208] |
| comment_tagger | comment | remove tag | `comments:tag:remove` | `IsCommentTaggerPolicy` | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:203,209] |
| comment_admin | comment | suppress | `comments:suppress` | `CanSuppressCommentPolicy` | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:204,210] |
| authenticated_user | comment | read | `comments:read` | `CanSeePrivateCommentsPolicy` (query-side filter) | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:205,211] |
| authenticated_user | comment | read bug comments | `comments:read` | `CanSeePrivateCommentsPolicy` (query-side filter) | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:206,211] |
| insider | comment | search tags | `comments:tag:search` | — | [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:207] |
| canconfirm_group_member | bug | confirm | `bugs:update` (implied) | `CanConfirmBugPolicy` — checks if user's group has `canconfirm` for the bug's product, OR user is reporter | [source: output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md:93] |
| users_admin | user | create | `users:create` | — | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:219] |
| self_or_admin | user | update profile | `users:update` | `OwnsProfilePolicy` — `userId` must match target; admin with `users:update` bypasses | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:220,244] |
| self | user | change email | `users:email` | `OwnsProfilePolicy` + `NotDisabledPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:221,246] |
| editusers_group_member | user | disable | `users:disable` | `CanDisableUserPolicy` — must hold `users:disable` AND be `editusers` group member; cannot disable last admin | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:222,252] |
| editusers_group_member | user | enable | `users:enable` | `CanEnableUserPolicy` — must hold `users:enable` AND be `editusers` group member | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:223,253] |
| self | user | change password | `users:password` | `OwnsProfilePolicy` + `NotDisabledPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:224,245] |
| self | user-setting | update | `users:update` | `OwnsProfilePolicy` + `NotDisabledPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:225,247] |
| self | email-preferences | update | `users:update` | `OwnsProfilePolicy` + `NotDisabledPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:226,248] |
| groups_admin | group | create | `groups:create` | `CanAdministerGroupsPolicy` — must belong to `admin` group or have `creategroups` system group | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:227,254] |
| groups_admin | group | update | `groups:update` | `CanAdministerGroupsPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:228,255] |
| group_blesser | group | add member | `groups:bless` | `CanBlessGroupPolicy` — user must have `BLESS` grant on the target group | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:229,258] |
| group_blesser | group | remove member | `groups:bless` | `CanBlessGroupPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:230,259] |
| groups_admin | group-inheritance | add | `groups:update` | `CanAdministerGroupsPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:231,256] |
| groups_admin | group-inheritance | remove | `groups:update` | `CanAdministerGroupsPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:232,257] |
| self | api-key | create | `apikeys:create` | `OwnsProfilePolicy` — `userId` must match target | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:233,249] |
| self | api-key | revoke | `apikeys:revoke` | `OwnsKeyPolicy` — key must belong to caller; admin with `apikeys:revoke` may revoke any | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:234,250] |
| self | api-key | list | `(authenticated)` | `OwnsProfilePolicy` — caller can only list own keys | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:251] |
| users_admin | user | list | `users:admin` | — | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:235] |
| users_admin, self | user | read (get) | `users:admin` (or self via Layer 2) | `CanViewUserPolicy` — when `usevisibilitygroups` enabled, requester needs `GROUP_VISIBLE` grant on a group target belongs to, or is target | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:236,260] |
| users_admin, self | user | read by email | `users:admin` (or self via Layer 2) | `CanViewUserPolicy` | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:237,261] |
| anonymous | user | authenticate | `(none — public)` | — | [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:238] |
| products_admin | product | create | `products:create` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:420] |
| products_admin | product | update | `products:update` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:421] |
| products_admin | product | deactivate | `products:update` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:422] |
| products_admin | product | set default milestone | `products:update` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:423] |
| group_controls_admin | product-group-controls | update | `products:manage-groups` | membercontrol/othercontrol matrix validation (NA ≤ SHOWN ≤ DEFAULT ≤ MANDATORY) | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:424,460] |
| editcomponents_group_member | component | create | `components:create` | `CanAdminProductPolicy` — user must have `editcomponents` group control for product | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:425,451] |
| editcomponents_group_member | component | update | `components:update` | `CanAdminProductPolicy` | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:426,452] |
| editcomponents_group_member | component | delete | `components:delete` | `CanAdminProductPolicy` + `MinimumComponentPolicy` (product must have >1 component) | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:427,453] |
| editcomponents_group_member | version | create | `versions:create` | `CanAdminProductPolicy` | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:428,454] |
| editcomponents_group_member | version | update | `versions:update` | `CanAdminProductPolicy` | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:429,455] |
| editcomponents_group_member | version | delete | `versions:delete` | `CanAdminProductPolicy` + `MinimumVersionPolicy` (product must have >1 version) | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:430,456] |
| editcomponents_group_member | milestone | create | `milestones:create` | `CanAdminProductPolicy` | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:431,457] |
| editcomponents_group_member | milestone | update | `milestones:update` | `CanAdminProductPolicy` | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:432,458] |
| editcomponents_group_member | milestone | delete | `milestones:delete` | `CanAdminProductPolicy` + `NotDefaultMilestonePolicy` | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:433,459] |
| classifications_admin | classification | create | `classifications:create` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:434] |
| classifications_admin | classification | update | `classifications:update` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:435] |
| classifications_admin | classification | delete | `classifications:delete` | `DefaultClassificationProtectionPolicy` — classification id `'1'` cannot be deleted | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:436,461] |
| authenticated_user | product | read (get) | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:437] |
| authenticated_user | product | list | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:438] |
| authenticated_user | product | check access | `(authenticated)` | `ProductAccessPolicy` — if any group has `entry=true`, user must be member of an entry group; otherwise open | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:445,462] |
| group_controls_admin | product-group-controls | read | `products:manage-groups` | — | [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:444] |
| authenticated_user | attachment | create | `attachments:create` | `CanEditPrivateAttachmentPolicy` — when `isPrivate=true`, user must be in insider group for bug's product | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:347,377] |
| attachment_submitter, editbugs_group_member | attachment | update metadata | `attachments:update` | `CanEditAttachmentPolicy` — submitter or `editbugs` for product; private attachments require insider | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:348,374] |
| attachment_submitter, editbugs_group_member | attachment | mark obsolete | `attachments:update` | `CanEditAttachmentPolicy` | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:349,375] |
| attachment_submitter, editbugs_group_member | attachment | delete | `attachments:delete` | `CanEditAttachmentPolicy` | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:350,376] |
| flag_request_group_member | attachment-flag | set request (?) | `flags:request` | `CanSetFlagPolicy` (`?` requires `requestGroupId` membership; null group = anyone) + `FlagTypeApplicabilityPolicy` + `RequesteeVisibilityPolicy` + `MultiplicableFlagPolicy` | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:351,379,381,383,385] |
| flag_grant_group_member | attachment-flag | set grant/deny (+/-/X) | `flags:set` | `CanSetFlagPolicy` (`+`/`-` requires `grantGroupId`; `X` requires original setter or grant/request group) + `FlagTypeApplicabilityPolicy` + `RequesteeVisibilityPolicy` + `MultiplicableFlagPolicy` | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:352,379,381,383,385] |
| flag_request_group_member | bug-flag | set request (?) | `flags:request` | `CanSetFlagPolicy` + `FlagTypeApplicabilityPolicy` (`targetType='bug'`) + `RequesteeVisibilityPolicy` + `MultiplicableFlagPolicy` | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:353,380,382,384,386] |
| flag_grant_group_member | bug-flag | set grant/deny (+/-/X) | `flags:set` | `CanSetFlagPolicy` + `FlagTypeApplicabilityPolicy` (`targetType='bug'`) + `RequesteeVisibilityPolicy` + `MultiplicableFlagPolicy` | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:354,380,382,384,386] |
| flag_grant_group_member | bug-flag | clear | `flags:set` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:355] |
| flag_types_admin | flag-type | create | `flag-types:create` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:356] |
| flag_types_admin | flag-type | update | `flag-types:update` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:357] |
| flag_types_admin | flag-type | update inclusions | `flag-types:update` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:358] |
| flag_types_admin | flag-type | update exclusions | `flag-types:update` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:359] |
| flag_types_admin | flag-type | deactivate | `flag-types:update` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:360] |
| authenticated_user | attachment | read (bug attachments) | `(authenticated)` | query-side privacy filtering (private attachments visible only to insider/submitter) | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:361] |
| authenticated_user | attachment | read data | `(authenticated)` | query-side privacy filtering | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:362] |
| authenticated_user | attachment | read detail | `(authenticated)` | query-side privacy filtering | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:363] |
| authenticated_user | flag-type | list available | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:364] |
| authenticated_user | bug-flag | read | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:365] |
| authenticated_user | flag | query | `(authenticated)` | — | [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:366] |
| authenticated_user | bug | search (execute) | `search:execute` | Bug security filter — Elasticsearch results filtered by user's group membership, reporter/CC identity, accessibility flags | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:147,176] |
| authenticated_user | bug | quicksearch | `search:execute` | Bug security filter | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:148,177] |
| authenticated_user | recent-searches | read | `search:execute` | — | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:149] |
| authenticated_user | saved-search | create | `search:saved-search:create` | — | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:150] |
| saved_search_owner, shared_group_member | saved-search | read | `search:saved-search:read` | `SearchVisibilityPolicy` — owner or member of shared group | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:151,175] |
| authenticated_user | saved-search | list | `search:saved-search:read` | — | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:152] |
| saved_search_owner | saved-search | update | `search:saved-search:update` | `OwnsSearchPolicy` — only creator may update | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:153,168] |
| saved_search_owner | saved-search | delete | `search:saved-search:delete` | `OwnsSearchPolicy` + `NotUsedByReportPolicy` (no scheduled report references it) | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:154,169,170] |
| saved_search_owner | saved-search | share | `search:saved-search:share` | `OwnsSearchPolicy` + `GroupExistsPolicy` (target group must exist and be active) | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:155,171,172] |
| saved_search_owner | saved-search | unshare | `search:saved-search:share` | `OwnsSearchPolicy` | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:156,173] |
| saved_search_owner, shared_group_member | saved-search | toggle footer link | `search:saved-search:footer` | `SearchVisibilityPolicy` | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:157,174] |
| authenticated_user | chart | read data | `search:chart:view` | `SeriesVisibilityPolicy` — chart series filtered per user group membership | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:158,178] |
| authenticated_user | chart | read visible series | `search:chart:view` | `SeriesVisibilityPolicy` | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:159,178] |
| authenticated_user | scheduled-report (search) | create | `search:report:create` | `SavedSearchVisiblePolicy` — referenced saved search must be visible to creator | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:160,179] |
| scheduled_report_owner | scheduled-report (search) | update | `search:report:update` | `OwnsReportPolicy` — only owner may update | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:161,180] |
| scheduled_report_owner | scheduled-report (search) | delete | `search:report:delete` | `OwnsReportPolicy` | [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:162,181] |
| authenticated_user | scheduled-report (notif) | create | `notifications:schedule:create` | — | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:220] |
| schedule_owner | scheduled-report (notif) | update | `notifications:schedule:update` | `OwnsSchedulePolicy` — caller userId must match `ownerUserId` | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:221,239] |
| schedule_owner | scheduled-report (notif) | delete | `notifications:schedule:delete` | `OwnsSchedulePolicy` | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:222,240] |
| schedule_owner | scheduled-report (notif) | read | `notifications:schedule:read` | `OwnsSchedulePolicy` (single-report read) | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:223,224,241] |
| self | notification-preferences | update | `notifications:preferences:update` | — | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:225] |
| self | user-watcher | set | `notifications:preferences:update` | — | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:226] |
| self | notification-preferences | read | `notifications:preferences:read` | `OwnsPreferencesPolicy` — caller can only read own preferences | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:227,242] |
| self | watcher-list | read | `notifications:preferences:read` | — | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:228] |
| self | email-delivery-status | read | `notifications:delivery:read` | `OwnsDeliveryLogPolicy` — caller can only read own delivery log | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:229,243] |
| global_watcher | bug-mail | receive (recipient computation) | `notifications:global-watch` (implicit, not gateway-enforced) | applied at recipient resolution; user receives all bug mail | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:231] |
| (recipient) | notification | deliver | (event-driven, no Layer 1) | `RecipientNotDisabledPolicy` + `BugNotIgnoredPolicy` + `CanSeeBugPolicy` + `WantsBugMailPolicy` — programmatic checks inside `NotificationRecipientResolver` | [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:244,245,246,247] |

---

## Gaps & Inconsistencies

### Audit Drift (corrected in this revision)

- **`bugs:transition_status` was previously documented as `bugs:update`** [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:273,274,286]. Earlier revisions of this matrix listed `bugs:update` for `TransitionBugStatus`, `SetBugResolution`, and `MarkBugDuplicate`. The actual SERVICE_SPEC declares a dedicated `bugs:transition_status` permission for these three commands. Rows 39, 40, and 50 above have been corrected; downstream consumers (gateway config, role-to-permission seed data) should re-check whether they were affected by the earlier drift.

### Missing Layer 2 Policies

- **`bugs:admin_workflow` has no Layer 2 policy** [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:257]. The `UpdateStatusWorkflowConfig` command is gated only by Layer 1 (`bugs:admin_workflow`). Any user with that JWT claim can modify the global workflow configuration without any domain-level business check. This is a high-privilege operation (adding/removing statuses, transitions) that should have at minimum an `IsAdminPolicy` or `CanModifyWorkflowPolicy` at Layer 2.

- **`bugs:edit_timetracking` has no Layer 2 policy** [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:256]. The `UpdateTimetracking` command relies solely on gateway-enforced `bugs:edit_timetracking`. No domain policy verifies the user should be allowed to edit time-tracking for this specific bug (e.g., only the assignee or a timetracking group member).

- **Many `bugs:update` commands have no Layer 2 policy** — `AddKeyword`, `RemoveKeyword`, `SetBugAlias`, `AddSeeAlso`, `RemoveSeeAlso`, `AddBugGroup`, `RemoveBugGroup` all carry `bugs:update` at Layer 1 but list no Layer 2 policy [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:246–254]. A user with `bugs:update` on any product can modify these fields on any bug regardless of product-level group controls. This may be intentional (the gateway check is sufficient), but it means the `editbugs` product-level control flag does not gate these specific operations.

- **`CreateUser` has no Layer 2 policy** [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:219]. `users:create` is the sole guard on account creation. There is no domain check that the requester belongs to an admin group or has any specific role. This is consistent with self-service signup but is not documented as such in the spec.

- **All `service-product` admin commands except component/version/milestone CRUD have no Layer 2 policy** [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:420–423]. `CreateProduct`, `UpdateProduct`, `DeactivateProduct`, and `SetDefaultMilestone` each rely solely on `products:create` or `products:update`. There is no domain-side check that the requester is in the `editproducts` admin group. This is asymmetric with component/version/milestone commands, which all require `CanAdminProductPolicy` at Layer 2.

- **All `service-product` classification commands have no Layer 2 policy** [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:434,435,436] except `DeleteClassification` (which only checks the default-classification protection). `CreateClassification` and `UpdateClassification` are gateway-only.

- **All `service-attachment` flag-type admin commands have no Layer 2 policy** [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:356–360]. `CreateFlagType`, `UpdateFlagType`, `UpdateFlagTypeInclusions`, `UpdateFlagTypeExclusions`, and `DeactivateFlagType` are guarded only by `flag-types:create` / `flag-types:update`. Any user with that JWT claim can edit the flag type catalog without a domain-level admin check.

- **`service-search` `ListSavedSearches`, `GetRecentSearches`, `CreateSavedSearch`, `Quicksearch`, `ExecuteSearch` have no per-user Layer 2 ownership policy** [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:147–152]. The bug security filter applies to query results but not to which saved-search records the user can list. A user with `search:saved-search:read` can list every saved search (visibility filtering happens only on `GetSavedSearch`/`ToggleFooterLink` via `SearchVisibilityPolicy`).

- **`service-notification` `GetScheduledReports` (list) has no Layer 2 policy** [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:223,239–241]. Only `GetScheduledReport` (single) is guarded by `OwnsSchedulePolicy`. The list query is implicitly scoped to the caller in spec prose but does not declare an `OwnsSchedulePolicy` check.

- **`service-notification` `GetWatcherList` has no Layer 2 policy** [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:228]. The `OwnsPreferencesPolicy` covers only `GetNotificationPreferences`; watcher list reads are gateway-only.

### Layer-1/Layer-2 Mismatches

- **`bugs:update` is overly broad** — 15+ distinct commands share the single `bugs:update` Layer 1 string [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:237–254]. The Layer 2 `CanChangeFieldPolicy` only narrows behavior for field-specific operations (reporter severity, assignee reassignment). Commands like `AddKeyword`, `SetBugAlias`, `AddSeeAlso` pass through with no Layer 2 check at all. A compromised JWT with `bugs:update` grants far more capability than any single role needs.

- **`CanChangeFieldPolicy` is a single policy for heterogeneous checks** [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276]. It simultaneously checks: reporter → severity/priority, assignee → reassign, QA contact → QA contact change, and editbugs group → most fields. A reviewer cannot tell from the policy name alone what it gates. Field-specific policies (e.g., `CanChangeSeverityPolicy`, `CanReassignPolicy`) would be more auditable.

- **`CanSetFlagPolicy` overloads three distinct checks under one name** [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:379]. It selects between `requestGroupId` (for `?`), `grantGroupId` (for `+`/`-`), and "original setter or grant/request group" (for `X`) based on the requested status. This is similar to the `CanChangeFieldPolicy` anti-pattern in service-bug — the policy's effective rule depends on a command argument, making it harder to audit.

- **`UpdateScheduledReport` exists in two services with different Layer 1 strings** — `search:report:update` in service-search [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:161] and `notifications:schedule:update` in service-notification [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:221]. They guard different aggregates (saved-search reports vs whine reports), but the duplicate command name and overlapping concept ("scheduled report") creates confusion. A reviewer could mistake one for the other.

- **`CreateScheduledReport` likewise exists in both services** with the same naming clash [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:160, output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:220].

### Duplicated Authorization Logic Across Services

- **`editbugs` group control is checked in both service-bug and service-comment** — `CanChangeFieldPolicy` in service-bug [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276] and `CanCommentOnBugPolicy` in service-comment [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:206] both consult the product's `editbugs` group control flag. Both depend on the same `ProductGroupControlsReadModel`. If service-comment does not maintain this read model, it makes a synchronous `GetBug` query to service-bug instead [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:240]. This creates a dual-path dependency: if the synchronous query is used, it bypasses the eventual-consistency boundary that ADR-006 was designed to avoid.

- **`editbugs` group is also consulted in service-attachment** — `CanEditAttachmentPolicy` checks `editbugs` group membership for the bug's product [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:374]. This is a third service replicating the same pattern. Combined with service-bug and service-comment, the `editbugs` flag is now read in three independent local projections, each subject to its own propagation lag.

- **`UserGroupMembershipReadModel` is projected by at least four services** — service-bug, service-comment, service-attachment, and service-product all maintain a local copy projected from `user.Events.GroupMemberAdded`/`GroupMemberRemoved` [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:277,278, output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:388, output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:496]. Each projection drifts independently. A user removed from a group may still appear authorized in one service's projection while already removed from another's.

- **`OwnsProfilePolicy`-style "caller is target" checks are duplicated across services** — `OwnsProfilePolicy` (service-user) [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:244], `OwnsSchedulePolicy` (service-notification) [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:239], `OwnsPreferencesPolicy` (service-notification) [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:242], `OwnsDeliveryLogPolicy` (service-notification) [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:243], `OwnsKeyPolicy` (service-user) [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:250], `OwnsSearchPolicy` (service-search) [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:168], and `OwnsReportPolicy` (service-search) [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:180] all implement the same essential check: "caller's userId equals the target resource's owner field." Seven near-identical policies across four services.

### Orphan Permissions

- **`comments:view:private` is not tied to a specific command** [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:199]. It is described as checked "at query time" rather than declared on any `@Command`. This permission exists outside the standard Layer 1 contract enforcement model. It is unclear whether the gateway enforces it or if it is only checked inside query handlers.

- **`notifications:global-watch` is not attached to any command or query** [source: output/phase-5-specification/specs/service-notification/SERVICE_SPEC.md:231]. The spec explicitly notes it is "checked during recipient computation" inside `NotificationRecipientResolver`, not gateway-enforced. This is a similar pattern to `comments:view:private` — a capability claim with no contract-level enforcement.

- **`creategroups` system group is referenced but not defined** [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:254–257]. `CanAdministerGroupsPolicy` accepts membership in the `creategroups` system group as an alternative to the `admin` group, but the user spec does not document where this group is created or how it is provisioned.

- **`editusers` group is referenced but not defined** [source: output/phase-5-specification/specs/service-user/SERVICE_SPEC.md:252,253]. `CanDisableUserPolicy` and `CanEnableUserPolicy` require `editusers` membership, but the spec does not specify the group's lifecycle or who can manage its membership.

- **`editproducts` admin group is implied but absent** — there is no Layer 2 policy gating `CreateProduct`, `UpdateProduct`, or classification CRUD against an admin-style group. Bugzilla traditionally has an `editproducts` group; the spec [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:447–462] does not reference it at all.

- **`CanEnterProductPolicy` and `CanEditProductPolicy` are declared in ADR-006** [source: output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md:93–96] but do not appear in the service-bug SERVICE_SPEC Layer 2 table [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:274–279]. They may be invoked internally by `CanChangeFieldPolicy` and `CreateBug` validation, but this is not documented in the spec.

### Cross-Service Stale-Read Risk

- **Every Layer 2 policy that depends on a projected read model has eventual-consistency exposure**. ADR-006 acknowledges this: "when a user is added to or removed from a group, there is a propagation delay before service-bug's `UserGroupMembershipReadModel` reflects the change" [source: output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md:124]. The mitigation cited is "group membership changes are infrequent and typically admin-driven" with "sub-second" projection lag.

- **`ProductGroupControlsReadModel` has the same risk** — `product.Events.GroupControlsUpdated` propagates via RabbitMQ before service-bug updates its local read model [source: output/phase-4-architecture/interaction-map.md:E-13]. An admin revoking `editbugs` for a group will not take effect until the projection completes.

- **`GroupControlsMadeMandatory` saga is especially risky** [source: output/phase-4-architecture/interaction-map.md:E-14] — the retroactive addition of a mandatory group to all existing bugs in a product runs asynchronously. Bugs may be briefly accessible without the new mandatory group restriction during the saga's execution.

### Unowned Roles

- **`entry_group_member`** — referenced by `CanEnterProductPolicy` in ADR-006 [source: output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md:94] but the `CreateBug` command's Layer 2 table in SERVICE_SPEC.md does not list `CanEnterProductPolicy` as a check [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:274–279]. The role exists in architecture but may not be enforced in specification.

- **`anonymous` role** — no spec references unauthenticated access. All queries require `(authenticated)` and all commands require a permission string. If any future read-only public access is needed, there is no framework for it.

---

## Effective-Permissions

### Sample role: `reporter` (the user who filed a bug)

**Layer 1 grants**: `bugs:create`, `bugs:update`, `comments:create`, `comments:read`

**Layer 2 narrows**:
- `CanChangeFieldPolicy` — reporters may set severity and priority on their **own** bugs only [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276]
- `CanSeeBugPolicy` — reporter is always visible if `reporterAccessible` flag is true [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:279]
- `CanConfirmBugPolicy` — reporter may confirm own bug if product allows [source: output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md:93]
- `CanCommentOnBugPolicy` — reporter can comment on own bug [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:206]

**Effective set** (operations a reporter can actually perform on their own bugs):
- `bug.create` — file new bugs (if entry group allows)
- `bug.read` — read own bugs (always, subject to reporterAccessible)
- `bug.update.severity` — change severity on own bugs
- `bug.update.priority` — change priority on own bugs
- `bug.confirm` — confirm own bug if product permits
- `bug.add_dependency` — add dependencies (no Layer 2 restricts reporters)
- `bug.remove_dependency` — remove dependencies
- `bug.add_keyword`, `bug.remove_keyword` — no Layer 2 restriction
- `bug.set_alias` — no Layer 2 restriction
- `bug.add_see_also`, `bug.remove_see_also` — no Layer 2 restriction
- `bug.update_custom_field` — subject to MandatoryFieldPolicy
- `bug.transition_status` — subject to ValidStatusTransitionPolicy and NoOpenBlockersPolicy
- `comment.create` — on own bugs (subject to CanCommentOnBugPolicy)
- `comment.read` — non-private comments only

**Operations the reporter CANNOT perform** (despite having `bugs:update`):
- Reassign the bug (assignee-only) [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276]
- Change the QA contact (QA-contact-only)
- Edit time-tracking fields (requires `bugs:edit_timetracking`)
- Modify workflow configuration (requires `bugs:admin_workflow`)
- View private comments (requires insider status)

---

### Sample role: `assignee` (developer assigned to a bug)

**Layer 1 grants**: `bugs:update`, `comments:create`, `comments:read`

**Layer 2 narrows**:
- `CanChangeFieldPolicy` — only the current assignee can reassign (reassign to another user) [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276]
- `CanSeeBugPolicy` — assignee visibility is not gated by reporterAccessible/ccListAccessible flags; implicit
- `CanCommentOnBugPolicy` — assignee can always comment on their assigned bug [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:206]

**Effective set**:
- `bug.read` — always visible for assigned bugs
- `bug.update` — all fields except those restricted to other roles (QA contact change) and time-tracking
- `bug.reassign` — can reassign to another developer
- `bug.transition_status` — full access to valid transitions
- `bug.add_cc`, `bug.remove_cc` — can add CCs; can remove self from CC
- `comment.create`, `comment.read` — full access on assigned bugs

**Operations the assignee CANNOT perform**:
- Change the QA contact (QA-contact-only)
- Edit time-tracking (requires `bugs:edit_timetracking`)
- Suppress comments (requires `comments:suppress`)
- View private comments (requires insider status)
- Change comment privacy (requires insider status)

---

### Sample role: `editbugs_group_member` (user in a group with editbugs=true for the product)

**Layer 1 grants**: `bugs:update`, `comments:create`, `comments:read`

**Layer 2 narrows**:
- `CanChangeFieldPolicy` — users with editbugs for the product can change most fields [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276]
- `CanSeeBugPolicy` — users with `bugs:update` for the product can always see the bug [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:279]
- `CanCommentOnBugPolicy` — editbugs privilege grants comment permission [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:206]

**Effective set**:
- `bug.read` — all bugs in products where editbugs is granted
- `bug.update` — all fields except time-tracking, QA contact (unless also the QA contact), and reassignment (unless also the assignee)
- `bug.transition_status` — all valid transitions
- `bug.add/remove_dependency`, `bug.mark_duplicate` — full access (visibility guaranteed)
- `bug.add/remove_group` — full access (cannot remove mandatory groups)
- `comment.create` — on any bug in the product
- `comment.read` — all non-private comments

**Operations the editbugs_group_member CANNOT perform**:
- Edit time-tracking (requires dedicated `bugs:edit_timetracking`)
- Modify workflow config (requires `bugs:admin_workflow`)
- View private comments (requires insider status)
- Suppress comments (requires `comments:suppress`)

---

### Sample role: `insider` (member of the insider group)

**Layer 1 grants**: inherits `comments:read` (and any other JWT claims)

**Layer 2 narrows**:
- `IsInsiderPolicy` — user must have the `is_insider` flag / `comments:view:private` [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:207]
- `CanSeePrivateCommentsPolicy` — query-side filter that reveals private comments [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:211]

**Effective set**:
- `comment.set_privacy` — toggle private flag on any comment (if user also has `comments:update:privacy`)
- `comment.read` — can see private comments in `GetComment` and `GetBugComments` results
- `comment.create_private` — can create comments with `isPrivate = true`

**Operations the insider CANNOT perform** (unless also holding other roles):
- Edit bugs (requires `bugs:update`)
- Suppress comments (requires `comments:suppress`)
- Add/remove tags (requires `comment_taggers_group` membership)
- Any bug mutation

---

### Sample role: `bugs_admin` (administrator with workflow configuration access)

**Layer 1 grants**: `bugs:admin_workflow`, plus likely `bugs:update`, `bugs:create`, etc.

**Layer 2 checks**:
- `UpdateStatusWorkflowConfig` — **no Layer 2 policy** [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:257]

**Effective set**:
- `workflow-config.update` — add/remove statuses, transitions, resolutions
- All standard bug operations (if also holds `bugs:update`)

**Key concern**: The `bugs:admin_workflow` permission has no Layer 2 guard. Any user with this JWT claim can modify the global workflow without domain-level validation beyond what the aggregate enforces (static status protection). This is a significant gap for a high-privilege operation.

---

### Sample role: `attachment_submitter` (the user who uploaded an attachment)

**Layer 1 grants**: `attachments:create`, `attachments:update`, `attachments:delete`, `flags:request` (depending on JWT)

**Layer 2 narrows**:
- `CanEditAttachmentPolicy` — submitter may update metadata, mark obsolete, or delete the attachment (`editbugs` membership grants the same powers to non-submitters) [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:374–376]
- `CanEditPrivateAttachmentPolicy` — when toggling `isPrivate=true`, submitter must additionally be in the insider group for the bug's product [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:377,378]
- `CanSetFlagPolicy` — submitter has no automatic flag-setting privilege; group membership rules apply normally [source: output/phase-5-specification/specs/service-attachment/SERVICE_SPEC.md:379]

**Effective set**:
- `attachment.create` — upload new attachments to bugs they can see
- `attachment.update_metadata` — edit description, content type, filename on own attachments
- `attachment.mark_obsolete` — flag own attachments as obsolete
- `attachment.delete` — delete own attachments
- `attachment.create_private` — only if also in the insider group for the bug's product

**Operations the attachment_submitter CANNOT perform** (without other roles):
- Edit attachments uploaded by other users (requires `editbugs`)
- Set or grant flags requiring `grantGroupId` membership
- Create or update flag types (requires `flag-types:create` / `flag-types:update`)

---

### Sample role: `editcomponents_group_member` (admin user with editcomponents grant on a product)

**Layer 1 grants**: `components:create`, `components:update`, `components:delete`, `versions:create`, `versions:update`, `versions:delete`, `milestones:create`, `milestones:update`, `milestones:delete`

**Layer 2 narrows**:
- `CanAdminProductPolicy` — for every component/version/milestone command, the user's group must have `editcomponents=true` for the target product, resolved via `UserGroupMembershipReadModel` and `GroupControlMapReadModel` [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:451–458]
- `MinimumComponentPolicy` — cannot delete the last component of a product [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:453]
- `MinimumVersionPolicy` — cannot delete the last version [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:456]
- `NotDefaultMilestonePolicy` — cannot delete the product's current default milestone [source: output/phase-5-specification/specs/service-product/SERVICE_SPEC.md:459]

**Effective set** (only on products where the user has `editcomponents`):
- `component.create`, `component.update`, `component.delete` (subject to minimum-count constraints)
- `version.create`, `version.update`, `version.delete`
- `milestone.create`, `milestone.update`, `milestone.delete`

**Operations the editcomponents_group_member CANNOT perform**:
- Create or update the product itself (requires `products:create` / `products:update`)
- Update group controls for the product (requires `products:manage-groups`)
- Manage classifications (requires `classifications:*`)
- Touch a product where they don't have `editcomponents`

---

### Sample role: `saved_search_owner` (creator of a personal saved search)

**Layer 1 grants**: `search:saved-search:create`, `search:saved-search:read`, `search:saved-search:update`, `search:saved-search:delete`, `search:saved-search:share`, `search:saved-search:footer`, `search:execute`

**Layer 2 narrows**:
- `OwnsSearchPolicy` — only the creator may update, delete, share, or unshare the saved search [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:168,169,171,173]
- `SearchVisibilityPolicy` — read access requires owner or shared-group member [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:174,175]
- `NotUsedByReportPolicy` — cannot delete a saved search referenced by any `ScheduledReportAggregate` [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:170]
- `GroupExistsPolicy` — share target group must exist and be active [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:172]
- Bug security filter — Elasticsearch results are post-filtered by user's group membership and visibility flags regardless of saved-search ownership [source: output/phase-5-specification/specs/service-search/SERVICE_SPEC.md:176,177]

**Effective set**:
- `saved-search.create`, `saved-search.read`, `saved-search.update`, `saved-search.delete` (only when not used by a report), `saved-search.share` (to active groups only), `saved-search.unshare`, `saved-search.toggle-footer`
- `search.execute`, `search.quicksearch` — but results are filtered by bug visibility, not by saved-search ownership
- `chart.read` — when the user has `search:chart:view`
- `scheduled-report.create` — when referencing a visible saved search

**Operations the saved_search_owner CANNOT perform**:
- Update or delete other users' saved searches
- See bugs they aren't otherwise authorized for, even if a saved-search query would surface them
- Create scheduled reports referencing saved searches they cannot see

---

## Audit Trail Recommendations

### Where to Log Permission Checks

1. **Layer 1 (Gateway)** — Log all denied requests at the API gateway. The gateway sees the JWT, the requested command, and the required permission string. Denied Layer 1 checks never reach the service. Log fields:
   - `userId`, `commandName`, `requiredPermission`, `decision` (denied), `reason` (e.g., "missing claim", "expired token"), `correlationId`, `timestamp`, `sourceIp`

2. **Layer 2 (Service Handlers)** — Log both allowed and denied policy evaluations inside each service handler. Every `@RequirePolicy` evaluation should emit a structured log entry:
   - `userId`, `commandName`, `policyName`, `decision` (allowed/denied), `reason` (denial cause, e.g., "not in editbugs group", "reporterAccessible=false"), `correlationId`, `resourceId` (bugId/commentId), `productId`, `timestamp`

3. **Cross-service event subscriptions** — When `GroupMemberAdded` or `GroupControlsUpdated` events are consumed and read models are updated, log the projection with:
   - `eventId`, `eventType`, `readModelName`, `affectedKeys` (userId or productId:groupId), `projectionTimestamp`

### Recommended Log Fields

| Field | Purpose |
|-------|---------|
| `correlationId` | Ties gateway Layer 1 log to service Layer 2 log for the same request |
| `userId` | The authenticated user performing the action |
| `commandName` | e.g., `UpdateBug`, `CreateComment` |
| `policyName` | e.g., `CanChangeFieldPolicy`, `IsInsiderPolicy` |
| `decision` | `allowed` or `denied` |
| `reason` | Human-readable denial cause (e.g., "user not in editbugs group for product X") |
| `resourceId` | bugId, commentId, etc. |
| `productId` | Product context for authorization decisions |
| `timestamp` | ISO 8601 |

### Retention Guidance

- **Hot tier (searchable, 90 days)**: All permission denied logs from both gateway and service handlers. This supports security incident investigation and access review.
- **Cold tier (1 year)**: All permission evaluations (allowed + denied). Full audit trail for compliance.
- **Extended retention (3+ years)**: Group control matrix changes (`product.Events.GroupControlsUpdated`, `product.Events.GroupControlsMadeMandatory`), group membership changes (`user.Events.GroupMemberAdded/Removed`), and admin workflow changes (`StatusWorkflowConfigUpdated`). These are infrequent but carry high compliance weight — they define who could access what.
- **Correlation**: The `correlationId` on every command ties the gateway-side Layer 1 log to the service-side Layer 2 log, enabling end-to-end permission audit traces.

---

## Modernization Recommendations

### 1. Standardize Permission String Namespace

**Problem**: Service-comment uses a three-level namespace (`comments:update:privacy`, `comments:tag:add`, `comments:tag:search`) while service-bug uses two levels (`bugs:update`, `bugs:admin_workflow`, `bugs:edit_timetracking`). The `comments:view:private` permission doesn't follow the `<resource>:<action>` pattern at all — it is a capability check not tied to a command.

**Recommendation**: Enforce `<resource>:<action>` uniformly. Refactor to:
- `bugs:update` → `bugs:update` (OK as-is)
- `bugs:edit_timetracking` → `bugs:update:timetracking` (align with comment's three-level pattern)
- `bugs:admin_workflow` → `bugs:update:workflow_config`
- `comments:view:private` → `comments:read:private` (read action, not a standalone capability)

**Status quo cost**: Low — inconsistent naming makes security review harder but doesn't cause bugs.
**Migration cost**: Low — rename strings in contracts and gateway config; no domain logic changes.

### 2. Introduce a Permission Catalog

**Problem**: Each SERVICE_SPEC.md is the source of truth for its own permissions. There is no single file listing every permission string in the system. The `comments:view:private` permission exists in prose only [source: output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:199], not in a contract decorator.

**Recommendation**: Create a `PERMISSION_CATALOG.md` that enumerates every Layer 1 permission string, the service that owns it, the commands that use it, and the Layer 2 policies that complement it. This becomes the single source of truth for gateway configuration and security review.

**Status quo cost**: Medium — security auditors must read every spec to build a complete picture.
**Migration cost**: Low — extract from existing specs, no code changes.

### 3. Add Layer 2 Default-Deny for High-Privilege Operations

**Problem**: `bugs:admin_workflow` and `bugs:edit_timetracking` have no Layer 2 policy [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:256–257]. Any handler without an explicit `@RequirePolicy` silently passes Layer 2.

**Recommendation**: Introduce a platform-level guard that rejects any command handler at startup if:
- The command's Layer 1 permission is NOT in an allowlist of "Layer 1 only" permissions (e.g., simple reads)
- The handler has no `@RequirePolicy` decorator

This forces developers to make an explicit decision: either add a Layer 2 policy or declare the handler as "Layer 1 only" with a justification comment.

**Status quo cost**: High — `bugs:admin_workflow` with no Layer 2 means a compromised gateway or JWT forgery gives unrestricted workflow modification.
**Migration cost**: Medium — requires platform-level change plus adding explicit policies to existing handlers.

### 4. Consolidate Group Control Checks into a Shared Policy Library

**Problem**: Both `CanChangeFieldPolicy` (service-bug) and `CanCommentOnBugPolicy` (service-comment) check the `editbugs` group control flag for the bug's product [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276, output/phase-5-specification/specs/service-comment/SERVICE_SPEC.md:206]. The same read model query pattern is duplicated.

**Recommendation**: Extract a shared `HasProductPermissionPolicy` (or similar) that encapsulates: "does user's group have permission X for product Y?" Both services import this shared policy. The policy queries `UserGroupMembershipReadModel` and `ProductGroupControlsReadModel`.

**Trade-off note**: ADR-006 argues against a dedicated authorization service [source: output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md:156–161]. This recommendation does NOT propose a separate service — it proposes a shared code library that both services import, keeping the local read model pattern intact.

**Status quo cost**: Low — duplication is manageable with two services but will grow as more services need product-level permission checks.
**Migration cost**: Low — extract common logic into a shared package; no architectural change.

### 5. Split `CanChangeFieldPolicy` into Field-Specific Policies

**Problem**: `CanChangeFieldPolicy` is a monolithic policy that implements at least four distinct authorization rules: reporter → severity/priority, assignee → reassign, QA contact → QA contact, editbugs group → most fields [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:276]. This makes testing and auditing harder.

**Recommendation**: Decompose into:
- `CanChangeSeverityPolicy` (reporter + editbugs)
- `CanReassignPolicy` (current assignee + editbugs)
- `CanChangeQaContactPolicy` (current QA contact + editbugs)
- `CanChangeGeneralFieldsPolicy` (editbugs)

Handlers declare which field-specific policies they require.

**Status quo cost**: Medium — the monolithic policy is harder to test in isolation and harder to reason about in security review.
**Migration cost**: Medium — requires refactoring the policy class and updating all handler decorators, but no architectural change.

### 6. Document and Enforce `CanEnterProductPolicy` and `CanEditProductPolicy`

**Problem**: These policies are declared in ADR-006 [source: output/phase-4-architecture/decisions/ADR-adr-006-group-permission-three-way-split.md:94–95] but absent from the service-bug SERVICE_SPEC Layer 2 table [source: output/phase-5-specification/specs/service-bug/SERVICE_SPEC.md:274–279]. It is unclear whether they are enforced in code or only exist in the architecture document.

**Recommendation**: Either add them to the SERVICE_SPEC Layer 2 table (if implemented) or surface their absence as a specification gap. If they are implemented but not documented, the spec is incomplete. If they are not implemented, the architecture decision is not being enforced.

**Status quo cost**: High — undefined enforcement means product-level entry/canedit controls may not be working as designed.
**Migration cost**: Low — add policy references to spec, then verify implementation matches.

### 7. Extract a Shared `OwnsResourcePolicy` Base

**Problem**: Seven near-identical policies — `OwnsProfilePolicy`, `OwnsKeyPolicy`, `OwnsSchedulePolicy`, `OwnsPreferencesPolicy`, `OwnsDeliveryLogPolicy`, `OwnsSearchPolicy`, `OwnsReportPolicy` — implement the same "caller userId equals owner field" check across four services [sources: service-user 244,250; service-notification 239,242,243; service-search 168,180]. Each is a separate class with its own test coverage and review burden.

**Recommendation**: Extract a generic `OwnsResourcePolicy<T>(ownerField: keyof T)` policy in a shared platform package. Each service keeps a thin alias (e.g., `OwnsSchedulePolicy = OwnsResourcePolicy.bind('ownerUserId')`) for documentation, but the implementation is one tested file. Admin-bypass behavior (e.g., `users:disable` admin can bypass `OwnsProfilePolicy`) is parameterized via an optional bypass-permission argument.

**Status quo cost**: Medium — seven duplicated policies inflate the test surface and create drift risk if one is updated and others aren't.
**Migration cost**: Low — extract a shared base class, swap in each service.

### 8. Disambiguate `ScheduledReport` Across Service-Search and Service-Notification

**Problem**: Both `service-search` and `service-notification` define commands named `CreateScheduledReport`, `UpdateScheduledReport`, `DeleteScheduledReport` with overlapping but distinct semantics [sources: service-search 160–162; service-notification 220–222]. They guard different aggregates (ScheduledReportAggregate in search vs whine reports in notification) but share the prefix `ScheduledReport`. The Layer 1 strings (`search:report:*` vs `notifications:schedule:*`) disambiguate at the gateway, but a code reviewer reading a contract reference cannot tell which service is being invoked without checking the package.

**Recommendation**: Rename one set. Either:
- Rename service-search's commands to `CreateSearchReport`/etc. (since they target saved searches) — preferred because "scheduled report" is the canonical Bugzilla "whine" terminology and service-notification owns the whine pipeline.
- Or merge: have service-search's `CreateScheduledReport` produce an event consumed by service-notification, eliminating the parallel command.

**Status quo cost**: Low — gateway routing is unambiguous; the cost is reviewer confusion.
**Migration cost**: Medium — coordinated rename across contracts, frontend command-builders, and Gherkin scenarios.

### 9. Consolidate `editbugs` Group Lookup Across service-bug, service-comment, and service-attachment

**Problem**: Three services now check `editbugs` group control independently: `CanChangeFieldPolicy` (service-bug) [source: service-bug 276], `CanCommentOnBugPolicy` (service-comment) [source: service-comment 206], and `CanEditAttachmentPolicy` (service-attachment) [source: service-attachment 374]. Each maintains its own projection of `UserGroupMembershipReadModel` and `ProductGroupControlsReadModel` and runs its own group-control evaluation. Three independent eventual-consistency lag windows for the same authorization decision.

**Recommendation**: Promote `HasProductPermissionPolicy(permission, productId, userId)` to a shared package (proposed in Recommendation #4) and require it in service-attachment as well. This does not change the architecture (each service still owns its own read models) but ensures the decision logic — including precedence of `editbugs` vs `canconfirm` vs `canedit` — is implemented exactly once.

**Status quo cost**: Medium — three implementations means three places where a `editbugs` semantics change must be applied.
**Migration cost**: Low — shared library; each service swaps in the import.

### 10. Standardize Three-Level Permission Namespace Across All Services

**Problem**: Permission strings are inconsistent across services [sources: service-bug 269–289; service-comment 199–204; service-user 219–238; service-product 420–445; service-attachment 347–360; service-search 147–162; service-notification 220–229]:
- Two-level (`bugs:update`, `users:update`, `products:update`, `attachments:update`)
- Three-level (`comments:update:privacy`, `search:saved-search:update`, `notifications:schedule:update`, `notifications:preferences:update`)
- Snake-case actions (`bugs:transition_status`, `bugs:edit_timetracking`, `bugs:admin_workflow`)
- Hyphenated resource names (`flag-types:create`, `products:manage-groups`)

**Recommendation**: Adopt a uniform `<service>:<resource>:<action>` three-level lowercase-kebab format. Migration table:
- `bugs:update` → `bugs:bug:update`
- `bugs:transition_status` → `bugs:bug:transition-status`
- `users:update` → `users:user:update`
- `groups:bless` → `users:group:bless`
- `attachments:create` → `attachments:attachment:create`
- `flag-types:create` → `attachments:flag-type:create`
- `products:manage-groups` → `products:group-control:update`

This makes role-to-permission seed data and gateway config self-describing.

**Status quo cost**: Medium — inconsistent namespace makes role definitions hard to write and audit.
**Migration cost**: Medium — touches every contract decorator and every gateway routing rule, but no domain logic.
