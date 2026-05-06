# C4 Stability Ratings

> Source of truth: `audit-output/cluster-inventory.md` (W1). All source files report `(unversioned)` with zero churn and no sibling test coverage — every element defaults to `unknown`.

## Ratings Table

| Level | Element | Service | Stability | Rationale | Source |
|---|---|---|---|---|---|
| L2 | API Gateway | — | unknown | Infrastructure; no W1 data | [source: output/phase-4-architecture/interaction-map.md:24] |
| L2 | service-bug | bug | unknown | Bugzilla::Bug 5 123 LOC god-object; 0 churn, no nearby tests | [source: audit-output/cluster-inventory.md:31] |
| L2 | service-user | user | unknown | Bugzilla::User 3 407 LOC; 0 churn, no nearby tests | [source: audit-output/cluster-inventory.md:55] |
| L2 | service-product | product | unknown | Bugzilla::Product 1 185 LOC; 0 churn, no nearby tests | [source: audit-output/cluster-inventory.md:70] |
| L2 | service-comment | comment | unknown | Smallest domain at 724 total LOC; 0 churn | [source: audit-output/cluster-inventory.md:67] |
| L2 | service-attachment | attachment | unknown | Flag.pm 1 309 LOC + Attachment.pm 1 055 LOC; 0 churn | [source: audit-output/cluster-inventory.md:26] |
| L2 | service-search | search | unknown | Bugzilla::Search 3 561 LOC; 0 churn | [source: audit-output/cluster-inventory.md:63] |
| L2 | service-notification | notification | unknown | Terminal consumer; 0 churn, no W1 data | [source: audit-output/cluster-inventory.md:1] |
| L2 | PostgreSQL Event Store | — | unknown | Infrastructure | [source: output/phase-4-architecture/context-map.md:1] |
| L2 | PostgreSQL Read Models | — | unknown | Infrastructure | [source: output/phase-4-architecture/context-map.md:1] |
| L2 | Redis | — | unknown | Infrastructure | [source: output/phase-4-architecture/context-map.md:1] |
| L2 | RabbitMQ | — | unknown | Infrastructure | [source: output/phase-4-architecture/context-map.md:89] |
| L2 | Elasticsearch | — | unknown | Infrastructure | [source: audit-output/c4/component-search.md:16] |
| L2 | S3/MinIO | — | unknown | Infrastructure | [source: output/phase-4-architecture/services/service-attachment.md:440] |
| L2 | SMTP | — | unknown | External system | [source: output/phase-4-architecture/interaction-map.md:345] |
| L2 | Jaeger | — | unknown | Infrastructure | [source: output/phase-4-architecture/context-map.md:1] |
| L3 | *(component-bug.md missing)* | bug | unknown | Slicer did not produce component diagram | — |
| L3 | CreateUserHandler | user | unknown | Command handler; creates UserAggregate | [source: audit-output/c4/component-user.md:28] |
| L3 | UpdateUserProfileHandler | user | unknown | Command handler; updates realName | [source: audit-output/c4/component-user.md:29] |
| L3 | ChangeEmailHandler | user | unknown | Command handler; two-step email change | [source: audit-output/c4/component-user.md:30] |
| L3 | DisableUserHandler | user | unknown | Command handler; sets disabledText | [source: audit-output/c4/component-user.md:31] |
| L3 | EnableUserHandler | user | unknown | Command handler; clears disabledText | [source: audit-output/c4/component-user.md:32] |
| L3 | ChangePasswordHandler | user | unknown | Command handler; validates + rehashes | [source: audit-output/c4/component-user.md:33] |
| L3 | AuthenticateUserHandler | user | unknown | Command handler; public, no events | [source: audit-output/c4/component-user.md:34] |
| L3 | CreateGroupHandler | user | unknown | Command handler; creates GroupAggregate | [source: audit-output/c4/component-user.md:35] |
| L3 | UpdateGroupHandler | user | unknown | Command handler; re-derives regex | [source: audit-output/c4/component-user.md:36] |
| L3 | AddGroupMemberHandler | user | unknown | Command handler; emits GroupMemberAdded | [source: audit-output/c4/component-user.md:37] |
| L3 | RemoveGroupMemberHandler | user | unknown | Command handler; emits GroupMemberRemoved | [source: audit-output/c4/component-user.md:38] |
| L3 | AddGroupInheritanceHandler | user | unknown | Command handler; validates acyclicity | [source: audit-output/c4/component-user.md:39] |
| L3 | RemoveGroupInheritanceHandler | user | unknown | Command handler; removes DAG edge | [source: audit-output/c4/component-user.md:40] |
| L3 | CreateAPIKeyHandler | user | unknown | Command handler; generates 40-char key | [source: audit-output/c4/component-user.md:41] |
| L3 | RevokeAPIKeyHandler | user | unknown | Command handler; marks key revoked | [source: audit-output/c4/component-user.md:42] |
| L3 | UpdateUserSettingHandler | user | unknown | Command handler; updates preference | [source: audit-output/c4/component-user.md:43] |
| L3 | UpdateEmailPreferencesHandler | user | unknown | Command handler; updates email prefs | [source: audit-output/c4/component-user.md:44] |
| L3 | GetUserHandler | user | unknown | Query handler; reads UserProfileReadModel | [source: audit-output/c4/component-user.md:45] |
| L3 | GetUserByEmailHandler | user | unknown | Query handler; reads by loginName | [source: audit-output/c4/component-user.md:46] |
| L3 | ListUsersHandler | user | unknown | Query handler; paginated list | [source: audit-output/c4/component-user.md:47] |
| L3 | GetGroupHandler | user | unknown | Query handler; reads GroupListReadModel | [source: audit-output/c4/component-user.md:48] |
| L3 | GetGroupByNameHandler | user | unknown | Query handler; reads by name | [source: audit-output/c4/component-user.md:49] |
| L3 | ListGroupsHandler | user | unknown | Query handler; paginated group listing | [source: audit-output/c4/component-user.md:50] |
| L3 | ListGroupMembersHandler | user | unknown | Query handler; reads UserGroupMembershipReadModel | [source: audit-output/c4/component-user.md:51] |
| L3 | ListUserGroupsHandler | user | unknown | Query handler; reads UserGroupMembershipReadModel | [source: audit-output/c4/component-user.md:52] |
| L3 | ListAPIKeysHandler | user | unknown | Query handler; reads APIKeyListReadModel | [source: audit-output/c4/component-user.md:53] |
| L3 | GetUserSettingsHandler | user | unknown | Query handler; reads UserSettingsReadModel | [source: audit-output/c4/component-user.md:54] |
| L3 | GetEmailPreferencesHandler | user | unknown | Query handler; reads UserEmailPreferencesReadModel | [source: audit-output/c4/component-user.md:55] |
| L3 | UserAggregate | user | unknown | Aggregate root; @Aggregate('User'), UUID ID | [source: audit-output/c4/component-user.md:56] |
| L3 | GroupAggregate | user | unknown | Aggregate root; @Aggregate('Group'), UUID ID | [source: audit-output/c4/component-user.md:57] |
| L3 | UserProfileReadModel | user | unknown | Read model; rm_user_profile | [source: audit-output/c4/component-user.md:58] |
| L3 | UserGroupMembershipReadModel | user | unknown | Read model; rm_user_group_membership | [source: audit-output/c4/component-user.md:59] |
| L3 | GroupListReadModel | user | unknown | Read model; rm_group_list | [source: audit-output/c4/component-user.md:60] |
| L3 | APIKeyListReadModel | user | unknown | Read model; rm_api_key_list | [source: audit-output/c4/component-user.md:61] |
| L3 | UserSettingsReadModel | user | unknown | Read model; rm_user_settings | [source: audit-output/c4/component-user.md:62] |
| L3 | UserEmailPreferencesReadModel | user | unknown | Read model; rm_user_email_preferences | [source: audit-output/c4/component-user.md:63] |
| L3 | OwnsProfilePolicy | user | unknown | Layer-2 policy; userId === targetUserId | [source: audit-output/c4/component-user.md:64] |
| L3 | CanBlessGroupPolicy | user | unknown | Layer-2 policy; GROUP_BLESS grant | [source: audit-output/c4/component-user.md:65] |
| L3 | CanAdministerGroupsPolicy | user | unknown | Layer-2 policy; admin or creategroups | [source: audit-output/c4/component-user.md:66] |
| L3 | NotDisabledPolicy | user | unknown | Layer-2 policy; isEnabled check | [source: audit-output/c4/component-user.md:67] |
| L3 | CanDisableUserPolicy | user | unknown | Layer-2 policy; editusers + last-admin guard | [source: audit-output/c4/component-user.md:68] |
| L3 | CanEnableUserPolicy | user | unknown | Layer-2 policy; editusers group | [source: audit-output/c4/component-user.md:69] |
| L3 | CanViewUserPolicy | user | unknown | Layer-2 policy; GROUP_VISIBLE check | [source: audit-output/c4/component-user.md:70] |
| L3 | OwnsKeyPolicy | user | unknown | Layer-2 policy; API key ownership | [source: audit-output/c4/component-user.md:71] |
| L3 | CreateProductCommandHandler | product | unknown | Command handler; saga initiator | [source: audit-output/c4/component-product.md:38] |
| L3 | UpdateProductCommandHandler | product | unknown | Command handler; scalar updates | [source: audit-output/c4/component-product.md:39] |
| L3 | DeactivateProductCommandHandler | product | unknown | Command handler; sets isActive=false | [source: audit-output/c4/component-product.md:40] |
| L3 | SetDefaultMilestoneCommandHandler | product | unknown | Command handler; changes defaultMilestoneId | [source: audit-output/c4/component-product.md:41] |
| L3 | UpdateGroupControlsCommandHandler | product | unknown | Command handler; validates control matrix | [source: audit-output/c4/component-product.md:42] |
| L3 | CreateComponentCommandHandler | product | unknown | Command handler; CanAdminProductPolicy | [source: audit-output/c4/component-product.md:43] |
| L3 | UpdateComponentCommandHandler | product | unknown | Command handler; CanAdminProductPolicy | [source: audit-output/c4/component-product.md:44] |
| L3 | DeleteComponentCommandHandler | product | unknown | Command handler; MinComponent + NoBugs | [source: audit-output/c4/component-product.md:45] |
| L3 | CreateVersionCommandHandler | product | unknown | Command handler; CanAdminProductPolicy | [source: audit-output/c4/component-product.md:46] |
| L3 | UpdateVersionCommandHandler | product | unknown | Command handler; CanAdminProductPolicy | [source: audit-output/c4/component-product.md:47] |
| L3 | DeleteVersionCommandHandler | product | unknown | Command handler; MinVersion + NoBugs | [source: audit-output/c4/component-product.md:48] |
| L3 | CreateMilestoneCommandHandler | product | unknown | Command handler; CanAdminProductPolicy | [source: audit-output/c4/component-product.md:49] |
| L3 | UpdateMilestoneCommandHandler | product | unknown | Command handler; CanAdminProductPolicy | [source: audit-output/c4/component-product.md:50] |
| L3 | DeleteMilestoneCommandHandler | product | unknown | Command handler; NotDefaultMilestonePolicy | [source: audit-output/c4/component-product.md:51] |
| L3 | CreateClassificationCommandHandler | product | unknown | Command handler; direct CRUD | [source: audit-output/c4/component-product.md:52] |
| L3 | UpdateClassificationCommandHandler | product | unknown | Command handler; direct CRUD | [source: audit-output/c4/component-product.md:53] |
| L3 | DeleteClassificationCommandHandler | product | unknown | Command handler; DefaultClassificationProtectionPolicy | [source: audit-output/c4/component-product.md:54] |
| L3 | GetProductQueryHandler | product | unknown | Query handler; reads ProductDetailReadModel | [source: audit-output/c4/component-product.md:55] |
| L3 | ListProductsQueryHandler | product | unknown | Query handler; reads ProductListReadModel | [source: audit-output/c4/component-product.md:56] |
| L3 | GetProductComponentsQueryHandler | product | unknown | Query handler; reads ComponentListReadModel | [source: audit-output/c4/component-product.md:57] |
| L3 | GetProductVersionsQueryHandler | product | unknown | Query handler; reads VersionListReadModel | [source: audit-output/c4/component-product.md:58] |
| L3 | GetProductMilestonesQueryHandler | product | unknown | Query handler; reads MilestoneListReadModel | [source: audit-output/c4/component-product.md:59] |
| L3 | GetClassificationQueryHandler | product | unknown | Query handler; reads ClassificationReadModel | [source: audit-output/c4/component-product.md:60] |
| L3 | ListClassificationsQueryHandler | product | unknown | Query handler; reads ClassificationReadModel | [source: audit-output/c4/component-product.md:61] |
| L3 | GetGroupControlsQueryHandler | product | unknown | Query handler; reads GroupControlMapReadModel | [source: audit-output/c4/component-product.md:62] |
| L3 | CheckProductAccessQueryHandler | product | unknown | Query handler; ProductAccessPolicy | [source: audit-output/c4/component-product.md:63] |
| L3 | ProductAggregate | product | unknown | Aggregate root; @Aggregate('Product') | [source: audit-output/c4/component-product.md:64] |
| L3 | ComponentAggregate | product | unknown | Aggregate root; @Aggregate('Component') | [source: audit-output/c4/component-product.md:65] |
| L3 | VersionAggregate | product | unknown | Aggregate root; @Aggregate('Version') | [source: audit-output/c4/component-product.md:66] |
| L3 | MilestoneAggregate | product | unknown | Aggregate root; @Aggregate('Milestone') | [source: audit-output/c4/component-product.md:67] |
| L3 | ProductListReadModel | product | unknown | Read model; rm_product_list | [source: audit-output/c4/component-product.md:68] |
| L3 | ProductDetailReadModel | product | unknown | Read model; rm_product_detail | [source: audit-output/c4/component-product.md:69] |
| L3 | ComponentListReadModel | product | unknown | Read model; rm_component_list | [source: audit-output/c4/component-product.md:70] |
| L3 | VersionListReadModel | product | unknown | Read model; rm_version_list | [source: audit-output/c4/component-product.md:71] |
| L3 | MilestoneListReadModel | product | unknown | Read model; rm_milestone_list | [source: audit-output/c4/component-product.md:72] |
| L3 | GroupControlMapReadModel | product | unknown | Read model; rm_group_control_map | [source: audit-output/c4/component-product.md:73] |
| L3 | ClassificationReadModel | product | unknown | Read model; simple table (not event-sourced) | [source: audit-output/c4/component-product.md:74] |
| L3 | ProductCreationSaga | product | unknown | Process manager; orchestrates multi-step creation | [source: audit-output/c4/component-product.md:75] |
| L3 | ClearDefaultAssigneeOnUserDisabledHandler | product | unknown | Event subscription; user.Events.UserDisabled | [source: audit-output/c4/component-product.md:76] |
| L3 | CanAdminProductPolicy | product | unknown | Layer-2 policy; editcomponents + group check | [source: audit-output/c4/component-product.md:77] |
| L3 | ProductAccessPolicy | product | unknown | Layer-2 policy; entry group control | [source: audit-output/c4/component-product.md:78] |
| L3 | ProductHasNoBugsPolicy | product | unknown | Layer-2 policy; blocks deletion | [source: audit-output/c4/component-product.md:79] |
| L3 | MinimumComponentPolicy | product | unknown | Layer-2 policy; ≥1 component | [source: audit-output/c4/component-product.md:80] |
| L3 | MinimumVersionPolicy | product | unknown | Layer-2 policy; ≥1 version | [source: audit-output/c4/component-product.md:81] |
| L3 | NotDefaultMilestonePolicy | product | unknown | Layer-2 policy; prevents default deletion | [source: audit-output/c4/component-product.md:82] |
| L3 | DefaultClassificationProtectionPolicy | product | unknown | Layer-2 policy; id=1 protection | [source: audit-output/c4/component-product.md:83] |
| L3 | CreateCommentCommandHandler | comment | unknown | Command handler; CanCommentOnBugPolicy + IsInsiderPolicy | [source: audit-output/c4/component-comment.md:17] |
| L3 | SetCommentPrivacyCommandHandler | comment | unknown | Command handler; IsInsiderPolicy | [source: audit-output/c4/component-comment.md:18] |
| L3 | AddCommentTagCommandHandler | comment | unknown | Command handler; IsCommentTaggerPolicy | [source: audit-output/c4/component-comment.md:19] |
| L3 | RemoveCommentTagCommandHandler | comment | unknown | Command handler; IsCommentTaggerPolicy | [source: audit-output/c4/component-comment.md:20] |
| L3 | SuppressCommentCommandHandler | comment | unknown | Command handler; IsAdminPolicy | [source: audit-output/c4/component-comment.md:21] |
| L3 | GetCommentQueryHandler | comment | unknown | Query handler; reads CommentDetailReadModel | [source: audit-output/c4/component-comment.md:22] |
| L3 | GetBugCommentsQueryHandler | comment | unknown | Query handler; reads CommentListReadModel | [source: audit-output/c4/component-comment.md:23] |
| L3 | SearchCommentTagsQueryHandler | comment | unknown | Query handler; reads CommentTagWeightReadModel | [source: audit-output/c4/component-comment.md:24] |
| L3 | CommentAggregate | comment | unknown | Aggregate root; @Aggregate('CommentAggregate'), append-only | [source: audit-output/c4/component-comment.md:25] |
| L3 | CommentListReadModel | comment | unknown | Read model; rm_comment_list | [source: audit-output/c4/component-comment.md:26] |
| L3 | CommentDetailReadModel | comment | unknown | Read model; rm_comment_detail | [source: audit-output/c4/component-comment.md:27] |
| L3 | CommentTagWeightReadModel | comment | unknown | Read model; rm_comment_tag_weight | [source: audit-output/c4/component-comment.md:28] |
| L3 | CommentTagActivityReadModel | comment | unknown | Read model; rm_comment_tag_activity | [source: audit-output/c4/component-comment.md:29] |
| L3 | BugDuplicateSubscriptionHandler | comment | unknown | Event subscription; bug.Events.BugMarkedDuplicate | [source: audit-output/c4/component-comment.md:30] |
| L3 | AttachmentCreatedSubscriptionHandler | comment | unknown | Event subscription; attachment.Events.AttachmentCreated | [source: audit-output/c4/component-comment.md:31] |
| L3 | AttachmentUpdatedSubscriptionHandler | comment | unknown | Event subscription; attachment.Events.AttachmentUpdated | [source: audit-output/c4/component-comment.md:32] |
| L3 | CanCommentOnBugPolicy | comment | unknown | Layer-2 policy; product-edit permission | [source: audit-output/c4/component-comment.md:33] |
| L3 | IsInsiderPolicy | comment | unknown | Layer-2 policy; insider group membership | [source: audit-output/c4/component-comment.md:34] |
| L3 | IsCommentTaggerPolicy | comment | unknown | Layer-2 policy; comment_taggers_group | [source: audit-output/c4/component-comment.md:35] |
| L3 | CanSeePrivateCommentsPolicy | comment | unknown | Layer-2 policy; query-side filter | [source: audit-output/c4/component-comment.md:36] |
| L3 | IsAdminPolicy | comment | unknown | Layer-2 policy; admin privileges | [source: audit-output/c4/component-comment.md:37] |
| L3 | CreateAttachmentHandler | attachment | unknown | Command handler; creates attachment + writes to S3 | [source: audit-output/c4/component-attachment.md:29] |
| L3 | UpdateAttachmentMetadataHandler | attachment | unknown | Command handler; batch update metadata | [source: audit-output/c4/component-attachment.md:30] |
| L3 | MarkAttachmentObsoleteHandler | attachment | unknown | Command handler; cascades pending flag cancellation | [source: audit-output/c4/component-attachment.md:31] |
| L3 | DeleteAttachmentHandler | attachment | unknown | Command handler; soft-delete + S3 cleanup | [source: audit-output/c4/component-attachment.md:32] |
| L3 | SetAttachmentFlagHandler | attachment | unknown | Command handler; attachment-level flags | [source: audit-output/c4/component-attachment.md:33] |
| L3 | SetBugFlagHandler | attachment | unknown | Command handler; bug-level flags (ADR-005) | [source: audit-output/c4/component-attachment.md:34] |
| L3 | ClearBugFlagHandler | attachment | unknown | Command handler; retargeting cleanup | [source: audit-output/c4/component-attachment.md:35] |
| L3 | CreateFlagTypeHandler | attachment | unknown | Command handler; admin flag type creation | [source: audit-output/c4/component-attachment.md:36] |
| L3 | UpdateFlagTypeHandler | attachment | unknown | Command handler; admin flag type update | [source: audit-output/c4/component-attachment.md:37] |
| L3 | UpdateFlagTypeInclusionsHandler | attachment | unknown | Command handler; triggers retargeting | [source: audit-output/c4/component-attachment.md:38] |
| L3 | UpdateFlagTypeExclusionsHandler | attachment | unknown | Command handler; triggers retargeting | [source: audit-output/c4/component-attachment.md:39] |
| L3 | DeactivateFlagTypeHandler | attachment | unknown | Command handler; sets isActive=false | [source: audit-output/c4/component-attachment.md:40] |
| L3 | GetAttachmentDataHandler | attachment | unknown | Query handler; pre-signed URL from S3 | [source: audit-output/c4/component-attachment.md:41] |
| L3 | GetAttachmentListHandler | attachment | unknown | Query handler; reads AttachmentListReadModel | [source: audit-output/c4/component-attachment.md:42] |
| L3 | GetAttachmentFlagsHandler | attachment | unknown | Query handler; reads AttachmentFlagReadModel | [source: audit-output/c4/component-attachment.md:43] |
| L3 | GetBugFlagsHandler | attachment | unknown | Query handler; reads BugFlagListReadModel | [source: audit-output/c4/component-attachment.md:44] |
| L3 | GetAvailableFlagTypesHandler | attachment | unknown | Query handler; reads FlagTypeReadModel + ScopeReadModel | [source: audit-output/c4/component-attachment.md:45] |
| L3 | GetFlagTypeHandler | attachment | unknown | Query handler; reads FlagTypeReadModel | [source: audit-output/c4/component-attachment.md:46] |
| L3 | AttachmentAggregate | attachment | unknown | Aggregate root; @Aggregate('Attachment'), owns FlagInstance children | [source: audit-output/c4/component-attachment.md:47] |
| L3 | FlagTypeAggregate | attachment | unknown | Aggregate root; @Aggregate('FlagType'), inclusion/exclusion rules | [source: audit-output/c4/component-attachment.md:48] |
| L3 | AttachmentListReadModel | attachment | unknown | Read model; rm_attachment_list | [source: audit-output/c4/component-attachment.md:49] |
| L3 | AttachmentFlagReadModel | attachment | unknown | Read model; rm_attachment_flag | [source: audit-output/c4/component-attachment.md:50] |
| L3 | BugFlagListReadModel | attachment | unknown | Read model; rm_bug_flag_list | [source: audit-output/c4/component-attachment.md:51] |
| L3 | FlagTypeReadModel | attachment | unknown | Read model; rm_flag_type | [source: audit-output/c4/component-attachment.md:52] |
| L3 | FlagTypeScopeReadModel | attachment | unknown | Read model; rm_flag_type_scope | [source: audit-output/c4/component-attachment.md:53] |
| L3 | UserGroupMembershipReadModel (projected) | attachment | unknown | Projected read model from user events | [source: audit-output/c4/component-attachment.md:54] |
| L3 | ProductReadModel (projected) | attachment | unknown | Projected read model from product events | [source: audit-output/c4/component-attachment.md:55] |
| L3 | ComponentReadModel (projected) | attachment | unknown | Projected read model from product events | [source: audit-output/c4/component-attachment.md:56] |
| L3 | BugMovedFlagRetargetHandler | attachment | unknown | Event subscription; bug.Events.BugProductChanged — HIGH risk | [source: audit-output/c4/component-attachment.md:57] |
| L3 | BugDeletedFlagCleanupHandler | attachment | unknown | Event subscription; bug.Events.BugDeleted | [source: audit-output/c4/component-attachment.md:58] |
| L3 | FlagTypeScopeChangeRetargetHandler | attachment | unknown | Event subscription; FlagTypeInclusionsChanged | [source: audit-output/c4/component-attachment.md:59] |
| L3 | GroupMembershipProjectionHandler | attachment | unknown | Event subscription; user.Events.GroupMemberAdded | [source: audit-output/c4/component-attachment.md:60] |
| L3 | ProductProjectionHandler | attachment | unknown | Event subscription; product.Events.* | [source: audit-output/c4/component-attachment.md:61] |
| L3 | CanEditAttachmentPolicy | attachment | unknown | Layer-2 policy; submitter OR editbugs | [source: audit-output/c4/component-attachment.md:62] |
| L3 | CanEditPrivateAttachmentPolicy | attachment | unknown | Layer-2 policy; insider group | [source: audit-output/c4/component-attachment.md:63] |
| L3 | CanSetFlagPolicy | attachment | unknown | Layer-2 policy; grant/request group check | [source: audit-output/c4/component-attachment.md:64] |
| L3 | FlagTypeApplicabilityPolicy | attachment | unknown | Layer-2 policy; inclusion/exclusion + isActive | [source: audit-output/c4/component-attachment.md:65] |
| L3 | RequesteeVisibilityPolicy | attachment | unknown | Layer-2 policy; requestee validation | [source: audit-output/c4/component-attachment.md:66] |
| L3 | MultiplicableFlagPolicy | attachment | unknown | Layer-2 policy; single-instance constraint | [source: audit-output/c4/component-attachment.md:67] |
| L3 | CreateSavedSearchCommandHandler | search | unknown | Command handler; CRUD SavedSearchAggregate | [source: audit-output/c4/component-search.md:22] |
| L3 | UpdateSavedSearchCommandHandler | search | unknown | Command handler; OwnsSearchPolicy | [source: audit-output/c4/component-search.md:23] |
| L3 | DeleteSavedSearchCommandHandler | search | unknown | Command handler; OwnsSearchPolicy + NotUsedByReportPolicy | [source: audit-output/c4/component-search.md:24] |
| L3 | ShareSavedSearchCommandHandler | search | unknown | Command handler; OwnsSearchPolicy + GroupExistsPolicy | [source: audit-output/c4/component-search.md:25] |
| L3 | UnshareSavedSearchCommandHandler | search | unknown | Command handler; OwnsSearchPolicy | [source: audit-output/c4/component-search.md:26] |
| L3 | ToggleFooterLinkCommandHandler | search | unknown | Command handler; SearchVisibilityPolicy | [source: audit-output/c4/component-search.md:27] |
| L3 | CreateScheduledReportCommandHandler | search | unknown | Command handler; ScheduledReportAggregate | [source: audit-output/c4/component-search.md:28] |
| L3 | UpdateScheduledReportCommandHandler | search | unknown | Command handler; OwnsReportPolicy | [source: audit-output/c4/component-search.md:29] |
| L3 | DeleteScheduledReportCommandHandler | search | unknown | Command handler; OwnsReportPolicy | [source: audit-output/c4/component-search.md:30] |
| L3 | ExecuteSearchQueryHandler | search | unknown | Query handler; boolean chart AST → ES bool query | [source: audit-output/c4/component-search.md:31] |
| L3 | QuicksearchQueryHandler | search | unknown | Query handler; quicksearch → ES multi_match | [source: audit-output/c4/component-search.md:32] |
| L3 | GetSavedSearchQueryHandler | search | unknown | Query handler; reads SavedSearchListReadModel | [source: audit-output/c4/component-search.md:33] |
| L3 | ListSavedSearchesQueryHandler | search | unknown | Query handler; reads SavedSearchListReadModel | [source: audit-output/c4/component-search.md:34] |
| L3 | GetRecentSearchesQueryHandler | search | unknown | Query handler; reads RecentSearchReadModel | [source: audit-output/c4/component-search.md:35] |
| L3 | GetChartDataQueryHandler | search | unknown | Query handler; ES aggregations | [source: audit-output/c4/component-search.md:36] |
| L3 | GetVisibleSeriesQueryHandler | search | unknown | Query handler; reads SeriesCatalogReadModel | [source: audit-output/c4/component-search.md:37] |
| L3 | SavedSearchAggregate | search | unknown | CRUD-mode aggregate (not event-sourced) | [source: audit-output/c4/component-search.md:38] |
| L3 | ScheduledReportAggregate | search | unknown | Event-sourced aggregate for Whine reports | [source: audit-output/c4/component-search.md:39] |
| L3 | SavedSearchListReadModel | search | unknown | Read model; rm_saved_search_list | [source: audit-output/c4/component-search.md:40] |
| L3 | RecentSearchReadModel | search | unknown | Read model; rm_recent_search | [source: audit-output/c4/component-search.md:41] |
| L3 | SeriesCatalogReadModel | search | unknown | Read model; rm_series_catalog | [source: audit-output/c4/component-search.md:42] |
| L3 | Elasticsearch bugs index | search | unknown | External index; 1s refresh, 20+ fields | [source: audit-output/c4/component-search.md:43] |
| L3 | BugCreatedSubscriptionHandler | search | unknown | Event subscription; bug.Events.BugCreated | [source: audit-output/c4/component-search.md:44] |
| L3 | BugUpdatedSubscriptionHandler | search | unknown | Event subscription; bug.Events.BugUpdated | [source: audit-output/c4/component-search.md:45] |
| L3 | BugStatusTransitionedSubscriptionHandler | search | unknown | Event subscription; bug.Events.BugStatusTransitioned | [source: audit-output/c4/component-search.md:46] |
| L3 | BugResolvedSubscriptionHandler | search | unknown | Event subscription; bug.Events.BugResolved | [source: audit-output/c4/component-search.md:47] |
| L3 | CommentCreatedSubscriptionHandler | search | unknown | Event subscription; comment.Events.CommentCreated | [source: audit-output/c4/component-search.md:48] |
| L3 | CommentPrivacyChangedSubscriptionHandler | search | unknown | Event subscription; comment.Events.CommentPrivacyChanged | [source: audit-output/c4/component-search.md:49] |
| L3 | AttachmentCreatedSubscriptionHandler | search | unknown | Event subscription; attachment.Events.AttachmentCreated | [source: audit-output/c4/component-search.md:50] |
| L3 | ProductRenamedSubscriptionHandler | search | unknown | Event subscription; product.Events.ProductRenamed | [source: audit-output/c4/component-search.md:51] |
| L3 | ComponentRenamedSubscriptionHandler | search | unknown | Event subscription; product.Events.ComponentRenamed | [source: audit-output/c4/component-search.md:52] |
| L3 | boolean-chart-ast.ts | search | unknown | Domain module; AST → ES bool query | [source: audit-output/c4/component-search.md:53] |
| L3 | operator-map.ts | search | unknown | Domain module; 25+ operators → ES filter | [source: audit-output/c4/component-search.md:54] |
| L3 | quicksearch-parser.ts | search | unknown | Domain module; mini-language parser | [source: audit-output/c4/component-search.md:55] |
| L3 | security-filter.ts | search | unknown | Domain module; group visibility → ES filter | [source: audit-output/c4/component-search.md:56] |
| L3 | CreateScheduledReportHandler | notification | unknown | Command handler; ScheduledReportAggregate | [source: audit-output/c4/component-notification.md:17] |
| L3 | UpdateScheduledReportHandler | notification | unknown | Command handler; OwnsSchedulePolicy | [source: audit-output/c4/component-notification.md:18] |
| L3 | DeleteScheduledReportHandler | notification | unknown | Command handler; OwnsSchedulePolicy | [source: audit-output/c4/component-notification.md:19] |
| L3 | UpdateNotificationPreferencesHandler | notification | unknown | Command handler; side-effect write | [source: audit-output/c4/component-notification.md:20] |
| L3 | SetUserWatcherHandler | notification | unknown | Command handler; side-effect write | [source: audit-output/c4/component-notification.md:21] |
| L3 | GetNotificationPreferencesHandler | notification | unknown | Query handler; reads NotificationPreferencesReadModel | [source: audit-output/c4/component-notification.md:22] |
| L3 | GetScheduledReportsHandler | notification | unknown | Query handler; reads ScheduledReportReadModel | [source: audit-output/c4/component-notification.md:23] |
| L3 | GetScheduledReportHandler | notification | unknown | Query handler; reads ScheduledReportReadModel | [source: audit-output/c4/component-notification.md:24] |
| L3 | GetWatcherListHandler | notification | unknown | Query handler; reads WatcherMapReadModel | [source: audit-output/c4/component-notification.md:25] |
| L3 | GetEmailDeliveryStatusHandler | notification | unknown | Query handler; reads EmailDeliveryLogReadModel | [source: audit-output/c4/component-notification.md:26] |
| L3 | ScheduledReportAggregate | notification | unknown | Aggregate root; @Aggregate('ScheduledReport') | [source: audit-output/c4/component-notification.md:27] |
| L3 | NotificationPreferencesReadModel | notification | unknown | Read model; rm_notification_preferences | [source: audit-output/c4/component-notification.md:28] |
| L3 | WatcherMapReadModel | notification | unknown | Read model; rm_watcher_map | [source: audit-output/c4/component-notification.md:29] |
| L3 | ScheduledReportReadModel | notification | unknown | Read model; rm_scheduled_report | [source: audit-output/c4/component-notification.md:30] |
| L3 | EmailDeliveryLogReadModel | notification | unknown | Read model; rm_email_delivery_log | [source: audit-output/c4/component-notification.md:31] |
| L3 | GlobalWatcherReadModel | notification | unknown | Read model; rm_global_watchers | [source: audit-output/c4/component-notification.md:32] |
| L3 | BugCreatedNotificationHandler | notification | unknown | Event subscription; bug.Events.BugCreated | [source: audit-output/c4/component-notification.md:33] |
| L3 | BugUpdatedNotificationHandler | notification | unknown | Event subscription; bug.Events.BugUpdated | [source: audit-output/c4/component-notification.md:34] |
| L3 | BugStatusChangedNotificationHandler | notification | unknown | Event subscription; bug.Events.BugStatusTransitioned | [source: audit-output/c4/component-notification.md:35] |
| L3 | BugAssignedNotificationHandler | notification | unknown | Event subscription; bug.Events.BugAssigned | [source: audit-output/c4/component-notification.md:36] |
| L3 | BugResolvedNotificationHandler | notification | unknown | Event subscription; bug.Events.BugResolved | [source: audit-output/c4/component-notification.md:37] |
| L3 | BugMarkedDuplicateNotificationHandler | notification | unknown | Event subscription; bug.Events.BugMarkedDuplicate | [source: audit-output/c4/component-notification.md:38] |
| L3 | CCChangedNotificationHandler | notification | unknown | Event subscription; bug.Events.BugCcChanged | [source: audit-output/c4/component-notification.md:39] |
| L3 | DependencyNotificationHandler | notification | unknown | Event subscription; bug.Events.BugDependencyAdded/Removed | [source: audit-output/c4/component-notification.md:40] |
| L3 | CommentNotificationHandler | notification | unknown | Event subscription; comment.Events.CommentCreated | [source: audit-output/c4/component-notification.md:41] |
| L3 | CommentTagNotificationHandler | notification | unknown | Event subscription; comment.Events.CommentTagged | [source: audit-output/c4/component-notification.md:42] |
| L3 | AttachmentNotificationHandler | notification | unknown | Event subscription; attachment.Events.AttachmentCreated | [source: audit-output/c4/component-notification.md:43] |
| L3 | FlagNotificationHandler | notification | unknown | Event subscription; attachment flag events + bug flag events | [source: audit-output/c4/component-notification.md:44] |
| L3 | UserPreferencesSyncHandler | notification | unknown | Event subscription; user.Events.EmailPreferencesUpdated | [source: audit-output/c4/component-notification.md:45] |
| L3 | UserWatchingSyncHandler | notification | unknown | Event subscription; user.Events.UserWatchingChanged | [source: audit-output/c4/component-notification.md:46] |
| L3 | NotificationRecipientResolver | notification | unknown | Domain service; recipient computation pipeline | [source: audit-output/c4/component-notification.md:47] |
| L3 | EmailRenderer | notification | unknown | Domain service; locale-aware template rendering | [source: audit-output/c4/component-notification.md:48] |
| L3 | EmailDispatcher | notification | unknown | Infrastructure service; SMTP delivery | [source: audit-output/c4/component-notification.md:49] |
| L3 | NotificationPreferencesChecker | notification | unknown | Domain service; per-role wantsBugMail check | [source: audit-output/c4/component-notification.md:50] |

## Mermaid classDef Snippets

Append these class definitions to any rendered C4 diagram to color-code by stability:

```mermaid
classDef stable      fill:#7bc47f,stroke:#4a8c4f,color:#fff;
classDef evolving    fill:#f5a623,stroke:#c47d17,color:#fff;
classDef fragile     fill:#d9534f,stroke:#a02622,color:#fff;
classDef deprecated  fill:#888,stroke:#555,color:#fff,stroke-dasharray:3 3;
classDef unknown     fill:#bbb,stroke:#555,color:#000;
```

## Citations

- [source: audit-output/cluster-inventory.md:31]
- [source: output/phase-4-architecture/context-map.md:20]
- [source: audit-output/c4/component-search.md:5]
- [source: audit-output/c4/component-notification.md:4]
- [source: audit-output/c4/code-attachment-code.md:1]
