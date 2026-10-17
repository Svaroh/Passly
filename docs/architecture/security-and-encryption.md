# Security & Encryption Architecture

Passly enforces a comprehensive multi-layered defense-in-depth security model. Even in the event of an adversary acquiring full access to the device file system or gaining root access to the backend infrastructure, user secrets cannot be extracted in plaintext.

---

## 1. Four Levels of Encryption

| Level | Protected Scope | Mechanism & Implementation |
|-------|-----------------|----------------------------|
| **L1. E2E (End-to-End)** | Password values, TOTP seeds, notes, v5 metadata | **OpenPGP (gopenpgp / openpgp.js)**. Secrets are encrypted client-side prior to transit and stored in the local database **in the exact PGP-armored format** received from the server. The server never observes plaintext secrets. |
| **L2. Storage Encryption** | Relational tables, relations, cached metadata, offline queue | **SQLCipher (AES-256-CBC, HMAC-SHA512)** on Android; **AES-GCM (WebCrypto)** / OPFS in the browser extension. The entire database is encrypted at the storage page level. |
| **L3. DB Key Protection** | Unlocking key for the SQLCipher storage file | A 256-bit random key produced by a CSPRNG, wrapped with AES-GCM and stored in the hardware-backed keystore (**Android Keystore** / StrongBox / Apple Keychain). |
| **L4. Platform Level** | Screenshots, system backups, app task switchers | `FLAG_SECURE` on all sensitive UI screens; `allowBackup=false` in the Android manifest; clipboard auto-clear timers. |

---

## 2. Decryption Policy & Memory Protection

- **Plaintext is never written to disk.** The disk stores exclusively doubly-encrypted data: OpenPGP ciphertext within an encrypted SQLCipher container.
- Decryption of secrets (L1) occurs **strictly on-demand in volatile memory** (e.g., when a user taps "reveal", during autofill, or when copying to clipboard).
- The user's private key and decrypted passphrase are held in a secure in-memory cache (`PassphraseMemoryCache`) subject to strict inactivity timeouts and immediate zeroization upon app backgrounding, locking, or explicit sign-out.

---

## 3. Critical Invariant: Decoupling DB Keys from Biometrics

Android developer guidelines often suggest creating hardware keys with:
```kotlin
setUserAuthenticationRequired(true)
setInvalidatedByBiometricEnrollment(true)
```

> [!CAUTION]
> **Binding the database encryption key (L3) to biometric enrolment is strictly prohibited.**

If the database encryption key is tied to biometric enrolment, routine OS events:
- registering a new finger,
- temporary biometric sensor faults,
- altering lock screen credentials,
will trigger a `KeyPermanentlyInvalidatedException` inside Android Keystore, causing the hardware security module to permanently destroy the key.

In a cloud-first application that caches data, this merely forces a cache eviction and re-login. For Passly, which operates under **R0 (server-independent autonomy)**, this would result in **permanent, irreversible data loss while offline**.

### How Passly implements biometrics:
1. **The database master key (L3)** is held in Android Keystore without biometric invalidation flags. It is accessible to the application process whenever the device is booted and unlocked by the user.
2. **Biometrics protect only the user's passphrase** or an in-memory session unlock token.
3. If biometric enrolment changes, the application falls back to asking the user for their master passphrase. The underlying local database remains intact and accessible.

---

## 4. Threat Matrix, Controls, and Conscious Trade-offs

| Threat | Passly Control Mechanism | Design Trade-off / Rationale |
|--------|--------------------------|------------------------------|
| **Theft of an unlocked device** | Inactivity auto-lock, biometric/PIN requirement, immediate zeroization of `PassphraseMemoryCache`. | Defends against opportunist snooping. |
| **Database file extraction (ADB / Root / direct flash readout)** | SQLCipher full-page encryption (AES-256 + HMAC-SHA512). Encryption key held in hardware Keystore/StrongBox. `allowBackup=false`. | Without extracting the hardware-bound key, the database cannot be decrypted. |
| **Full backend server compromise** | All secrets are end-to-end encrypted with OpenPGP. The server database stores only armored ciphertext. | Even a root compromise of the server yields no plaintext credentials. |
| **Access revocation while a client is offline** | **Deliberate architectural trade-off:** the local replica retains access indefinitely until the device reconnects and processes a `deletion` event. | **Direct consequence of requirement R0.** Because remote wipes are prohibited, the valid security control is **rotating secrets on the server**, rather than revoking permissions. Administrators must be aware: a compromised lost device requires secret rotation. |
| **Secret leakage into system logs (Logcat / Sentry)** | Static analysis rules (Detekt/ESLint), stripped debug trees in release builds, sensitive wrapper types with redacted `toString()`. | Prevents plaintext secrets from leaking into crash reports or logging pipelines. |
| **Device hardware failure / Keystore loss** | Standalone encrypted export functionality (**R8**). | A local database without its hardware key is unrecoverable; regular offline exports are recommended. |
