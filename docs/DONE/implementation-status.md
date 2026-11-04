# Completed Implementation Milestones: Plan (`PLAN.md`) vs. Reality

**Audit Date:** November 4, 2026  
**Reference Document:** `PLAN.md` (Dated 2026-08-07)  
**Status:** Landed & Verified Deliverables

---

## 1. Executive Summary

An audit of the codebase across all subprojects confirms that the **foundational offline architecture has been implemented in production**, with the backend protocol `sync/1` landed in API **v6.1.0**, the web extension reaching release **v6.1.15** with client-side incremental delta sync and data absence refresh policy, the styleguide at **v6.1.0**, and the Android client reaching release **v6.0.12** with full incremental delta sync and Room schema 31.

| Component | Target in PLAN.md | Actual Status in Codebase | Completed Deliverables |
|-----------|-------------------|---------------------------|------------------------|
| **`passly_android`** | 3.1.x | **6.0.12** (versionCode `69`) | **M0, M1 (Autonomy), M3 (Incremental Delta Sync)**: Local secret persistence, non-blocking unlock, hardware Keystore decoupled from biometrics, `core/sync` engine (`SyncEngine`, `DeltaSyncStrategy`, `ChangesApplier`, `BootstrapStrategy`, `ManifestReconciler`), Room schema 31 (`SyncState`), WorkManager scheduler (`SyncWorker`, `SyncScheduler`), fast delta refresh integrated into `FullDataRefreshExecutor` (full sync restricted strictly to empty local replica). |
| **`passly_api`** | 5.12.x | **6.1.0** (commits up to `2c021d5`) | **M2 (Full Protocol `sync/1`)**: `PassboltCe/Sync` plugin (`sync_log`, `/sync/meta`, `/sync/changes`, `/sync/bootstrap`, `/sync/manifest`, sync audit logging), adapted `OfflineMode` plugin. |
| **`passly_browser_extension`** | 6.0.13 | **6.1.15** (PR [#54](https://github.com/Svaroh/passly_browser_extension/pull/54)) | **E-A, E-B, E-B1, E-C, E-V, E-D, M3 (Incremental Delta Sync)**: Containerized workspace, offline session unlock, local-first reads, Seamless Offline Routing, offline-first passkeys, biometric unlock, and full client-side delta sync engine (`SyncEngine`, `DeltaSyncStrategy`, `ChangesApplier`, `BootstrapStrategy`, `ManifestReconciler`, `SyncStateLocalStorage`) enforcing strict data absence refresh policy. |
| **`passly_styleguide`** | v5.12.1 (fork) | **6.1.0** (tag `v6.1.0`) | Fully adapted for autonomous execution, offline states, and biometric auth dialogs. |

> [!NOTE]
> All pending milestones and unfulfilled requirements are tracked in the roadmap backlog in **[docs/TODO/roadmap-backlog.md](../TODO/roadmap-backlog.md)**.

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

### Milestone M2: Backend API — Protocol `sync/1` Complete (`passly_api`)
- **Reality:** ✅ **Fully Landed (commit `e033560f6d` and subsequent tests).**
  - Plugin established in `plugins/PassboltCe/Sync`.
  - Database migration `config/Migrations/20260812200000_V600CreateSyncLog.php` (`sync_log` table with `seq`, `entity_type`, `entity_id`, `op`, `created`).
  - Event listener `SyncLogModelListener` and recorder `SyncLogRecorderService` capturing mutations to `resources`, `secrets`, and `permissions`.
  - Endpoint `GET /sync/meta.json` (`SyncMetaController`, `SyncMetaService`).
  - Endpoint `GET /sync/changes.json` (`SyncChangesController`, `SyncChangesService`) supporting `since` cursor, limit, watermark settling (2s lag), 90-day retention checks (`410 Gone`), and dynamic ACL filtering.
  - Endpoint `GET /sync/bootstrap.json` (`SyncBootstrapController`, `SyncBootstrapService`, `SyncBootstrapDto`): Paginated initial snapshot with `header.pagination` for clean onboarding and fast recovery from `410 Gone`.
  - Endpoint `GET /sync/manifest.json` (`SyncManifestController`, `SyncManifestService`): Lightweight state reconciliation grouped by entity types.
  - Secret Access Audit Logging: `SyncSecretAccessLoggerTrait` registers `Resource.Secrets.sync` audit log action types and `password_sync` SIEM actions during bootstrap and delta sync.
  - Maintenance pruning: `SyncLogRetentionService` and CLI command `SyncLogPruneCommand` (`bin/cake sync_log_prune`).
  - Plugin `PassboltCe/OfflineMode` adapted with perpetual offline defaults.

---

### Milestone M3: Android Client Incremental Delta Sync (Requirement R3)
- **Reality:** ✅ **Fully Landed in v6.0.12.**
  - **Detailed Audit & Verification:** See dedicated report in **[docs/DONE/android-delta-sync-m3.md](android-delta-sync-m3.md)**.
  - **Delta Synchronization Strategies & UI Integration:**
    - `DeltaSyncStrategy`: Pulls `/sync/changes.json?since={localSeq}`, loops through paginated changes up to target watermark, and applies updates to Room DB.
    - `ChangesApplier`: Atomically applies entity additions, updates, and deletions for `resources`, `secrets`, and `permissions` inside a Room transaction while advancing `localSeq`.
    - `BootstrapStrategy`: Handles initial synchronization and transparent recovery when receiving `410 Gone` (cursor expired), fetching full encrypted snapshots from `/sync/bootstrap.json`.
    - `ManifestReconciler`: Lightweight periodic sanity check verifying local entity counts against `/sync/manifest.json`, triggering automatic recovery upon discrepancy.
    - `SyncEngine`: Master coordinator orchestrating delta sync, 410 Gone recovery fallback, manifest reconciliation, and telemetry.
    - `FullDataRefreshExecutor` Integration: Fully wired into `SyncEngine`. Enforces strict offline-first policy where full synchronization is executed **strictly only when local replica data is completely absent**. Whenever local data exists, rapid incremental delta sync is executed, eliminating UI blocking spinners on `HomeScreen` and `OtpScreen`.
  - **Database Migration (Room Schema 31):**
    - Created `SyncState` entity and `SyncStateDao` to persist `localSeq`, `lastSyncTimestamp`, and `lastManifestCheck`.
    - Implemented `Migration30to31.kt` with auto-migration and automated Room migration verification tests.
  - **Background WorkManager Integration:**
    - `SyncWorker`: Periodic and on-demand `CoroutineWorker` integrated with network connectivity constraints and exponential backoff.
    - `SyncScheduler`: Schedules periodic background sync (15-min intervals) and dispatches immediate manual sync requests.
  - **Koin DI Wiring:** Complete module registration in `SyncModule.kt` loaded into `KoinInitializer.kt`.
  - **Verification & Test Coverage:** 31 automated unit and E2E tests passing with 0 failures (`./gradlew :sync:testDebugUnitTest`), production debug APK built (`app-debug.apk` v6.0.11).

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
4. **Client Incremental Delta Sync & Data Absence Refresh Policy (v6.1.15, PR [#54](https://github.com/Svaroh/passly_browser_extension/pull/54)):**
   - **Detailed Audit & Verification:** See dedicated report in **[docs/DONE/browser-delta-sync-m3.md](browser-delta-sync-m3.md)**.
   - Implemented `sync/1` wire protocol engine (`SyncApiService`, `SyncStateLocalStorage`, `ChangesApplier`, `DeltaSyncStrategy`, `BootstrapStrategy`, `ManifestReconciler`, `SyncEngine`).
   - Enforced strict data absence synchronization policy in `FindAndUpdateResourcesLocalStorage`: full synchronization is strictly executed only when local replica data is completely absent; existing data triggers non-blocking incremental delta sync.
   - Background messaging ports (`passbolt.sync.sync-now`, `passbolt.sync.get-status`) via `SyncNowController`.
   - Verified by 72 sync unit tests and 5,157 full extension unit tests. Packed/unpacked artifacts rebuilt.

---

## 4. Pending Initiatives

The following unfulfilled roadmap items are tracked in **[docs/TODO/roadmap-backlog.md](../TODO/roadmap-backlog.md)**:
- **Milestone M4**: User-configurable sync preferences screen.
- **Milestone M5 & Step E-E**: Offline write queue (`pending_ops`) and conflict resolution (`sync_conflicts`).
- **Milestone M6**: Standalone emergency vault export (KDBX 4 / PGP-JSON).
