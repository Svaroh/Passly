# Milestone M3: Android Client Incremental Delta Sync (`sync/1`)

**Release Version:** `6.0.12` (Build code `69`, initial release `6.0.11`)  
**Target Subproject:** `passly_android`  
**Reference Specification:** [`passly_android/docs/sync-protocol.md`](../../passly_android/docs/sync-protocol.md)  
**Status:** Completed, Verified, and Merged  

---

## 1. Overview

Milestone M3 delivers the client-side delta synchronization engine for the Android application (`passly_android`). Prior to this milestone, the Android client relied on full background data refreshing (`core/fulldatarefresh`). M3 fully operationalizes the client-side implementation of the `sync/1` wire protocol, consuming the server change journal (`sync_log`) and endpoints delivered in Milestone M2 (`/sync/changes.json`, `/sync/bootstrap.json`, `/sync/manifest.json`, and `/sync/meta.json`).

The implementation strictly satisfies all core invariants of the Passly offline architecture:
- **R0 (Autonomous Store):** The local Room database is a self-sufficient replica. Sync is an enhancement for freshness, never availability.
- **R3 (Monotonic Delta & Atomic Application):** Sync cursor is a monotonic 64-bit integer, never a timestamp. Delta pages and cursor updates commit in a single database transaction.
- **R4 (Transparent Recovery):** Server retention purges (`410 Gone`) or epoch changes transparently trigger a bootstrap on top of existing local data without data loss.
- **Non-blocking Execution:** Synchronization operates in the background via WorkManager or as user-initiated actions, never blocking local reads or writes.

---

## 2. Architecture & Delivered Components

```
passly_android/
├── core/database/
│   ├── schemas/net.svaroh.passly.database.ResourceDatabase/31.json
│   └── src/main/java/
│       ├── net/svaroh/passly/database/impl/sync/SyncStateDao.kt
│       └── com/passbolt/mobile/android/database/
│           ├── migrations/Migration30to31.kt
│           └── ResourceDatabase.kt (version 31)
├── model/entity/
│   └── src/main/java/net/svaroh/passly/entity/sync/SyncState.kt
└── core/sync/
    └── src/main/java/net/svaroh/passly/core/sync/
        ├── api/SyncApi.kt
        ├── dto/SyncDtos.kt
        ├── ChangesApplier.kt
        ├── DeltaSyncStrategy.kt
        ├── BootstrapStrategy.kt
        ├── ManifestReconciler.kt
        ├── SyncEngine.kt
        ├── SyncResult.kt
        ├── SyncTrigger.kt
        ├── SyncStatus.kt
        ├── SyncStatusRepository.kt
        ├── SyncBackoff.kt
        ├── SyncModule.kt
        └── work/
            ├── SyncWorker.kt
            └── SyncScheduler.kt
```

### 2.1. Persistent Sync State & Room Database Migration 30 -> 31
- **`SyncState` Entity:** Stores `cursor: Long?`, `epoch: String?`, `lastSyncTimestamp: Long?`, `lastReconcileTimestamp: Long?`, and `failureCount: Int`.
- **`SyncStateDao`:** Provides queries for retrieving sync state, updating the cursor, recording reconciliation timestamps, and managing retry failure counters.
- **`Migration30to31`:** Automates database schema evolution, creating the `SyncState` table with zero downtime or data loss across app updates.
- **Batch Deletions:** Added `deleteAll(ids: List<String>)` in `ResourcesDao`, `FoldersDao`, `UsersDao`, and `GroupsDao` to support atomic batch deletions.

### 2.2. Atomic Batch Application: `ChangesApplier`
- Evaluates additions, modifications (`upserts`), and deletions (`deletions`) inside a single Room transaction (`DatabaseTransactionRunner`).
- Idempotently updates metadata keys, resource types, users, groups, folders, resources, secrets, tags, and cross-references.
- Applies deletions cascaded across resources, folders, users, and groups.
- Persists the new cursor and epoch atomically with the data, preventing the cursor from advancing ahead of uncommitted local state.

### 2.3. Delta Engine: `DeltaSyncStrategy`
- Queries `GET /sync/changes.json?since={cursor}&limit=500`.
- Executes iterative pagination while `has_more == true`, accumulating applied changes.
- Intercepts HTTP `410 Gone` (purged cursor past server 90-day retention or epoch reset) and returns `DeltaSyncResult.Gone` for transparent recovery.

### 2.4. Initial & Recovery Bootstrap: `BootstrapStrategy`
- Queries `GET /sync/bootstrap.json?page=N&limit=500&cursor={pinnedCursor}`.
- Pins the cursor on page 1 and passes it through subsequent pages to guarantee a consistent snapshot even if new mutations occur concurrently on the server.
- Safely overlays data on top of existing local records without purging user data.

### 2.5. Access Revocation & State Reconciliation: `ManifestReconciler`
- Queries the compact manifest `GET /sync/manifest.json?type=resources`.
- Diffing algorithm compares server manifest against local `Resource` and `Secret` entries.
- Purges orphaned local resources whose visibility was revoked on the server.
- Updates `lastReconcileTimestamp` upon completion.

### 2.6. Central Orchestrator: `SyncEngine`
- Single entry point (`sync(trigger: SyncTrigger): SyncResult`) for all sync invocations:
  - `MANUAL`: Pull-to-refresh or force-sync button (forces manifest reconciliation).
  - `UNLOCK`: Post-authentication refresh.
  - `PENDING_WRITE`: Triggered when offline writes are queued.
  - `BACKGROUND`: Scheduled background runs.
- Guarded by coroutine `Mutex` to prevent concurrent sync operations.
- Checks connectivity via `NetworkStateProvider`. Returns `Skipped(NO_NETWORK)` when offline without throwing errors or blocking the UI.
- Schedules retries using `SyncBackoff` (30s -> 1m -> 5m -> 15m -> 1h -> 6h) and records failure telemetry in `SyncStatusRepository`.

### 2.7. Background WorkManager Scheduler: `SyncWorker` & `SyncScheduler`
- **`SyncWorker`:** `CoroutineWorker` integrated with Koin dependency injection, invoking `SyncEngine.sync(SyncTrigger.BACKGROUND)`.
- **`SyncScheduler`:** Enqueues periodic WorkManager jobs with `NetworkType.CONNECTED` constraints and exponential backoff, plus support for immediate one-time triggers.

### 2.8. UI Refresh Integration & Data Absence Policy: `FullDataRefreshExecutor` (v6.0.12)
- Integrates `SyncEngine` directly into `FullDataRefreshExecutor` used by `HomeScreen` and `OtpScreen` pull-to-refresh and initial screen loading.
- **Strict Rule: Full synchronization is executed strictly only when local replica data is absent.**
- When local replica data exists (`hasLocalReplica == true`), the system runs lightweight incremental delta sync (`SyncEngine.sync(SyncTrigger.MANUAL)`), eliminating prolonged UI blocking progress spinners.
- When local replica data is completely absent (first app run on a fresh install), the system executes full initial data refresh / bootstrap.

---

## 3. Verification & Test Suite

The implementation is verified by 31 automated tests across 9 test suites in `core/sync/src/test`:

| Test Suite | Tests | Verification Focus |
|---|:---:|---|
| **`SyncIntegrationE2ETest`** | 3 | End-to-end integration lifecycle: cold-start bootstrap, incremental delta sync, 410 Gone recovery, and access revocation via manifest reconciliation. |
| **`SyncEngineTest`** | 7 | Orchestration, mutex locks, offline skipping, transparent 410 fallback, session failure handling, and retry scheduling. |
| **`DeltaSyncStrategyTest`** | 4 | Multi-page delta pagination (`hasMore`), cursor advancement, 410 Gone interception, and network failure propagation. |
| **`BootstrapStrategyTest`** | 3 | Multi-page bootstrap with pinned cursor consistency and error recovery. |
| **`ManifestReconcilerTest`** | 2 | Detection and deletion of revoked local resources; error handling. |
| **`ChangesApplierTest`** | 1 | Atomic database transactions, batch deletions, and cursor advancement. |
| **`SyncWorkerTest`** | 3 | WorkManager `doWork()` execution, mapping `SyncResult` to `Result.success()` and `Result.retry()`. |
| **`SyncStatusRepositoryTest`** | 5 | In-memory status publisher state transitions (`Idle`, `Running`, `Offline`, `Failed`). |
| **`SyncBackoffTest`** | 3 | Adherence to backoff ladder; immediate zero-delay retry for user-initiated triggers. |

### Build & Migration Verification
- Room migration test compiled: `:database:compileDebugAndroidTestKotlin` — **BUILD SUCCESSFUL**.
- Full test suite passed: `:sync:testDebugUnitTest` — **31 tests passed, 0 failures, 0 errors**.
- Full application package build: `:app:assembleDebug` — **BUILD SUCCESSFUL** (output artifact: `app-debug.apk`, version `6.0.11`, build code `68`).
