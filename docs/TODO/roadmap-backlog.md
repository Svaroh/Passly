# Passly: Roadmap Backlog & Pending Implementation Tasks

This document tracks unimplemented features, architectural milestones, and pending endpoints extracted from the system roadmap and implementation audit.

---

## 1. Backend API (`passly_api`): Protcol `sync/1` Completion

While the core delta synchronization engine (`/sync/changes`, `/sync/meta`, and `sync_log`) is fully landed, the following endpoints and features remain to be implemented:

### 1.1. Paginated Initial Sync (`GET /sync/bootstrap.json`)
- **Objective:** Provide a fast, chunked snapshot of all current resources and permissions for fresh client installations or clients recovering from expired cursors (`410 Gone`).
- **Target Component:** `plugins/PassboltCe/Sync/src/Controller/SyncBootstrapController.php`, `SyncBootstrapService.php`.
- **Requirements:**
  - Support cursor pagination (`limit`, `page`/`cursor`).
  - Stream current active resources, secrets, and permissions without querying history tables.
  - Return the latest monotonic `seq` watermark alongside data items so the client can immediately switch to delta synchronization via `/sync/changes`.

### 1.2. Lightweight State Reconciliation (`GET /sync/manifest.json`)
- **Objective:** Enable clients to verify the integrity of their local replicas without performing full data transfers.
- **Target Component:** `plugins/PassboltCe/Sync/src/Controller/SyncManifestController.php`, `SyncManifestService.php`.
- **Requirements:**
  - Emit lightweight cryptographic hashes (e.g. SHA-256) or count/version summaries grouped by resource types and folders.
  - Allow clients to detect local store corruption or missed delta windows.

### 1.3. Sync Audit Log Classification
- **Objective:** Optimize access log retention and avoid audit log bloating during bulk synchronization.
- **Target Component:** `src/Model/Table/AuditLogsTable.php` / `PassboltCe/Sync`.
- **Requirements:**
  - Classify high-frequency secret reads originating from delta queries as `sync` actions rather than ad-hoc individual user reads.

---

## 2. Android Client (`passly_android`): Incremental Delta Sync (Milestone M3, Requirement R3)

The Android client currently relies on `core/fulldatarefresh` for background data fetching. The existing `core/sync` primitives must be wired into an active delta engine.

### 2.1. Implement Delta Strategies
- **Target Module:** `passly_android/core/sync`
- **Key Deliverables:**
  - `DeltaSyncStrategy`: Query `/sync/changes.json?since={localSeq}` and parse additions, modifications, and deletions.
  - `ChangesApplier`: Apply delta batches directly to Room DB (`secrets`, `resources`, `permissions`) within a single database transaction.
  - `BootstrapStrategy`: Fallback handler when server returns `410 Gone` (cursor older than retention threshold). Trigger `/sync/bootstrap.json` and reset `localSeq`.
  - `ManifestReconciler`: Periodic sanity check comparing local record counts against `/sync/manifest.json`.

### 2.2. Background WorkManager Scheduler
- **Target Module:** `passly_android/services/sync` (or `core/sync/work`)
- **Key Deliverables:**
  - Periodic `CoroutineWorker` integrated with `NetworkStateProvider` and `SyncBackoff`.
  - Trigger delta sync on app foregrounding, network reconnection, or configurable time intervals (e.g. 15 min, 1 hr).

---

## 3. Client UI: Sync Management (Milestone M4, Requirement R4)

Both Android and Web Extension require user-facing controls to monitor and trigger synchronization manually.

### 3.1. Manual Sync Action & Status Indicators
- **Targets:** `passly_android` (Settings & HomeScreen), `passly_browser_extension` (QuickAccess & Workspace).
- **Deliverables:**
  - "Sync Now" button providing instant visual feedback (idle, in-progress, failed, up-to-date).
  - Human-readable timestamp of last successful sync (e.g. "Synced 5 minutes ago").
  - Indicator for pending offline operations count.

### 3.2. Sync Preferences Screen
- Configurable auto-sync frequency (periodic interval, on connect only, or manual-only).
- Diagnostic logs and manual database reconciliation trigger.

---

## 4. Offline Writes & Conflict Handling (Milestone M5 & Step E-E, Requirement R7)

Currently, mutations made while offline are blocked until connectivity is re-established. True offline autonomy requires queued mutations and optimistic local writes.

### 4.1. Local Operation Queue (`pending_ops`)
- **Targets:** `passly_android` (Room table `pending_ops`), `passly_browser_extension` (IndexedDB / storage).
- **Deliverables:**
  - Store pending operations (`CREATE`, `UPDATE`, `DELETE`) with local timestamps and client UUIDs.
  - Apply mutations immediately to local store with an optimistic flag (`is_pending = true`).

### 4.2. Conflict Resolution (`sync_conflicts`)
- Detect version divergence when replaying `pending_ops` against the server (`409 Conflict`).
- Preserve conflicting versions in a dedicated `sync_conflicts` store rather than discarding user data.
- UI flow for users to review and resolve conflicting secret changes.

---

## 5. Emergency Autonomous Data Export (Milestone M6, Requirement R8)

Guarantee that users can export their secrets even if the server is permanently offline.

### 5.1. Standalone Vault Decryption & Export
- **Targets:** `passly_android`, `passly_browser_extension`.
- **Deliverables:**
  - Export local SQLCipher/IndexedDB replica into an encrypted KeePass 4 (`.kdbx`) container.
  - Optional unencrypted / OpenPGP-encrypted standard JSON backup file.
  - Zero network round-trips: export operation decrypts secrets locally using the user's master passphrase and private PGP key.
