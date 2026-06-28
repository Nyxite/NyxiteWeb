# 04 — Local Data Model

The in-browser store (`LocalStore`, Dexie over IndexedDB) is the **offline-first source of truth** the UI renders from ([01 §1.1](01-architecture.md)). It mirrors the server's structure graph, caches decrypted names/metadata and the decrypted **local subset**, tracks per-file sync state, mirrors CRDT/version pointers for offline catch-up, and backs the search index. It is the web analogue of the Android Room schema ([android 04](https://github.com/Nyxite/android)), constrained by the browser being the most limited peer ([00 §0.3](00-overview.md)).

## 4.1 Storage engine, DB naming, at-rest reality

- **IndexedDB via Dexie.** The app is **multi-account from v1.0.0** ([01 §1.8](01-architecture.md)), so there is **one Dexie database per account**, named `nyxite-{accountId}`. Account databases are never cross-queried; switching accounts tears down the session and opens the other DB ([01 §1.8](01-architecture.md)).
- A tiny separate **account registry** DB (`nyxite-accounts`) lets the app render the account switcher and pick a DB to open *before* any account is unlocked. One record per known account: `accountId` (PK), `host` (instance base host), `displayName`, `lastUsedAt`, `dbName`. It holds no content, names, or keys.
- **At-rest reality (important):** unlike Android, IndexedDB has **no SQLCipher equivalent** — Dexie writes are plaintext-at-rest by default. So cached decrypted names/metadata/body **and** wrapped key blobs sit in IndexedDB largely unprotected by the platform. The at-rest protection model — an optional app-managed **vault key** that encrypts sensitive columns, and an optional app lock — lives in [17-security.md](17-security.md). This document defines the schema; [17](17-security.md) defines what of it is encrypted-at-rest and under what key.
- Identity **private keys** and unwrapped **file keys** are **not** stored as bytes here. They live as **non-extractable WebCrypto `CryptoKey` handles** in a dedicated `KeyVault` IndexedDB store ([07](07-key-and-device-management.md), [§4.4](#44-in-memory-session-state)).
- Large blobs (ink/binary ciphertext, decrypted ink for kept files, cached snapshots) live in `BlobCache` (Cache Storage + IDB metadata), not in these tables ([16](16-offline-and-pwa.md)).

## 4.2 Versioning & migrations

- Dexie schema is **versioned** via `db.version(n).stores({...}).upgrade(tx => …)`; migrations are **forward-only**, matching the server discipline.
- A `meta` table row records app version + **crypto-frame version** expectations so a client detects and refuses to misread newer encrypted frames ([06 §6.3](06-cryptography.md)).
- Schema strings (below) are asserted against a golden snapshot in tests ([18](18-build-ci-testing.md)).

## 4.3 Tables

Dexie schema strings list **indexed** keys only (`&` unique, `++` auto-increment, `*` multi-entry); non-indexed fields live on the stored object and are typed by the interface. IDs are **UUIDv7** strings. `*Enc` fields hold **ciphertext** (`Uint8Array`); the decrypted form is cached in a sibling field and is the at-rest concern of [17](17-security.md).

```ts
// db.ts — new Dexie(`nyxite-${accountId}`)
this.version(1).stores({
  projects:         'id, ownerId, deletedAt',
  folders:          'id, projectId, parentFolderId, deletedAt',
  files:            'id, projectId, folderId, ownerId, syncState, keepInBrowser, contentType, [keepInBrowser+lastOpenedAt], deletedAt',
  fileKeys:         'id, fileId, [fileId+keyId], keyId, memberId, shareId',
  crdtUpdatesCache: '++localId, crdtDocId, [crdtDocId+seq], direction, acked',
  versions:         'id, fileId, [fileId+seq]',
  blobs:            'ref, contentHash, decrypted',
  shares:           'id, fileId, folderId, projectId, kind, linkTokenHash, revokedAt',
  devices:          'id, isThisDevice',
  userKeys:         '[userId+keyId], userId, email',          // own keys + directory cache
  recoveryBlobMeta: 'userId',
  settings:         'id',                                      // single 'me' row
  syncCursor:       'scope',                                   // per-project + global
  outbox:           '++localId, kind, nextAttemptAt, idempotencyKey',
  searchMeta:       'fileId',                                  // see §4.5
  meta:             'key',
});
```

### projects
```ts
interface ProjectRow {
  id: string; ownerId: string;
  nameEnc: Uint8Array; name?: string;          // decrypted cache ([17])
  metadataEnc?: Uint8Array; metadata?: unknown;
  keepInBrowserDefault?: 'keep' | 'dontKeep';  // client-local; cascades to descendants ([16 §16.2])
  createdAt: string; updatedAt: string; deletedAt?: string; // RFC3339
}
```

### folders
```ts
interface FolderRow {
  id: string; projectId: string; parentFolderId?: string;     // null = project root
  nameEnc: Uint8Array; name?: string;
  metadataEnc?: Uint8Array;
  keepInBrowserDefault?: 'keep' | 'dontKeep';                  // cascades ([16 §16.2])
  createdAt: string; updatedAt: string; deletedAt?: string;
}
```
Acyclic parent chain enforced client-side.

### files
```ts
interface FileRow {
  id: string; projectId: string; folderId?: string; ownerId: string;
  nameEnc: Uint8Array; name?: string;                          // decrypted cache ([17])
  contentType: 'markdown' | 'plaintext' | 'ink' | 'sourcecode' | 'office' | 'image' | 'binary'; // immutable
  syncPolicy: 'server-default' | 'excluded';                  // server enum only ([08 §8.2])
  keepInBrowser?: 'keep' | 'dontKeep' | 'inherit';            // CLIENT-LOCAL pinning, never sent ([16 §16.2])
  currentVersionSeq?: number;                                  // head pointer
  crdtDocId?: string;                                          // client-allocated UUIDv7; text types only
  keyGeneration: number;                                       // monotonic FK rotation counter; drives 412
  contentHash?: Uint8Array;                                    // BLAKE3 of head plaintext (cache)
  syncState: SyncState;                                        // §4.6
  cacheState: 'none' | 'metadataOnly' | 'ciphertextCached' | 'plaintextCached';
  metadataEnc?: Uint8Array; metadata?: unknown;               // ink LWW version-vector lives here ([08 §8.5])
  updatedAt: string; createdAt: string; deletedAt?: string;
  lastOpenedAt?: string;                                       // convenience-cache eviction ([16 §16.3])
}
```
Effective keep-in-browser resolves `file.keepInBrowser` (`inherit`/null) → nearest ancestor `keepInBrowserDefault` → account default (`dontKeep`).

### fileKeys
```ts
interface FileKeyRow {
  id: string;                          // UUIDv7 surrogate (mirrors server)
  fileId: string; keyId: string; generation: number;
  memberId?: string; shareId?: string; // CHECK: exactly one non-null
  wrappedKey: Uint8Array;              // as fetched, for re-upload/rotation
  // unwrapped FK is NEVER persisted as bytes — see UserSession.fkHandles ([§4.4], [07])
}
```
Invariant: exactly one of `memberId`/`shareId` set. The schema's `[fileId+keyId]` plus `memberId`/`shareId` indexes back the server's two partial-unique constraints.

### crdtUpdatesCache (offline catch-up + submit outbox)
```ts
interface CrdtUpdateRow {
  localId?: number; crdtDocId: string;
  seq?: number;                        // null until server-assigned (outbound)
  updateEnc: Uint8Array; keyId: string; authorId?: string;
  direction: 'inbound' | 'outbound'; acked: boolean; createdAt: string;
}
```
Backs offline replay and the relay submit queue ([09](09-realtime-collaboration.md)).

### versions (snapshot/version cache)
```ts
interface VersionRow {
  id: string; fileId: string; seq: number;
  contentHash: Uint8Array; blobRef: string; sizeCipher: number;
  keyId: string; authorId?: string; createdAt: string;
}
```
Metadata cache for history; the snapshot ciphertext itself is in `blobs`/`BlobCache` ([12](12-version-history.md)).

### blobs
```ts
interface BlobRow {
  ref: string;                         // blob_ref / content address
  contentHash: Uint8Array; sizeCipher: number;
  location: string;                    // Cache Storage key / IDB location of the ciphertext
  decrypted: boolean;                  // whether a decrypted copy exists in BlobCache ([16])
  cachedAt: string;
}
```
Holds **pointers + flags**; actual bytes live in `BlobCache` ([16](16-offline-and-pwa.md)).

### shares
```ts
interface ShareRow {
  id: string; fileId?: string; folderId?: string; projectId?: string;
  kind: 'user_grant' | 'link'; granteeId?: string; linkTokenHash?: Uint8Array;
  permission: 'read' | 'write'; createdBy: string;
  createdAt: string; expiresAt?: string; revokedAt?: string;
}
```
The link **token + fragment key are never stored** unless the user explicitly opts to keep a created link — then only as a vault-encrypted entry, clearly marked ([13](13-sharing.md), [17](17-security.md)).

### devices
```ts
interface DeviceRow { id: string; label?: string; pubkey: Uint8Array; enrolledAt: string; revokedAt?: string; isThisDevice: boolean; }
```

### userKeys (own keys + directory cache)
```ts
interface UserKeyRow {
  userId: string; email?: string; keyId: string;
  x25519: Uint8Array; ed25519: Uint8Array; generation: number;
  fetchedAt: string;                   // refresh-on-use for directory entries ([13])
}
```
Holds both the account's own published public keys and cached public keys of other users used to wrap shares.

### recoveryBlobMeta
```ts
interface RecoveryBlobMetaRow { userId: string; version: number; kdf: KdfParams; updatedAt: string; }
```
Non-secret KDF params + version only; the recovery ciphertext lives server-side and is never persisted locally in plaintext ([07](07-key-and-device-management.md)).

### settings
```ts
interface SettingsRow { id: 'me'; settingsEnc?: Uint8Array; settings?: unknown; updatedAt: string; }
```
Mirrors `GET/PUT /me/settings` (ciphertext); decrypted cache gated by [17](17-security.md).

### syncCursor
```ts
interface SyncCursorRow { scope: string; /* `project:{id}` | `global` */ cursor: string; updatedAt: string; }
```
Stores the **opaque** delta-sync cursor verbatim — never parsed ([05 §5.1](05-api-client.md)).

### outbox (queued local changes; drives Background Sync)
```ts
interface OutboxRow {
  localId?: number;
  kind: 'structure' | 'blobUpload' | 'crdtSubmit' | 'delete' | 'keyRotate' | 'shareCreate' | 'shareRevoke';
  targetId: string; payloadRef: string; idempotencyKey: string;  // uuidv7, stable across retries
  attempts: number; nextAttemptAt: string; lastError?: string;
}
```
Reliable, retryable, idempotent sync; replayed by the sync engine and the **Background Sync API** where supported ([05 §5.5](05-api-client.md), [08](08-sync-engine.md), [16](16-offline-and-pwa.md)).

### searchMeta / search index
The full-text index is a **separate MiniSearch store** persisted to its own IndexedDB blob, not a Dexie table (MiniSearch serializes its own structure). `searchMeta` tracks per-file index freshness (`fileId`, indexed `contentHash`, `indexedAt`) so `data-search` can reindex incrementally as content is decrypted ([11](11-search.md)). The index covers only the **local subset** and is never uploaded.

## 4.4 In-memory session state (`UserSession`)

Secret material is **never persisted in plaintext**; it lives only in an account-scoped, in-memory `UserSession` created after login + unlock and cleared on lock/logout/switch ([01 §1.8](01-architecture.md)):

```ts
interface UserSession {
  accountId: string; host: string;                 // instance base host
  identity: { x25519: CryptoKey; ed25519: CryptoKey };  // non-extractable handles ([07])
  fkHandles: Map<string /*keyId*/, CryptoKey>;     // unwrapped file keys, non-extractable, in-memory only
  tokens: { accessExp: number; /* bytes held by AuthManager, not here */ };
  guestFragmentKey?: CryptoKey;                     // link/guest: held in memory only, captured from location.hash ([13],[17])
}
```

- FK content keys are unwrapped (HPKE) on demand and held as **non-extractable `CryptoKey`** so they can be used by the `CryptoEngine` but never read back as bytes; the persisted `fileKeys.wrappedKey` is the only durable form.
- Tokens, fragment keys, and unwrapped keys are **never** written to `localStorage`/`sessionStorage`, URLs, or telemetry ([01 §1.2](01-architecture.md), [17](17-security.md)).

## 4.5 Search index — see [11](11-search.md)

(Stored separately from Dexie; see [§4.3 searchMeta](#searchmeta--search-index).)

## 4.6 Per-file sync state machine

`FileRow.syncState` is a persisted enum driving badges and the sync engine ([08](08-sync-engine.md)), **identical** to Android and [01 §1.7](01-architecture.md):

`Synced` · `PendingPush` · `PendingPull` · `Downloading` · `Uploading` · `Conflicted` (LWW loser retained) · `Rotating` (key rotation in progress) · `Excluded` · `Error(code)`.

## 4.7 Storage quota interplay

IndexedDB + Cache Storage share the origin's Storage-API quota and are subject to eviction. The local subset is bounded; `navigator.storage.persist()` is requested to avoid silent eviction of keys/subset, and eviction/eviction-recovery, quota pressure, and the keep-in-browser vs convenience-cache split are specified in [16](16-offline-and-pwa.md).

## 4.8 What is **not** stored

- No server-side search index (none exists); the local MiniSearch index covers the local subset only.
- No identity private key or unwrapped FK as bytes (non-extractable `CryptoKey` in `KeyVault`, [07](07-key-and-device-management.md)).
- No bearer/refresh/share tokens (held by `AuthManager`, in memory / [14](14-authentication.md)).
- No fragment key (in-memory `UserSession` only, [13](13-sharing.md)).
