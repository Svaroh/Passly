# Milestone M3: Browser Extension Client Incremental Delta Sync (`sync/1`)

**Release Version:** `6.1.16` (initial release `6.1.15`)  
**Target Subproject:** `passly_browser_extension`  
**Reference Specification:** [`docs/architecture/sync-protocol.md`](../architecture/sync-protocol.md)  
**Pull Request:** [#54](https://github.com/Svaroh/passly_browser_extension/pull/54)  
**Status:** Completed, Verified, and Merged into feature branch  

---

## 1. Overview

Milestone M3 delivers the client-side delta synchronization engine for the Passly browser extension (`passly_browser_extension`). Prior to this milestone, the browser extension relied on full background re-fetching of all paginated resources (`findAllForLocalStorage()`), transferring entire encrypted datasets and re-indexing all entries even when no changes or minimal changes had occurred.

M3 operationalizes the client-side implementation of the `sync/1` wire protocol, directly consuming the server change journal (`sync_log`) and endpoints delivered in Milestone M2 (`/sync/changes.json`, `/sync/bootstrap.json`, `/sync/manifest.json`, and `/sync/meta.json`).

Crucially, this release establishes and enforces the **strict data absence synchronization policy**:
> **Full synchronization / bootstrap is strictly executed only when local replica data is completely absent.**

When local cached data already exists, the extension executes exclusively lightweight, non-blocking incremental delta synchronization, dropping sync times from tens of seconds to milliseconds and eliminating UI freezes.

The implementation satisfies all foundational Passly offline invariants:
- **R0 (Autonomous Store):** Local storage and OPFS act as an autonomous, self-sufficient offline replica.
- **R3 (Monotonic Delta & Atomic Application):** Sync cursor is a monotonic 64-bit integer tracking server change sequences. Delta pages and cursor state apply atomically across local storage collections.
- **R4 (Transparent Recovery):** Server retention purges (`410 Gone`) or epoch changes transparently fall back to non-destructive bootstrap over existing data without data loss or user disruption.
- **Non-blocking Execution:** Synchronization operates asynchronously in background services, never stalling user interaction or vault decryption.

---

## 2. Architecture & Delivered Components

```
passly_browser_extension/src/all/background_page/
├── controller/sync/
│   ├── syncNowController.js
│   └── syncNowController.test.js
├── event/
│   ├── resourceEvents.js (passbolt.sync.* listeners)
│   └── resourceEvents.test.js
└── service/
    ├── api/sync/
    │   ├── syncApiService.js
    │   └── syncApiService.test.js
    ├── local_storage/
    │   ├── syncStateLocalStorage.js
    │   ├── syncStateLocalStorage.test.js
    │   ├── findAndUpdateResourcesLocalStorageService.js (data absence policy)
    │   └── findAndUpdateResourcesLocalStorageService.test.js
    └── sync/
        ├── changesApplier.js
        ├── changesApplier.test.js
        ├── deltaSyncStrategy.js
        ├── deltaSyncStrategy.test.js
        ├── bootstrapStrategy.js
        ├── bootstrapStrategy.test.js
        ├── manifestReconciler.js
        ├── manifestReconciler.test.js
        ├── syncEngine.js
        └── syncEngine.test.js
```

### 2.1. Persistent Sync State: `SyncStateLocalStorage`
- **Tracked Metadata:** Persists `cursor` (monotonic 64-bit sequence integer), `epoch` (server generation token), `lastSyncTimestamp`, `lastReconcileTimestamp`, and `failureCount`.
- **Atomic Updates:** Provides atomic mutators `saveCursor()`, `recordSuccess()`, `recordFailure()`, and `reset()`.

### 2.2. API Client: `SyncApiService`
- Implements direct consumption of all `sync/1` backend endpoints with full URL parameter serialization:
  - `getMeta()`: `GET /sync/meta.json`
  - `getChanges(since, limit)`: `GET /sync/changes.json?since={since}&limit={limit}`
  - `getBootstrap(page, limit, cursor)`: `GET /sync/bootstrap.json?page={page}&limit={limit}&cursor={cursor}`
  - `getManifest(type)`: `GET /sync/manifest.json?type={type}`

### 2.3. Atomic Batch Application: `ChangesApplier`
- Applies batched `upserts` and `deletions` across all client entity types:
  - `resources`: Updates `ResourceLocalStorage` cache and synchronizes encrypted secret payloads directly into Origin Private File System (OPFS) via `SecretLocalStorageService`.
  - `folders`: Updates `FolderLocalStorage`.
  - `users`: Updates `UserLocalStorage`.
  - `groups`: Updates `GroupLocalStorage`.
  - `resource_types`: Updates `ResourceTypeLocalStorage`.
- Cascades deletions: Deleting a resource automatically removes both its database metadata and its OPFS payload file.

### 2.4. Delta Engine: `DeltaSyncStrategy`
- Iteratively fetches `GET /sync/changes.json` while `has_more === true`.
- Applies changes batch by batch and advances `cursor` atomically after each page.
- Catches `410 Gone` (purged cursor past server 90-day retention or epoch reset) and returns `{ status: "gone" }` to signal transparent recovery.

### 2.5. Initial & Recovery Bootstrap: `BootstrapStrategy`
- Fetches paginated initial snapshots from `GET /sync/bootstrap.json`.
- Pins the server cursor from page 1 and passes it through all subsequent pages, guaranteeing a consistent snapshot even if new mutations occur concurrently on the server during the multi-page pull.
- Overlays data non-destructively over existing local records.

### 2.6. Access Revocation & State Reconciliation: `ManifestReconciler`
- Queries compact identifier manifest `GET /sync/manifest.json?type=resources`.
- Compares local resource IDs against server manifest and purges orphaned local records whose visibility was revoked on the server.
- Updates `lastReconcileTimestamp` upon completion.

### 2.7. Central Orchestrator: `SyncEngine`
- Central entry point `sync(trigger)` managing trigger types: `AUTO`, `MANUAL`, `FORCE_BOOTSTRAP`, and `RECONCILE`.
- Prevents concurrent synchronization using an asynchronous `Lock` mutex queue.
- Verifies network connectivity (`window.navigator.onLine`); safely returns `{ status: "offline" }` when offline without throwing exceptions or degrading the UI.
- Transparent fallback: When `DeltaSyncStrategy` reports `gone`, `SyncEngine` automatically transitions to `BootstrapStrategy` without error or data loss.

### 2.8. UI Refresh Integration & Data Absence Policy
- Updated `FindAndUpdateResourcesLocalStorage.findAndUpdateAll`:
  - **Condition:** Evaluates whether local replica data exists (`localStorageResourceCollection.length > 0`).
  - **Existing Data:** Calls `syncEngine.sync(SyncTrigger.AUTO)`. The legacy full-table `findAllForLocalStorage()` request is completely bypassed.
  - **Empty Cache:** Calls legacy initial download / bootstrap to populate local storage and pins the initial cursor in `SyncStateLocalStorage`.

### 2.9. Port Messaging Controllers: `SyncNowController` & `resourceEvents.js`
- Exposes runtime port event listeners:
  - `passbolt.sync.sync-now`: Triggers on-demand manual delta sync and manifest reconciliation.
  - `passbolt.sync.get-status`: Returns current sync cursor, epoch, last sync timestamp, and failure counters to UI components.

---

## 3. Verification & Test Suite

The implementation is verified by automated test suites covering all sync layers:

| Test Suite | Tests | Verification Focus |
|---|:---:|---|
| **`syncApiService.test.js`** | 6 | Wire protocol serialization, parameter encoding, 410 Gone propagation, and payload parsing. |
| **`syncStateLocalStorage.test.js`** | 8 | Cursor persistence, epoch handling, error backoff tracking, and state reset. |
| **`changesApplier.test.js`** | 4 | Atomic batch upserts, OPFS secret payload persistence, and cascading deletions. |
| **`deltaSyncStrategy.test.js`** | 4 | Multi-page pagination (`has_more`), cursor advancement, and 410 Gone interception. |
| **`bootstrapStrategy.test.js`** | 2 | Pinned cursor consistency across pages and error propagation. |
| **`manifestReconciler.test.js`** | 2 | Reconciliation diffing, access revocation pruning, and empty manifest handling. |
| **`syncEngine.test.js`** | 7 | Mutex locking, offline skipping, auto/manual triggers, and transparent 410 fallback. |
| **`syncNowController.test.js`** | 3 | Background port message handling (`sync-now`, `get-status`) and failure reporting. |
| **`resourceEvents.test.js`** | 2 | Integration of sync port message dispatchers in background event bus. |
| **`findAndUpdateResourcesLocalStorageService.test.js`** | 34 | Strict data absence refresh policy and fallback execution. |

### Full Extension Verification & E2E Testing Suite
- **Sync-specific unit suites:** 10 suites, 72 tests passed (100%).
- **Full extension unit test suite:** 697 test suites, 5,157 unit tests passed (`npm run test:unit`) with 0 failures and 0 errors.
- **Code hygiene & linting:** ESLint passed with 0 errors and 0 warnings (`npm run lint:eslint`).
- **Dedicated Chromium E2E Suite (`e2e/delta-sync.e2e.js`, `npm run test:e2e:sync`):**
  - Live DEV backend protocol contracts (`/sync/meta.json`, `/sync/changes.json`, `/sync/manifest.json`, `/sync/bootstrap.json`): **PASSED**
  - Headless Chromium CDP runtime attachment to active extension context: **PASSED**
  - Strict data absence refresh policy (cold-start bootstrap vs warm delta sync): **PASSED**
  - `ChangesApplier` atomic batch mutations, in-place updates, and cascading pruning: **PASSED**
  - Transparent `410 Gone` cursor expiration recovery to pinned bootstrap: **PASSED**
  - `ManifestReconciler` access revocation pruning: **PASSED**
  - Network disconnection offline mode resilience (`Network.emulateNetworkConditions`): **PASSED**
- **Full Suite E2E (`npm run test:e2e:all`):**
  - `e2e/offline-vault.e2e.js`: **ALL E2E & OFFLINE RESILIENCE CHECKS PASSED 100%**
  - `e2e/style-verification.e2e.js`: **DEDICATED STYLE E2E TEST COMPLETED SUCCESSFULLY**
  - `e2e/delta-sync.e2e.js`: **ALL INCREMENTAL DELTA SYNC (sync/1) E2E CHECKS PASSED 100%**
- **Artifact Build:** Unpacked Chromium MV3 bundle rebuilt successfully with version `6.1.16` in `build/chromium-mv3-unpacked/manifest.json`.
