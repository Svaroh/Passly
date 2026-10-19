# Completed Implementation Milestones: Plan (`PLAN.md`) vs. Reality

**Audit Date:** October 16, 2026  
**Reference Document:** `PLAN.md` (Dated 2026-08-07)  
**Status:** Landed & Verified Deliverables

---

## 1. Executive Summary

An audit of the codebase across all four subprojects confirms that the **foundational offline architecture has been implemented in production**, with the browser extension, styleguide, and WebAuthn / Passkeys progressing beyond initial targets to release **v6.1.0**.

| Component | Target in PLAN.md | Actual Status in Codebase | Completed Deliverables |
|-----------|-------------------|---------------------------|------------------------|
| **`passly_android`** | 3.1.x | **6.0.8** (commit `daebc1379`) | **M0** and **M1 (Autonomy)**: Local secret persistence, non-blocking unlock, hardware Keystore decoupled from biometrics, `core/sync` foundation. |
| **`passly_api`** | 5.12.x | **6.1.0** (tag `v6.1.0`) | **M2 (Core Delta)**: `PassboltCe/Sync` plugin (`sync_log` table, `/sync/meta`, `/sync/changes`), adapted `OfflineMode` plugin for perpetual offline defaults. |
| **`passly_browser_extension`** | 6.0.13 | **6.1.0** (tag `v6.1.0`) | **E-A, E-B, E-B1, E-C, E-V, E-D**: Containerized workspace, offline session unlock, local-first reads, Seamless Offline Routing, offline-first passkeys, biometric unlock. |
| **`passly_styleguide`** | v5.12.1 (fork) | **6.1.0** (tag `v6.1.0`) | Fully adapted for autonomous execution, offline states, and biometric auth dialogs. |

> [!NOTE]
> All pending milestones and unfulfilled requirements have been transitioned to the roadmap backlog in **[docs/TODO/roadmap-backlog.md](../TODO/roadmap-backlog.md)**.

---

## 2. Completed Milestones

### Milestone M0: Specification & Sync Foundation
- **Reality:** ✅ **Fully Landed.**
  - Specification codified in `passly_android/docs/sync-protocol.md` and `docs/architecture/sync-protocol.md`.
  - Module `core/sync` established in `passly_android/core/sync`: includes `SyncStatus`, `SyncResult`, `SyncTrigger`, `SyncBackoff` (with unit test suite), `SyncStatusRepository`, and `NetworkStateProvider`.

---

### Milestone M1: Android Autonomy (Requirements R0, R2, R5, R6)
- **Reality:** ✅ **Fully Landed (commit `1e1aea3c4` and subsequent refactors).**
  - **Database:** Created `secrets` table (`SecretEntity`), `SecretsDao`, and Room migration `Migration28to29.kt`.
  - **Local-first Reading:** `SecretInteractor` queries `GetLocalSecretUseCase` first; if an entry is cached locally, no network connection is attempted.
  - **Non-blocking Unlock (R6):** Implemented `BackgroundSignInExecutor` and `HasLocalReplicaUseCase`. Vault opens immediately upon local passphrase verification; server GPGAuth session establishment runs asynchronously.
  - **Network Timeouts:** Connection timeout trimmed to 10s; overall call timeout capped at 60s; server PGP and RSA keys fetched concurrently.
  - **Critical Path Audit:** `AuthenticatedOperationRunner` no longer blocks offline reads when session tokens are absent (`requiresSession = false`); `KeyInspectorViewModel` reads local `users` data; the creation button on `HomeScreen` is preserved during background sync failures.
  - **Keystore Security (R5):** Storage master key preserved without `setUserAuthenticationRequired` or `setInvalidatedByBiometricEnrollment`.

---

### Milestone M2: Backend API — Core Delta Engine (`passly_api`)
- **Reality:** ✅ **Landed & Operational.**
  - Plugin established in `plugins/PassboltCe/Sync`.
  - Database migration `config/Migrations/20260812200000_V600CreateSyncLog.php` (`sync_log` table with `seq`, `entity_type`, `entity_id`, `op`, `created`).
  - Event listener `SyncLogModelListener` and recorder `SyncLogRecorderService` capturing mutations to `resources`, `secrets`, and `permissions`.
  - Endpoint `GET /sync/meta.json` (`SyncMetaController`, `SyncMetaService`).
  - Endpoint `GET /sync/changes.json` (`SyncChangesController`, `SyncChangesService`) supporting `since` cursor, limit, watermark settling (2s lag), 90-day retention checks (`410 Gone`), and dynamic ACL filtering.
  - Maintenance pruning: `SyncLogRetentionService` and CLI command `SyncLogPruneCommand` (`bin/cake sync_log_prune`).
  - Plugin `PassboltCe/OfflineMode` adapted with perpetual offline defaults.
  - *(Remaining endpoints `/sync/bootstrap` and `/sync/manifest` transitioned to [docs/TODO/roadmap-backlog.md](../TODO/roadmap-backlog.md)).*

---

### Milestone M4: Status Messaging & Neutral Phrasing (Requirement R4)
- **Reality:** ✅ **Landed.**
  - Offline status messaging and neutral phrasing have been integrated across Android screens.
  - *(Dedicated sync preferences screen transitioned to [docs/TODO/roadmap-backlog.md](../TODO/roadmap-backlog.md)).*

---

## 3. Browser Extension Autonomy (Phase 2 / Section 11a)

All planned read-only and autonomous workspace objectives are fully implemented in production:

| Step | Planned Objective | Actual Status in v6.1.0 |
|------|-------------------|-------------------------|
| **E-A** | Local `workspace.html` & `signin.html` in `webAccessibleResources` | ✅ **Landed.** UI rendered within extension container without remote web host. |
| **E-B** | `OfflineSessionService`, local passphrase check, offline handling | ✅ **Landed.** Unlocking operates with zero network dependencies. |
| **E-B1** | Halt `LocalStorageService.flush()` on startup | ✅ **Landed.** Local replica persists indefinitely across browser restarts. |
| **E-C** | Local-first read path for resources, profile, groups | ✅ **Landed.** Vault items read from local storage before checking API. |
| **E-V** | Automated Playwright offline test suite | ✅ **Landed.** `test/offline/verifyOfflineWorkspace.js` verified passing. |
| **E-D** | Offline UI states in styleguide | ✅ **Landed.** Filters (including Trash), dialogs, and banners adapted for offline mode. |

### Features Delivered Beyond Initial Plan:
1. **Seamless Offline Routing (PR #45):** If the server host fails to resolve or drops connection, the extension seamlessly switches to the bundled `workspace.html` without presenting browser network failure screens.
2. **Offline-first Passkeys:** Direct WebAuthn credential filtering from local vault storage (`passkeyVaultResourceService`), stripping background API delays from passkey ceremonies.
3. **Biometric Vault Unlock:** WebAuthn User Verification integration (Touch ID / Windows Hello) for unlocking the extension workspace without entering a passphrase.

---

## 4. Pending Initiatives

The following unfulfilled roadmap items have been transferred to **[docs/TODO/roadmap-backlog.md](../TODO/roadmap-backlog.md)**:
- **Milestone M3**: Incremental Delta Sync client in `passly_android`.
- **Milestone M4**: User-configurable sync preferences screen.
- **Milestone M5 & Step E-E**: Offline write queue (`pending_ops`) and conflict resolution (`sync_conflicts`).
- **Milestone M6**: Standalone emergency vault export (KDBX 4 / PGP-JSON).
- **Backend Sync API**: `/sync/bootstrap.json` and `/sync/manifest.json`.
