# Requirements Document

## Introduction

This feature adds a "Sync from Main" capability to the workspace isolation module of the novel mini-program management platform. Currently, `SyncService.syncWorkspace()` only syncs Git code (via `git merge main`). Database data in the 9 isolated tables becomes stale when other users merge changes to main. This feature extends the sync operation to atomically synchronize both Git code and database data, using field-level diffing to intelligently merge database changes while preserving the user's local modifications.

## Glossary

- **Sync_Orchestrator**: The top-level service coordinating the "sync from main" operation across both Git and database layers. Extends the existing `SyncService`.
- **Git_Sync_Engine**: The component responsible for merging the main branch into the workspace branch via `git merge`. Currently implemented in `SyncService.doSync()`.
- **Database_Sync_Engine**: The new component responsible for syncing database data from main (workspace_id=0) into the user's workspace (workspace_id=userId) using field-level diffing.
- **Field_Diff_Engine**: The component that computes field-level diffs between workspace data and main data for a given table row, using `update_time` and `dbBaselineVersion` to classify each field into one of four merge categories.
- **Conflict_Reporter**: The component that collects and returns field-level database conflicts to the caller when auto-merge is not possible.
- **Workspace**: An isolated working environment for a user, consisting of a Git worktree (branch off main) and database data copies across 9 tables with `workspace_id` isolation.
- **Main_Data**: The canonical dataset in the 9 isolated tables where `workspace_id = 0`.
- **Workspace_Data**: The user's copy of data in the 9 isolated tables where `workspace_id = userId`.
- **dbBaselineVersion**: A `LocalDateTime` timestamp stored in the `workspace` table record, representing the point in time when the workspace's database data was last known to be in sync with Main_Data.
- **Isolated_Tables**: The 9 database tables subject to workspace isolation: `ad_config`, `app_ad`, `app_common_config`, `app_pay`, `app_preview_page`, `app_ui_config`, `app_weiju_business_type`, `app_weiju_public_switch`, `novel_app`.
- **Field_Merge_Category**: One of four classifications for a field during sync: `unchanged` (neither side changed), `main_changed` (only main changed), `workspace_changed` (only workspace changed), `conflict` (both sides changed to different values).
- **Sync_Result**: The response object returned by the sync operation, containing status (`up_to_date`, `merged`, `conflicts`, `git_conflicts`) and optional conflict details.

## Requirements

### Requirement 1: Atomic Git + Database Sync Orchestration

**User Story:** As a workspace user, I want the sync-from-main operation to update both my Git code and database data atomically, so that my workspace is never left in an inconsistent state where one layer is synced but the other is not.

#### Acceptance Criteria

1. WHEN a sync-from-main operation is triggered, THE Sync_Orchestrator SHALL execute Git sync first, then database sync, in that fixed order.
2. WHEN both Git sync and database sync complete without conflicts, THE Sync_Orchestrator SHALL return a Sync_Result with status `merged`.
3. IF the database sync fails after Git sync has succeeded, THEN THE Sync_Orchestrator SHALL rollback the Git merge by executing `git reset --hard {originalHead}` on the workspace branch, restoring the workspace to its pre-sync Git state.
4. WHEN the Git rollback after a database sync failure completes, THE Sync_Orchestrator SHALL return a Sync_Result with status `db_sync_failed` and include the error details.
5. IF the Git sync detects merge conflicts, THEN THE Sync_Orchestrator SHALL skip the database sync step and return a Sync_Result with status `git_conflicts` and the list of conflicting files.
6. THE Sync_Orchestrator SHALL record the Git HEAD commit hash before starting the sync, to use as the rollback target if database sync fails.
7. WHEN the workspace is already up-to-date with main on both Git and database layers, THE Sync_Orchestrator SHALL return a Sync_Result with status `up_to_date` without modifying any data.

### Requirement 2: Database Data Sync with Field-Level Diffing

**User Story:** As a workspace user, I want the database sync to detect and merge field-level changes from main into my workspace data, so that non-conflicting changes from other users are automatically applied while my own modifications are preserved.

#### Acceptance Criteria

1. WHEN the Database_Sync_Engine processes a table row, THE Field_Diff_Engine SHALL compare each field of the Workspace_Data row against the corresponding Main_Data row.
2. THE Field_Diff_Engine SHALL use the workspace's `dbBaselineVersion` timestamp and each row's `update_time` to determine which side has changed since the workspace was last synced.
3. WHEN a field has the same value in both Workspace_Data and Main_Data, THE Field_Diff_Engine SHALL classify the field as `unchanged` and take no action on that field.
4. WHEN a field differs and only the Main_Data row's `update_time` is after `dbBaselineVersion` (meaning only main changed), THE Field_Diff_Engine SHALL classify the field as `main_changed` and auto-apply the Main_Data value to the Workspace_Data row.
5. WHEN a field differs and only the Workspace_Data row's `update_time` is after `dbBaselineVersion` (meaning only the workspace user changed it), THE Field_Diff_Engine SHALL classify the field as `workspace_changed` and preserve the Workspace_Data value.
6. WHEN a field differs and both the Main_Data and Workspace_Data rows have `update_time` after `dbBaselineVersion`, THE Field_Diff_Engine SHALL classify the field as `conflict`.
7. THE Database_Sync_Engine SHALL process all 9 Isolated_Tables during a single sync operation.
8. WHEN the Database_Sync_Engine completes processing all tables with no `conflict` fields, THE Database_Sync_Engine SHALL apply all `main_changed` field updates to Workspace_Data within a single database transaction.

### Requirement 3: Row-Level Matching Between Workspace and Main Data

**User Story:** As a workspace user, I want the sync engine to correctly match rows between my workspace data and main data, so that field-level diffs are computed on the correct row pairs even when rows have been added or deleted.

#### Acceptance Criteria

1. THE Database_Sync_Engine SHALL match rows between Workspace_Data and Main_Data using a business key that uniquely identifies a logical row across workspace boundaries (excluding `workspace_id` and auto-increment primary keys).
2. WHEN a row exists in Main_Data but not in Workspace_Data, and the row's `update_time` is after `dbBaselineVersion`, THE Database_Sync_Engine SHALL insert the new row into Workspace_Data with the user's `workspace_id`.
3. WHEN a row exists in Workspace_Data but not in Main_Data, THE Database_Sync_Engine SHALL preserve the Workspace_Data row (the user added it locally).
4. WHEN a row exists in Main_Data but not in Workspace_Data, and the row's `update_time` is before `dbBaselineVersion`, THE Database_Sync_Engine SHALL treat the row as deleted by the workspace user and preserve the deletion.

### Requirement 4: Conflict Detection and Reporting for Database Sync

**User Story:** As a workspace user, I want to see exactly which fields conflict when both main and my workspace have changed the same field, so that I can make an informed decision about which value to keep.

#### Acceptance Criteria

1. WHEN the Field_Diff_Engine detects one or more `conflict` fields across any Isolated_Table, THE Conflict_Reporter SHALL collect all conflict details including table name, row business key, field name, main value, and workspace value.
2. WHEN conflicts are detected, THE Database_Sync_Engine SHALL abort the sync transaction without applying any `main_changed` updates, preserving the workspace data in its pre-sync state.
3. WHEN conflicts are detected, THE Sync_Orchestrator SHALL rollback the Git merge (via `git reset --hard {originalHead}`) and return a Sync_Result with status `conflicts` containing the full list of field-level conflict details.
4. THE Conflict_Reporter SHALL include the table name, row identifier, field name, the Main_Data field value, and the Workspace_Data field value for each conflict.

### Requirement 5: Database Baseline Version Update After Successful Sync

**User Story:** As a workspace user, I want my workspace's database baseline to be updated after a successful sync, so that subsequent syncs only detect changes made after this sync point.

#### Acceptance Criteria

1. WHEN the Database_Sync_Engine completes a successful sync (all `main_changed` fields applied, no conflicts), THE Sync_Orchestrator SHALL update the workspace record's `dbBaselineVersion` to the current timestamp.
2. THE Sync_Orchestrator SHALL update `dbBaselineVersion` only after both Git sync and database sync have completed successfully.
3. IF the sync operation fails at any stage, THEN THE Sync_Orchestrator SHALL leave `dbBaselineVersion` unchanged at its previous value.

### Requirement 6: Handling Uncommitted Git Changes During Sync

**User Story:** As a workspace user, I want to be able to sync from main even when I have uncommitted file changes in my workspace, so that I don't lose my work-in-progress edits.

#### Acceptance Criteria

1. WHEN the workspace has uncommitted Git changes and a sync-from-main is triggered, THE Sync_Orchestrator SHALL stash the uncommitted changes before performing the Git merge (via `git stash`).
2. WHEN the Git merge completes successfully and the stash was applied, THE Sync_Orchestrator SHALL restore the stashed changes (via `git stash pop`).
3. IF `git stash pop` produces merge conflicts between the stashed changes and the newly merged code, THEN THE Sync_Orchestrator SHALL return a Sync_Result with status `git_conflicts` indicating stash-pop conflicts, and leave the stash entry intact for manual resolution.
4. WHEN the workspace has no uncommitted Git changes, THE Sync_Orchestrator SHALL skip the stash/unstash steps and proceed directly with the Git merge.

### Requirement 7: Sync-from-Main API Endpoint

**User Story:** As a frontend developer, I want a REST API endpoint to trigger the sync-from-main operation, so that the workspace UI can initiate and display the results of a full sync.

#### Acceptance Criteria

1. THE Sync_Orchestrator SHALL expose a POST endpoint at `/api/workspace/sync-from-main` accepting a `userId` request parameter.
2. WHEN the endpoint is called with a valid `userId`, THE Sync_Orchestrator SHALL execute the full atomic sync (Git + database) and return the Sync_Result as JSON.
3. IF the `userId` does not correspond to an existing user, THEN THE Sync_Orchestrator SHALL return HTTP 404 with an error message.
4. IF the `userId` does not have an active workspace, THEN THE Sync_Orchestrator SHALL return HTTP 400 with an error message.
5. WHEN the sync operation encounters an unexpected error, THE Sync_Orchestrator SHALL return HTTP 500 with the error details.
6. THE Sync_Orchestrator SHALL apply idempotent protection: requests from the same user within 30 seconds SHALL return status `already_in_progress`.

### Requirement 8: Database Conflict Resolution API

**User Story:** As a workspace user, I want to resolve database field-level conflicts through the UI, so that I can choose which value to keep for each conflicting field and complete the sync.

#### Acceptance Criteria

1. THE Sync_Orchestrator SHALL expose a POST endpoint at `/api/workspace/resolve-db-conflicts` accepting a JSON body with `userId` and a list of field-level resolutions.
2. WHEN the resolution endpoint is called, THE Database_Sync_Engine SHALL apply the user's choices: for each conflict field, use the main value if `useMain` is true, or keep the workspace value if `useMain` is false.
3. WHEN all conflict resolutions are applied, THE Database_Sync_Engine SHALL also apply all previously identified `main_changed` fields in the same transaction.
4. WHEN the conflict resolution and auto-merge complete successfully, THE Sync_Orchestrator SHALL re-execute the Git sync and update `dbBaselineVersion`.
5. IF the conflict resolution fails, THEN THE Sync_Orchestrator SHALL rollback the database transaction and return an error.

### Requirement 9: Sync Status Detection for Database Layer

**User Story:** As a workspace user, I want to know whether my workspace database data is behind main, so that the UI can show me when a sync is needed.

#### Acceptance Criteria

1. WHEN the workspace status is queried, THE Sync_Orchestrator SHALL check whether any Main_Data row across the 9 Isolated_Tables has `update_time` after the workspace's `dbBaselineVersion`.
2. WHEN Main_Data has changed since `dbBaselineVersion`, THE Sync_Orchestrator SHALL include `dbBehindMain: true` in the workspace status response.
3. WHEN no Main_Data has changed since `dbBaselineVersion`, THE Sync_Orchestrator SHALL include `dbBehindMain: false` in the workspace status response.

### Requirement 10: Concurrency Safety During Sync

**User Story:** As a platform operator, I want the sync-from-main operation to be safe under concurrent access, so that simultaneous syncs or merges do not corrupt data.

#### Acceptance Criteria

1. THE Database_Sync_Engine SHALL execute all database modifications (field updates, row inserts) within a single database transaction with `@Transactional(rollbackFor = Exception.class)`.
2. THE Sync_Orchestrator SHALL acquire the existing global Git lock (from MergeEngine) before performing Git merge operations, to prevent conflicts with concurrent approve/merge operations.
3. WHEN the global Git lock cannot be acquired within 30 seconds, THE Sync_Orchestrator SHALL return a Sync_Result with status `busy` and a message indicating the system is processing another operation.
