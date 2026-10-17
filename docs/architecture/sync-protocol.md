# Passly Sync Protocol `sync/1`

**Status:** Normative Specification.  
Backend landed in plugin `PassboltCe/Sync` (`passly_api`); client foundation landed in module `core/sync` (`passly_android`).

The protocol is completely **platform-independent**. Android, Browser Extension, iOS, and Desktop clients implement the exact same wire format and transaction apply rules.

---

## 1. Design Invariants

1. **The client cursor is a monotonic integer (`seq`), never a timestamp.** Clock drift, identical timestamps across concurrent transactions, and timezone issues cannot cause lost events.
2. **The local database is a self-sufficient store.** Synchronization is an optional enhancement. Absence of connectivity degrades data freshness, never availability.
3. **`deletions` include access revocations.** If a user's permission to an entry is removed or they are evicted from a group, this surfaces to the client as an ordinary `deletion`. This ensures the delta mechanism is complete and self-contained.
4. **No protocol error is fatal to local data.** A `410 Gone` response instructs the client to perform a bootstrap snapshot on top of existing local data, never to wipe the local store.
5. **Page transaction atomicity:** Each batch of changes is applied in a single local ACID database transaction along with the updated cursor. If a network failure occurs mid-sync, the next run resumes precisely from the last committed cursor.

---

## 2. Vocabulary

| Term | Definition |
|------|------------|
| **`seq`** | Monotonic 64-bit unsigned change-journal sequence number (`BIGINT UNSIGNED AUTO_INCREMENT`) assigned to each event by the server. |
| **`cursor`** | The highest `seq` value that a client has successfully received, applied, and committed locally. |
| **`epoch`** | An opaque unique identifier representing the server database instance. A change in `epoch` (e.g., database restored from backup) forces a fresh bootstrap. |
| **`watermark`** | `MAX(seq)` among journal entries older than `WATERMARK_LAG` (default: 2 seconds). Prevents missing events committed out-of-order during long transactions. |
| **`upsert`** | Entity creation or modification, returned as a complete DTO. |
| **`deletion`** | Identifier of an entity that was either deleted on the server or is no longer visible to the calling user due to ACL changes. |

---

## 3. Server Change Journal (`sync_log`)

The journal is maintained in a dedicated table via migration `V600CreateSyncLog`:

```sql
CREATE TABLE sync_log (
  seq         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  entity_type VARCHAR(32) NOT NULL,
  entity_id   CHAR(36) NOT NULL,
  op          VARCHAR(16) NOT NULL,
  created     DATETIME(6) NOT NULL,
  INDEX (entity_type, entity_id),
  INDEX (created)
);
```

### Journal Rules:
- **One row per event, zero payload.** Payloads are resolved dynamically at read time so that permissions are evaluated against the **current** caller visibility.
- **Permission event translation:** Modifications to `permissions` and `groups_users` are translated into journal events on the target resources or folders, as those are the entities whose visibility changed.
- **Retention:** Journal entries older than 90 days are pruned by `bin/cake sync_log_prune`. If a client queries with a cursor older than the oldest retained row, the server returns `410 Gone`.

---

## 4. API Endpoints Specification

### 4.1 `GET /sync/meta.json`
Returns protocol metadata. Invoked during account setup and periodic compatibility checks.

```json
{
  "header": { "status": "success" },
  "body": {
    "protocol": "sync/1",
    "epoch": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "min_client": "6.0.0",
    "max_page_size": 500,
    "retention_days": 90
  }
}
```

### 4.2 `GET /sync/changes.json?since=<seq>&limit=500`
Primary delta synchronization endpoint.

#### Query Parameters:
- `since` (required): Current client cursor (non-negative integer).
- `limit` (optional, default 500): Maximum number of journal events to process.

#### Response Example:
```json
{
  "header": { "status": "success" },
  "body": {
    "cursor": 148213,
    "has_more": true,
    "upserts": {
      "resources": [
        {
          "id": "e2c3b4a1-0000-4000-8000-000000000001",
          "name": "Production Database Credentials",
          "username": "dbadmin",
          "uri": "https://db.internal",
          "description": "Primary Postgres cluster",
          "modified": "2026-10-16T12:00:00Z",
          "permission": { "type": 15 },
          "favorite": false
        }
      ],
      "secrets": [
        {
          "id": "s1c3b4a1-0000-4000-8000-000000000001",
          "resource_id": "e2c3b4a1-0000-4000-8000-000000000001",
          "data": "-----BEGIN PGP MESSAGE-----\nwcBMA...",
          "modified": "2026-10-16T12:00:00Z"
        }
      ],
      "folders": [],
      "users": [],
      "groups": [],
      "resource_types": [],
      "metadata_keys": []
    },
    "deletions": {
      "resources": [
        "d4c3b4a1-0000-4000-8000-000000000099"
      ],
      "folders": [],
      "users": [],
      "groups": []
    }
  }
}
```

### 4.3 `GET /sync/bootstrap.json?limit=500&page=N`
Full snapshot bootstrap for newly enrolled devices or recovery from `410 Gone`.

- Supplies resources **alongside their PGP-armored secrets in the same stream**, ensuring full autonomy (**R0**) is established before initial sign-in completes.
- The `cursor` is pinned on page 1 and returned unchanged across subsequent pagination calls.
- Applied incrementally; interrupted transfers resume seamlessly from the failing page.

### 4.4 `GET /sync/manifest.json?type=resources`
Compact self-healing reconciliation list (~90 bytes per record):
```json
{
  "header": { "status": "success" },
  "body": [
    {
      "id": "e2c3b4a1-0000-4000-8000-000000000001",
      "modified": "2026-10-16T12:00:00Z",
      "secret_modified": "2026-10-16T12:00:00Z"
    }
  ]
}
```
The client diffs this manifest against local records and fetches missing or divergent items in targeted batches (`filter[has-id][]`).

---

## 5. Client Synchronization Algorithm

```
┌─────────────────────────────────────────────────────────────┐
│                       Client Invocation                     │
│                    sync(trigger: Trigger)                   │
└──────────────────────────────┬──────────────────────────────┘
                               │
                 [Network up?] ─── NO ──► Return SyncResult.Skipped(NoNetwork)
                               │
                              YES
                               │
                   [Flush pending_ops queue]
                               │
                [Cursor set?] ─── NO ──► Execute bootstrap()
                               │
                              YES
                               │
        ┌──────────────► Call GET /sync/changes.json?since=cursor
        │                      │
        │             [Status 410 Gone?] ─── YES ──► Execute bootstrap() over local store
        │                      │
        │                      NO
        │                      │
        │               [ACID Transaction]:
        │               - apply upserts
        │               - apply deletions
        │               - commit cursor = page.cursor
        │                      │
        └─────── YES ── [has_more == true?]
                               │
                               NO
                               │
        [trigger == MANUAL or reconcileDue()?] ── YES ──► reconcileAll() (manifest)
                               │
                               NO
                               │
                    Return SyncResult.Success
```

---

## 6. Error Handling Matrix

| Scenario | Client Action | Local Data Integrity |
|----------|---------------|----------------------|
| **No internet / DNS failure / server down** | Status "Offline", exponential backoff. | Fully accessible for reads, search, decryption, and operations. |
| **401 Unauthorized / session expired** | Background re-auth attempt; if failing, pause sync. | Local data untouched; vault remains open. |
| **410 Gone (cursor pruned)** | Transparent incremental bootstrap over local store. | Existing records are updated/preserved, never erased. |
| **Server epoch change** | Re-bootstrap over local database. | Local records updated and reconciled. |
| **5xx Server Error / timeout** | Backoff: 30s → 1m → 5m → 15m → 1h → 6h. | UI and local operations remain responsive. |
| **Metadata decryption failure** | Row flagged as `undecryptable`. | Page transaction commits successfully for remaining items. |
