# Browser Extension Offline Workspace (`passly_browser_extension`)

This document outlines the architecture of the Passly multi-browser extension (Chrome MV3, Firefox, Safari), detailing the local user interface, Seamless Offline Routing, and the offline-first Passkey / WebAuthn implementation.

---

## 1. Architectural Breakthrough: Extension-Hosted UI

In vanilla Passbolt, the browser extension served primarily as a content script injecting an iframe on the remote server host (`https://passbolt.example.com/app/...`). If the server went down, accessing the vault workspace was impossible.

**Passly re-architects this into an autonomous workspace:**
- The entire workspace interface is bundled **inside the extension package itself**:
  - `webAccessibleResources/passbolt-iframe-app.html` (main workspace iframe),
  - `webAccessibleResources/workspace.html` (extension-hosted workspace container),
  - `webAccessibleResources/signin.html` (local unlock entry point).
- The content script connects a runtime port to the background page (`background_page`) via `PortManager` and mounts the `ExtBootstrapApp` React components from `passly_styleguide`.
- Consequently, viewing passwords, managing folders, search, and copying credentials require **no remote server availability whatsoever**.

---

## 2. Local Storage & Replica Preservation

The extension relies on `browser.storage.local` and Origin Private File System (OPFS):

- `ResourceLocalStorage`, `FolderLocalStorage`, `GroupLocalStorage`, and `SecretLocalStorage` maintain local account state.
- Secrets are persisted locally in OpenPGP-armored format.
- `FindSecretService.findByResourceId` follows a **local-first** policy: the local store is checked first, and network requests are only triggered for unknown items when connectivity is present.

### Replica Purge Fix (E-B1):
Previously, the extension executed `LocalStorageService.flush()` on every browser startup (`OnStartUpService`) or during authentication loop failures when the server could not be reached.
**This was completely resolved:**
- Browser startup clears only session-scoped state (`flushSessionScoped`).
- An offline condition is never interpreted as a user logout.
- The local replica persists indefinitely across browser restarts and device reboots.

---

## 3. Seamless Offline Routing (PR #45)

Prior to version 6.1.0, attempting to open the workspace while offline caused prolonged loading and browser network errors:
`Unable to reach the server, you are not connected to the network`.

### How Seamless Routing Works:
1. **Transport Failure Interception:** `checkAuthStatusService` intercepts network failures and returns `{ isOffline: true }` instead of throwing an unhandled exception.
2. **Immediate Redirection:** If the server domain cannot be resolved or times out, the extension immediately routes the user to the local `workspace.html` or `signin.html`.
3. **Local Passphrase Validation:** `AuthLoginController` validates the user's passphrase locally using WebCrypto and openpgp.js without performing a remote GPGAuth challenge.
4. **Non-blocking QuickAccess:** The QuickAccess popup mounts immediately using cached records without waiting for background API health checks.

---

## 4. Offline-First Passkeys & WebAuthn

Passly includes an embedded WebAuthn / Passkey credential provider:

- **Local Credential Matching:** When a website requests authentication via `navigator.credentials.get()`, the extension queries matching Passkey records by Relying Party ID directly from local storage (`passkeyVaultResourceService`).
- **Zero Network Delay:** Background API refresh delays are removed from the assertion flow.
- **Native WebAuthn Fallback:** If no matching passkey is found in the local vault, the request falls back seamlessly to the platform's native authenticator (Touch ID, Windows Hello, YubiKey).

---

## 5. Biometric Unlock

Starting with extension versions 6.0.28+:
- The passphrase prompt dialog (`InputPassphrase`) in `passly_styleguide` supports biometric authentication using WebAuthn User Verification.
- `biometricAuthService` securely caches a wrapped session token, enabling users to unlock their vault with Touch ID or Windows Hello without re-typing their master passphrase.
- Verified with end-to-end Playwright test suites: `scripts/test-biometric-e2e.js`.
