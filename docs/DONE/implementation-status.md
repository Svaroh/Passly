# Implementation Status: Plan (`PLAN.md`) vs. Reality

**Audit Date:** October 16, 2026  
**Reference Document:** `/Volumes/SRC/passly/PLAN.md` (Dated 2026-08-07)

---

## 1. Executive Summary

The initial roadmap document `PLAN.md` originally stated:  
> *“Status: design plan (implementation not yet started)”*

An audit of the codebase across all four subprojects reveals that **the foundational offline architecture has been largely implemented in production**, with several domains (notably the browser extension, styleguide, and WebAuthn / Passkeys) progressing beyond initial targets to release **v6.1.0**.

| Component | Target in PLAN.md | Actual Status in Codebase | Implementation Progress |
|-----------|-------------------|---------------------------|-------------------------|
| **`passly_android`** | 3.1.x | **6.0.8** (commit `daebc1379`) | Completed **M0** and **M1 (Autonomy)**; local secret persistence, non-blocking unlock, hardware Keystore decoupled from biometrics, `core/sync` foundation. |
| **`passly_api`** | 5.12.x | **6.1.0** (tag `v6.1.0`) | Landed `PassboltCe/Sync` plugin (`sync_log` table, `/sync/meta`, `/sync/changes`), adapted `OfflineMode` plugin for perpetual offline defaults. |
| **`passly_browser_extension`** | 6.0.13 | **6.1.0** (tag `v6.1.0`) | Landed **E-A, E-B, E-B1, E-C, E-V** + exceeded roadmap: implemented **Seamless Offline Routing (PR #45)**, offline-first passkeys, biometric unlock. |
| **`passly_styleguide`** | v5.12.1 (fork) | **6.1.0** (tag `v6.1.0`) | Fully adapted for autonomous execution, offline states, and biometric auth dialogs. |

---

## 2. Milestone-by-Milestone Audit (`PLAN.md`)

### Milestone M0: Specification & Sync Foundation
- **Plan:** `docs/sync-protocol.md`, Kotlin `core/sync` module skeleton, WorkManager dependency.
- **Reality:** ✅ **Fully Landed.**
  - Specification codified in `passly_android/docs/sync-protocol.md` and `docs/architecture/sync-protocol.md`.
  - Module `core/sync` established in `passly_android/core/sync`: includes `SyncStatus`, `SyncResult`, `SyncTrigger`, `SyncBackoff` (with unit test suite), `SyncStatusRepository`, and `NetworkStateProvider`.

---

### Milestone M1: Android Autonomy (Requirements R0, R2, R5, R6)
- **Plan:** `Secret` table creation, Room migration, local-first secret decryption, non-blocking offline unlock, Keystore master key unbound from biometrics.
- **Reality:** ✅ **Fully Landed (commit `1e1aea3c4` and subsequent refactors).**
  - **Database:** Created `secrets` table (`SecretEntity`), `SecretsDao`, and Room migration `Migration28to29.kt` (originally estimated as 23➔24; current schema version is 29).
  - **Local-first Reading:** `SecretInteractor` queries `GetLocalSecretUseCase` first. If an entry is cached locally, no network connection is attempted.
  - **Non-blocking Unlock (R6):** Implemented `BackgroundSignInExecutor` and `HasLocalReplicaUseCase`. With a local replica present, the vault opens immediately upon local passphrase verification; server GPGAuth session establishment runs asynchronously in a detached coroutine.
  - **Network Timeouts:** Connection timeout trimmed to 10s; overall call timeout capped at 60s; server PGP and RSA keys fetched concurrently.
  - **Critical Path Audit:** `AuthenticatedOperationRunner` no longer blocks offline reads when session tokens are absent (`requiresSession = false`); `KeyInspectorViewModel` reads local `users` data; the creation button on `HomeScreen` is preserved during background sync failures.
  - **Keystore Security (R5):** Storage master key preserved without `setUserAuthenticationRequired` or `setInvalidatedByBiometricEnrollment`.

---

### Milestone M2: Backend API (`passly_api`)
- **Plan:** `PassboltCe/Sync` plugin, `sync_log` table, `afterSave`/`afterDelete` hooks, `/sync/{meta, changes, bootstrap, manifest}` endpoints.
- **Reality:** 🟡 **Partially Landed (Core Delta Operational).**
  - ✅ Plugin established in `plugins/PassboltCe/Sync`.
  - ✅ Database migration `config/Migrations/20260812200000_V600CreateSyncLog.php` (`sync_log` table with `seq`, `entity_type`, `entity_id`, `op`, `created`).
  - ✅ Event listener `SyncLogModelListener` and recorder `SyncLogRecorderService` capturing mutations to `resources`, `secrets`, and `permissions`.
  - ✅ Endpoint `GET /sync/meta.json` (`SyncMetaController`, `SyncMetaService`).
  - ✅ Endpoint `GET /sync/changes.json` (`SyncChangesController`, `SyncChangesService`) supporting `since` cursor, limit, watermark settling (2s lag), 90-day retention checks (`410 Gone`), and dynamic ACL filtering (revoked permissions surface as `deletions`).
  - ✅ Maintenance pruning: `SyncLogRetentionService` and CLI command `SyncLogPruneCommand` (`bin/cake sync_log_prune`).
  - ✅ Plugin `PassboltCe/OfflineMode` adapted with perpetual offline defaults.
  - ⏳ **Pending Endpoints:**
    - `GET /sync/bootstrap.json` (`SyncBootstrapController`) for paginated initial sync snapshots.
    - `GET /sync/manifest.json` (`SyncManifestController`) for lightweight reconciliation.
    - Special `sync` audit log classification for secret queries during delta transfers.

---

### Milestone M3: Android Delta Client (Requirement R3)
- **Plan:** Client-side delta synchronization replacing full refresh (`core/fulldatarefresh`).
- **Reality:** ⏳ **Planned / In Progress.**
  - Module `core/sync` provides status repositories and backoff primitives, but `DeltaSyncStrategy`, `ChangesApplier`, `BootstrapStrategy`, and `ManifestReconciler` remain to be connected.
  - Android currently continues to rely on `core/fulldatarefresh` with silent background tracking.

---

### Milestone M4: Sync Management UI (Requirement R4)
- **Plan:** Force-sync button on Home, dedicated "Sync" settings screen with interval selectors and neutral offline indicators.
- **Reality:** 🟡 **Partially Landed.**
  - Offline status messaging and neutral phrasing have been integrated across screens.
  - The dedicated sync preferences screen is slated for delivery alongside the delta client.

---

### Milestone M5: Offline Writes & Conflict Handling (Requirement R7)
- **Plan:** `pending_ops` queue table, optimistic UI, conflict handling via `sync_conflicts`.
- **Reality:** ⏳ **Planned.** Offline mutations currently pause until network availability is restored.

---

### Milestone M6: Emergency Export & Polish (Requirement R8)
- **Plan:** Offline standalone export to KDBX 4 / PGP-JSON from the local store.
- **Reality:** ⏳ **Planned.**

---

## 3. Browser Extension Audit (Phase 2 / Section 11a)

Section 11a of `PLAN.md` charted milestones for extension autonomy:

| Step | Planned Objective | Actual Status in v6.1.0 |
|------|-------------------|-------------------------|
| **E-A** | Local `workspace.html` & `signin.html` in `webAccessibleResources` | ✅ **Landed.** UI rendered within extension container without remote web host. |
| **E-B** | `OfflineSessionService`, local passphrase check, offline handling | ✅ **Landed.** Unlocking operates with zero network dependencies. |
| **E-B1** | Halt `LocalStorageService.flush()` on startup | ✅ **Landed.** Local replica persists indefinitely across browser restarts. |
| **E-C** | Local-first read path for resources, profile, groups | ✅ **Landed.** Vault items read from local storage before checking API. |
| **E-V** | Automated Playwright offline test suite | ✅ **Landed.** `test/offline/verifyOfflineWorkspace.js` verified passing. |
| **E-D** | Offline UI states in styleguide | ✅ **Landed.** Filters (including Trash), dialogs, and banners adapted for offline mode. |
| **E-E** | Offline write queue | ⏳ **Planned.** |

### Features Delivered Beyond Initial Plan:
1. **Seamless Offline Routing (PR #45):** If the server host fails to resolve or drops connection, the extension seamlessly switches to the bundled `workspace.html` without presenting browser network failure screens.
2. **Offline-first Passkeys:** Direct WebAuthn credential filtering from local vault storage (`passkeyVaultResourceService`), stripping background API delays from passkey ceremonies.
3. **Biometric Vault Unlock:** WebAuthn User Verification integration (Touch ID / Windows Hello) for unlocking the extension workspace without entering a passphrase.
