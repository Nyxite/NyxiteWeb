# 07 — Key & Device Management

Covers the identity keypair, treating a **browser as a device**, in-browser key storage, the recovery key, the lock model, and FK/identity rotation. This is the Phase-0 foundation ([00 §0.9](00-overview.md)); nothing later retrofits it. It mirrors [server 07](https://github.com/Nyxite/server)/[08](https://github.com/Nyxite/server) and the sibling [android 07](https://github.com/Nyxite/android), adapted to the browser's lack of a hardware keystore. Crypto primitives live in [06-cryptography.md](06-cryptography.md); at-rest threat detail is in [17-security.md](17-security.md).

## 7.1 Key hierarchy (web client view)

```
Vault key (non-extractable WebCrypto AES-GCM CryptoKey, in IndexedDB)   [P]
   └─ wraps → Identity bundle blob (X25519 priv ‖ Ed25519 priv), AES-GCM
                 └─ unwrapped only into memory (UserSession) for the live session
                      └─ HPKE-unwraps → File Keys (FK, per file)
                                          (FK = non-extractable AES-GCM CryptoKey handle)
Recovery phrase (user-held BIP39) ──Argon2id──► wrapping key ──► AES-GCM escrow of identity bundle (server-opaque)
Browser-enrollment keypair (per browser) ──► used to receive the identity bundle when this browser enrolls
```

- **Public keys** (X25519 + Ed25519) go to the server directory via `PUT /keys`; **private keys never leave the browser** ([server 08 §8.3](https://github.com/Nyxite/server)).
- Per the system rule ([06 §6.2](06-cryptography.md)): FK wrapping to members and enrollment transport use **HPKE** (public-key targets); the at-rest vault wrap and recovery escrow use **AES-256-GCM** (symmetric keys).

## 7.2 First sign-in (brand-new account)

1. User authenticates with Keycloak (OIDC + PKCE + TOTP) → access token ([14](14-authentication.md)).
2. App checks the directory (`GET /keys/directory?userId=me`). If the user has **no identity keypair**, generate **X25519 + Ed25519** (libsodium, in the worker), then **publish the public keys** via `PUT /keys` (signed directory entry, [06 §6.10](06-cryptography.md)).
3. Create the **vault key** and store the wrapped identity bundle ([§7.3](#73-keyvault-design-p)).
4. Generate the **recovery key** and escrow ([§7.6](#76-recovery-flow)). The user is **forced through the recovery-key flow before setup completes**.
5. **Enroll this browser** as a device ([§7.5](#75-browser-enrollment)).

## 7.3 KeyVault design **[P]**

**The central web tension.** libsodium needs the **raw private-key bytes** to do X25519/Ed25519 ([06 §6.13](06-cryptography.md)), so the identity bundle **cannot** be a non-extractable WebCrypto key — unlike on Android there is no hardware keystore to hold it. FK content keys are different: they are only ever used for AES-GCM, so they *can* be non-extractable `CryptoKey` handles ([06 §6.7](06-cryptography.md)).

**Resolution:**

| Material | At rest | In session |
|----------|---------|------------|
| **Identity bundle** (X25519 priv ‖ Ed25519 priv) | AES-256-GCM **wrapped under the vault key**, stored as a blob in IndexedDB | unwrapped to raw bytes held **only in memory** in `UserSession`, passed to the worker for libsodium ops; `.fill(0)` on drop |
| **Vault key** | **non-extractable WebCrypto AES-GCM `CryptoKey`** stored in IndexedDB (structured-clone of the handle) | used to unwrap the bundle once per unlock; never exportable |
| **FK handles** | not persisted unwrapped (wrapped form cached) | **non-extractable AES-GCM `CryptoKey`** in the worker, short-lived ([06 §6.7](06-cryptography.md)) |

The vault key is the browser's nearest analogue to a keystore: its bits cannot be read back out, so the wrapped identity bundle in IndexedDB is useless without the live `CryptoKey` handle (and, if the lock gate is on, without passing it — [§7.4](#74-lock-model-p)).

**Session-only vs persistent ("remember this browser"):**

- **Persistent** (default for a trusted personal browser): the vault-key handle and wrapped bundle persist in IndexedDB across reloads; on return the app unwraps into memory after the lock gate.
- **Session-only** (shared/public machine, or user choice): the vault key is created ephemerally and **not persisted** (or held only in memory); closing the tab/profile loses it and the bundle must be re-obtained via enrollment or recovery. No identity material survives the session.

## 7.4 Lock model

> **Ratified** ([19 §19.1](19-open-questions.md)): non-extractable vault key; **WebAuthn passkey where available, else passphrase**; idle auto-lock; **persistent ("remember this browser") by default**, session-only as a user choice ([§7.3](#73-keyvault-design-p)).

Browsers have **no Keystore/StrongBox and no OS-level per-use auth**; the vault is the analogue. The **unlock gate** protects use of the vault key:

- **Preferred: WebAuthn / passkey.** A platform authenticator gates unlock; the WebAuthn assertion releases (or derives material to release) the vault key for the session. Phish-resistant, no secret to remember.
- **Alternative: passphrase.** A user passphrase run through **Argon2id (or PBKDF2 where required)** derives a key that wraps the vault key; entering it unlocks. Used where passkeys are unavailable.
- **Idle auto-lock timeout** (configurable; default on). On lock — explicit, idle-timeout, or tab close under session-only mode — the app **drops the in-memory identity bundle and all `FileKeyHandle`s** and zeroizes raw buffers ([06 §6.12](06-cryptography.md)). Structure (encrypted names cached) may still render, but no content decrypts until re-unlock.

This is best-effort: an unlocked, compromised tab can still be abused while open ([17](17-security.md)); the gate limits the at-rest window, not a live-XSS window.

## 7.5 Browser enrollment

A **browser profile counts as a device** ([00 §0.8](00-overview.md)). A browser that has the account token but **not** the identity private key cannot decrypt content; the UI shows a clear **"enroll this browser"** state. Two paths:

### (A) Device-to-device approval
1. New browser generates a **browser-enrollment X25519 keypair** and registers: `POST /devices { label, pubkey }` → `{ deviceId, status:"pending", pairingCode, qrPayload }`.
2. It displays the **QR (primary) + 6–8 digit numeric code (fallback)**.
3. An **already-enrolled** device/browser approves after explicit confirmation: `POST /devices/{id}/approve { wrappedIdentityKey }`, where `wrappedIdentityKey = HPKE-seal(identity bundle → newBrowserPubkey)` ([06 §6.5](06-cryptography.md)). The server relays the opaque blob in `pending_key_blob`.
4. The new browser fetches it **once**: `GET /devices/me/enrollment` → `{ wrappedIdentityKey }` (**single-use**; the server clears the blob and marks the device `active`), and **unwraps with its enrollment private key**, then wraps the bundle under a fresh local vault key ([§7.3](#73-keyvault-design-p)).

### (B) Recovery-phrase bootstrap
When no enrolled device is available, the user enters the recovery phrase and recovers the bundle directly ([§7.6](#76-recovery-flow)).

**List / revoke:** a Security/Keys area lists this browser and other enrolled devices (label + fingerprint) with **revoke** (`DELETE /devices/{id}`) ([15](15-ui-and-navigation.md)).

## 7.6 Recovery flow

- **Generate**: a **BIP39 24-word phrase** (256-bit, checksummed, vetted word list) generated in the worker and **shown once**. The flow shows a **non-dismissable, no-escrow warning** — *losing all browsers and this phrase = permanent, unrecoverable data loss* — and **requires re-entry to confirm** before proceeding. The phrase is never persisted by the app.
- **Escrow**: derive the wrapping key via **Argon2id** (m = 64 MiB, t = 3, p = 1; params in `recovery_blobs.kdf_params`) in the worker, then **AES-256-GCM-seal** the identity bundle ([06 §6.9](06-cryptography.md)). Blob shape `{ version, kdf, nonce, ciphertext, tag }`, **AAD = `userId ‖ version`**; upload the opaque blob + non-secret `kdf_params` via `PUT /recovery`. **No server escrow.**
- **Restore**: `GET /recovery` (+ `kdf_params`) → derive key from the entered phrase → unwrap the bundle → store wrapped under a fresh vault key → **re-enroll this browser**.
- **Re-issue** (rotation): re-wrap the escrow under a new phrase and re-`PUT`; the old phrase is invalidated client-side.

## 7.7 File-key lifecycle

- **Create**: generate FK (CSPRNG → non-extractable handle, [06 §6.7](06-cryptography.md)), encrypt initial content, HPKE-wrap FK to the owner's public key, `POST /files` with the owner wrapped key + initial ciphertext ([05](05-api-client.md)).
- **Open**: `GET /files/{id}/keys` → the caller's `wrapped_key`; HPKE-unwrap with the identity private key → `FileKeyHandle`; cache the **wrapped** form locally, hold the unwrapped handle in the worker only ([04 §4.2](04-local-data-model.md)).
- **Generation check**: every frame carries `key_id` (FK + generation). On `412 key_generation_stale`, refetch keys and re-encrypt with the current generation ([05 §5.4](05-api-client.md)).

## 7.8 Rotation & revocation (client-driven forward secrecy)

When a share is revoked ([13](13-sharing.md)), a remaining member's browser drives **`RotateFileKey`** so removed members cannot read **future** content:

1. Generate a **new FK** (new `keyId`, `generation + 1`).
2. **Re-encrypt** the current head and produce a fresh snapshot under the new FK.
3. **Re-wrap** the new FK to every **remaining** member's public key (HPKE).
4. `POST /files/{id}/keys/rotate { newKeyId, generation, wrappedKeys[], newHeadRef }`; the server bumps `key_generation`. If a concurrent rotation already won, the server returns **`409`** — abandon and refetch the new generation.
5. **In-flight old-key updates** are accepted until the rotation commits; after commit they get **`412 key_generation_stale`** and must be re-encrypted under the new key.

The server simultaneously **drops the removed member's ACL grant** (instant cutoff) even before rotation completes; already-downloaded ciphertext cannot be un-seen (inherent). The file shows `Rotating` state ([04 §4.4](04-local-data-model.md), [08](08-sync-engine.md)).

**Identity rotation (browser compromise):** revoke the device (`DELETE /devices/{id}`); if the browser may be compromised, **rotate the identity keypair** (`PUT /keys`, new generation) and re-wrap FKs in the background — heavier (touches every file the user holds), a Phase-4 security action with clear UX.

## 7.9 Multi-account isolation

The app is multi-account from v1.0.0 ([01 §1.8](01-architecture.md), [14 §14.7](14-authentication.md)). Each account has its **own IndexedDB database** (named per `accountId`) holding its **own vault key, wrapped identity bundle, cached wrapped FKs**, and search index; FK handles live in the per-account worker session. Switching accounts **tears down the in-memory session** — drops the identity bundle and all FK handles, locks the prior vault — and roots the UI at the new account's store. No account can read another's keys or content.

## 7.10 Forward links

- At-rest threat detail (IndexedDB exposure, XSS, the unlocked-tab window, zeroization limits) → [17-security.md](17-security.md).
- Open validation items (passkey vs passphrase default, idle-timeout default, Argon2id final tuning in-browser, persistent-storage eviction of the vault) → [19-open-questions.md](19-open-questions.md).
