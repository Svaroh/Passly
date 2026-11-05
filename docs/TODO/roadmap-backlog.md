# Passly: Roadmap Backlog & Pending Implementation Tasks

This document tracks unimplemented features, architectural milestones, and pending client components extracted from the system roadmap and implementation audit.

> [!NOTE]
> Completed milestones have been migrated to **[docs/DONE/](../DONE/)**:
> - **Milestone M2 (Protocol `sync/1` API)** is documented in **[docs/DONE/implementation-status.md](../DONE/implementation-status.md)**.
> - **Milestone M3 (Android Incremental Delta Sync v6.0.12)** is documented in **[docs/DONE/android-delta-sync-m3.md](../DONE/android-delta-sync-m3.md)** and **[docs/DONE/implementation-status.md](../DONE/implementation-status.md)**.
> - **Milestone M3 (Browser Extension Incremental Delta Sync v6.1.16)** is documented in **[docs/DONE/browser-delta-sync-m3.md](../DONE/browser-delta-sync-m3.md)** and **[docs/DONE/implementation-status.md](../DONE/implementation-status.md)**.

---

## 1. Client UI: Sync Management (Milestone M4, Requirement R4)

Both Android and Web Extension require user-facing controls to monitor and trigger synchronization manually.

### 1.1. Manual Sync Action & Status Indicators
- **Targets:** `passly_android` (Settings & HomeScreen), `passly_browser_extension` (QuickAccess & Workspace).
- **Deliverables:**
  - "Sync Now" button providing instant visual feedback (idle, in-progress, failed, up-to-date).
  - Human-readable timestamp of last successful sync (e.g. "Synced 5 minutes ago").
  - Indicator for pending offline operations count.

### 1.2. Sync Preferences Screen
- Configurable auto-sync frequency (periodic interval, on connect only, or manual-only).
- Diagnostic logs and manual database reconciliation trigger.

---

## 2. Offline Writes & Conflict Handling (Milestone M5 & Step E-E, Requirement R7)

Currently, mutations made while offline are blocked until connectivity is re-established. True offline autonomy requires queued mutations and optimistic local writes.

### 2.1. Local Operation Queue (`pending_ops`)
- **Targets:** `passly_android` (Room table `pending_ops`), `passly_browser_extension` (IndexedDB / storage).
- **Deliverables:**
  - Store pending operations (`CREATE`, `UPDATE`, `DELETE`) with local timestamps and client UUIDs.
  - Apply mutations immediately to local store with an optimistic flag (`is_pending = true`).

### 2.2. Conflict Resolution (`sync_conflicts`)
- Detect version divergence when replaying `pending_ops` against the server (`409 Conflict`).
- Preserve conflicting versions in a dedicated `sync_conflicts` store rather than discarding user data.
- UI flow for users to review and resolve conflicting secret changes.

---

## 3. Emergency Autonomous Data Export (Milestone M6, Requirement R8)

Guarantee that users can export their secrets even if the server is permanently offline.

### 3.1. Standalone Vault Decryption & Export
- **Targets:** `passly_android`, `passly_browser_extension`.
- **Deliverables:**
  - Export local SQLCipher/IndexedDB replica into an encrypted KeePass 4 (`.kdbx`) container.
  - Optional unencrypted / OpenPGP-encrypted standard JSON backup file.
  - Zero network round-trips: export operation decrypts secrets locally using the user's master passphrase and private PGP key.
