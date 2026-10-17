# Android Autonomous Client Architecture (`passly_android`)

This document details the internal architecture of the Passly Android native client, covering local persistence, local-first data access, non-blocking authentication, and the synchronization engine.

---

## 1. Local Storage Layer (Room + SQLCipher)

The Android client maintains an encrypted SQLite database per user account:
`sha256(userId)_resources.db`.

### 1.1 The `Secret` Table (Migration 28 ➔ 29)
To fulfill requirement **R0 (full offline autonomy)**, secrets are persisted directly on the device.

```kotlin
@Entity(
    tableName = "secrets",
    foreignKeys = [
        ForeignKey(
            entity = ResourceEntity::class,
            parentColumns = ["id"],
            childColumns = ["resource_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [
        Index(value = ["resource_id"], unique = true)
    ]
)
data class SecretEntity(
    @PrimaryKey
    @ColumnInfo(name = "resource_id")
    val resourceId: String,
    @ColumnInfo(name = "secret_id")
    val secretId: String,
    @ColumnInfo(name = "armored_data")
    val armoredData: String,
    @ColumnInfo(name = "modified")
    val modified: String,
    @ColumnInfo(name = "fetched_at")
    val fetchedAt: Long
)
```

- Secrets are strictly associated with parent resources with cascading deletes (`CASCADE`).
- `armored_data` holds the original PGP-armored ciphertext block encrypted with the user's public key.
- `SecretsDao` provides atomic Room operations: `upsert`, `deleteByResourceId`, `getByResourceId`, and reactive `Flow` queries.

---

## 2. Local-First Read Path

Prior to the offline architecture overhaul, revealing a password or generating TOTP triggered a remote API request (`GET /secrets/resource/{id}.json`).

### New Flow in `SecretInteractor`:
```
UI requests secret (view, copy, TOTP, autofill)
    │
    ▼
[GetLocalSecretUseCase] ─── Found locally? ───► YES ──► In-memory PGP decrypt ──► Return result
    │
    NO (newly added resource, pending sync)
    │
    ▼
[Check Network Availability]
    ├─ No Network ──► Error: "Secret not available locally"
    └─ Network Up ──► FetchSecretUseCase (API) ──► UpsertLocalSecretsUseCase ──► Decrypt
```

All cached and synced passwords open **instantly with zero network dependencies**.

---

## 3. Non-Blocking Autonomous Unlock (R6)

Previously, signing in (`AuthViewModel.performFullSignIn`) blocked the user if the server was unreachable, waiting for socket timeouts (30–60s) before reporting an error.

### Architecture of `BackgroundSignInExecutor`:
1. **Local Replica Probe (`HasLocalReplicaUseCase`):** If an existing local database and encrypted private key are present for the account, the vault unlocks **immediately**.
2. **Local Passphrase Validation:** The passphrase is authenticated locally via `gopenpgp` by unlocking the private key in memory.
3. **Asynchronous Session Handshake:**
   - Following local passphrase verification, the user navigates directly to the **Home Screen**.
   - Server GPGAuth negotiation and JWT retrieval proceed asynchronously in the background (`BackgroundSignInExecutor`) within a detached coroutine scope.
   - If the server is offline or unreachable, the application simply transitions to a "Sync Paused" state without disrupting the user.

### Network Stack Hardening:
- `connectTimeout` reduced to **10 seconds**.
- An overarching `callTimeout` ceiling of **60 seconds** prevents indefinite hanging.
- Server PGP and RSA public keys are fetched in **parallel**, halving cold-start network handshake times.
- Transport exceptions (`IOException`, `UnknownHostException`, `SocketTimeoutException`) are correctly categorized as "Offline" rather than authentication failures.

---

## 4. Critical Path Audit & Local Hardening

All user journeys have been audited for hidden network dependencies:

- **`AuthenticatedOperationRunner`:** The `requiresSession = false` flag enables local workflows (inspecting details, viewing passwords, TOTP) without requiring an active remote session.
- **`KeyInspectorViewModel`:** Key details load from the local `users` table instead of issuing an online `/users/me.json` query.
- **`HomeScreen`:** The resource creation capability (`canCreateResource`) is preserved when background refresh fails.
- **`DataRefreshTrackingFlow`:** Background sync operates silently, without loading spinners, modal blockers, or error snackbars.

---

## 5. The `core/sync` Module

A dedicated `core/sync` module hosts the core delta sync primitives:

- **`SyncStatus`:** Lifecycle states (`Idle`, `Running`, `Offline`, `AwaitingRetry`, `Failed`).
- **`SyncResult`:** Execution outcomes (`Success`, `Skipped`, `RetryNeeded`, `Error`).
- **`SyncTrigger`:** Execution triggers (`MANUAL`, `BACKGROUND_PERIODIC`, `APP_START`, `NETWORK_RESTORED`).
- **`SyncBackoff`:** Exponential backoff retry ladder:
  `30s` ➔ `1m` ➔ `5m` ➔ `15m` ➔ `1h` ➔ `6h` (with jitter).
- **`SyncStatusRepository`:** Reactive sync state exposed via `StateFlow<SyncStatus>`.
- **`NetworkStateProvider`:** System-level network monitoring via Android's `ConnectivityManager`.

---

## 6. Android Passkey Provider Integration

The client integrates with Android's Credential Manager:
- `PasskeyCredentialProviderActivity` is optimized for offline operation.
- Passkey credentials are queried directly from the local database.
- `RefreshSessionUseCase` has been removed from the hot path during passkey assertion.
