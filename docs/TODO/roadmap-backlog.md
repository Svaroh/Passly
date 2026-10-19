# Passly: Roadmap Backlog & Pending Implementation Tasks

This document tracks unimplemented features, architectural milestones, and pending client components extracted from the system roadmap and implementation audit.

> [!NOTE]
> The backend API implementation of the `sync/1` protocol — including `/sync/bootstrap.json`, `/sync/manifest.json`, and secret audit logging — is fully completed and documented in **[docs/DONE/implementation-status.md](../DONE/implementation-status.md)**.

---

## 1. Android Client (`passly_android`): Incremental Delta Sync (Milestone M3, Requirement R3)

The Android client currently relies on `core/fulldatarefresh` for background data fetching. The existing `core/sync` primitives must be wired into an active delta engine to consume the server's `/sync/changes`, `/sync/bootstrap`, and `/sync/manifest` endpoints.

### 1.1. Implement Delta Strategies
- **Target Module:** `passly_android/core/sync`
- **Key Deliverables:**
  - `DeltaSyncStrategy`: Query `/sync/changes.json?since={localSeq}` and parse additions, modifications, and deletions.
  - `ChangesApplier`: Apply delta batches directly to Room DB (`secrets`, `resources`, `permissions`) within a single database transaction.
  - `BootstrapStrategy`: Fallback handler when server returns `410 Gone` (cursor older than retention threshold). Trigger `/sync/bootstrap.json` and reset `localSeq`.
  - `ManifestReconciler`: Periodic sanity check comparing local record counts against `/sync/manifest.json`.

### 1.2. Background WorkManager Scheduler
- **Target Module:** `passly_android/services/sync` (or `core/sync/work`)
- **Key Deliverables:**
  - Periodic `CoroutineWorker` integrated with `NetworkStateProvider` and `SyncBackoff`.
  - Trigger delta sync on app foregrounding, network reconnection, or configurable time intervals (e.g. 15 min, 1 hr).

---

## 2. Client UI: Sync Management (Milestone M4, Requirement R4)

Both Android and Web Extension require user-facing controls to monitor and trigger synchronization manually.

### 2.1. Manual Sync Action & Status Indicators
- **Targets:** `passly_android` (Settings & HomeScreen), `passly_browser_extension` (QuickAccess & Workspace).
- **Deliverables:**
  - "Sync Now" button providing instant visual feedback (idle, in-progress, failed, up-to-date).
  - Human-readable timestamp of last successful sync (e.g. "Synced 5 minutes ago").
  - Indicator for pending offline operations count.

### 2.2. Sync Preferences Screen
- Configurable auto-sync frequency (periodic interval, on connect only, or manual-only).
- Diagnostic logs and manual database reconciliation trigger.

---

## 3. Offline Writes & Conflict Handling (Milestone M5 & Step E-E, Requirement R7)

Currently, mutations made while offline are blocked until connectivity is re-established. True offline autonomy requires queued mutations and optimistic local writes.

### 3.1. Local Operation Queue (`pending_ops`)
- **Targets:** `passly_android` (Room table `pending_ops`), `passly_browser_extension` (IndexedDB / storage).
- **Deliverables:**
  - Store pending operations (`CREATE`, `UPDATE`, `DELETE`) with local timestamps and client UUIDs.
  - Apply mutations immediately to local store with an optimistic flag (`is_pending = true`).

### 3.2. Conflict Resolution (`sync_conflicts`)
- Detect version divergence when replaying `pending_ops` against the server (`409 Conflict`).
- Preserve conflicting versions in a dedicated `sync_conflicts` store rather than discarding user data.
- UI flow for users to review and resolve conflicting secret changes.

---

## 4. Emergency Autonomous Data Export (Milestone M6, Requirement R8)

Guarantee that users can export their secrets even if the server is permanently offline.

### 4.1. Standalone Vault Decryption & Export
- **Targets:** `passly_android`, `passly_browser_extension`.
- **Deliverables:**
  - Export local SQLCipher/IndexedDB replica into an encrypted KeePass 4 (`.kdbx`) container.
  - Optional unencrypted / OpenPGP-encrypted standard JSON backup file.
  - Zero network round-trips: export operation decrypts secrets locally using the user's master passphrase and private PGP key.
