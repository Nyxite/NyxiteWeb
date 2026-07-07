# 01 — Architecture

## 1.1 Shape: offline-capable, layered, unidirectional, all in the browser

The app is a **static-export SPA** ([00 §0.5](00-overview.md)): a CDN serves immutable assets; **all logic runs in the browser**. It is **offline-capable** — the in-browser store (IndexedDB) is the source of truth the UI renders from, and the network is a background reconciler. With the service worker installed and the local subset cached, the user can read, edit, draw, organize, and search the cached subset with no connectivity; changes queue and sync when the relay/REST is reachable.

It follows a **layered (Clean-ish) architecture** with **unidirectional data flow** in the presentation layer, mirroring the Android client's discipline ([android 01](https://github.com/Nyxite/NyxiteAndroid)) and the server's separation of `Domain`/`Contracts`/`Crypto` from host concerns:

```
┌──────────────────────────────────────────────────────────────────┐
│ Presentation (React 19 + shadcn/ui + Tailwind, App Router routes)  │
│   Route segments ── components ── hooks ── stores (Zustand)         │
└───────────────▲───────────────────────────────────────┬──────────┘
                │ selectors / query results   actions     │
┌───────────────┴───────────────────────────────────────▼──────────┐
│ Domain (pure TypeScript, no DOM/IO/React deps)                     │
│   Use cases · entities/types · repository interfaces · policy      │
└───────────────▲───────────────────────────────────────┬──────────┘
                │                                         │
┌───────────────┴───────────────────────────────────────▼──────────┐
│ Data                                                              │
│  Repositories (impl) ── orchestrate:                             │
│   • LocalStore (IndexedDB / Dexie)   ← single source of truth     │
│   • ApiClient (fetch + TanStack Query) ← REST over ciphertext     │
│   • RelayClient (@microsoft/signalr)  ← encrypted CRDT relay      │
│   • CryptoEngine (WebCrypto + WASM)   ← all encryption            │
│   • CrdtEngine (Yjs)                  ← text merge                │
│   • KeyVault (non-extractable CryptoKey in IndexedDB) ← key mat.   │
│   • BlobCache (Cache Storage / IDB)   ← cached ciphertext/plain    │
│   • AuthManager (native fetch/WebAuthn;                            │
│       oidc-client-ts for enterprise) ← server tokens, share tokens │
└──────────────────────────────────────────────────────────────────┘
```

The **domain layer is platform-free** (no `window`, `fetch`, React, or library imports), so it is unit-testable in plain Node/jsdom and could later be shared. Decryption happens in the data layer; the presentation layer only ever receives already-plaintext state.

## 1.2 The cardinal rule: plaintext never crosses the network boundary

There is exactly one place where plaintext becomes ciphertext and vice-versa: the **CryptoEngine**, invoked by repositories. Enforced by construction:

- `ApiClient` and `RelayClient` accept and return **opaque `Uint8Array` / framed blobs and structural JSON only** — they have no method that takes a domain content object and **no reference to `CryptoEngine`**. They cannot serialize plaintext by accident.
- Repositories are the only components that hold both a `CryptoEngine` and a network client. Every outbound content payload passes through `CryptoEngine.seal(...)`; every inbound one through `CryptoEngine.open(...)`.
- The in-browser store may hold decrypted content for the cached subset; it is gated by the account session and the [17-security.md](17-security.md) at-rest model. Decrypted plaintext and unwrapped keys are **never** placed in `localStorage`/`sessionStorage`, URL, or any telemetry.
- An **ESLint boundary rule** (`eslint-plugin-boundaries`, [18](18-build-ci-testing.md)) fails the build if a network module imports the crypto/domain-content types, or if a type carrying a plaintext field reaches a network client. This is the web analogue of the Android `konsist` architecture test ([android 01 §1.2](https://github.com/Nyxite/NyxiteAndroid)).

## 1.3 Presentation: unidirectional data flow

- **Routes** are App Router segments rendered fully client-side (static export). Each screen is composed of shadcn/ui components and feature components.
- **Server state** (structure, manifests, versions, shares — all ciphertext or structural) is fetched and cached with **TanStack Query**; queries return *decrypted* view models because the query functions call repositories, which decrypt.
- **Client/UI state** (selection, editor mode, dialogs, presence, sync badges) lives in small **Zustand** stores. State is immutable and fully describes what to render (loading / empty / error / locked / content).
- **One-shot effects** (navigation, toasts, share-sheet, biometric/passphrase prompts) are dispatched imperatively from hooks, not stored.
- Heavy, expensive-to-rebuild editor state (the Yjs doc, in-progress ink strokes) is owned by an editor controller kept alive for the open file, not re-created per render.

## 1.4 Domain: use cases & repository interfaces

Representative use cases (each a single-responsibility function/class):

- Structure: `ObserveProjectTree`, `CreateFolder`, `MoveFile`, `RenameFile`, `SetSyncPolicy`, `SoftDeleteFile`.
- Content: `OpenFileForRead`, `OpenFileForEdit`, `ApplyTextEdit`, `AppendInkStroke`, `SaveInkPage`.
- Sync: `RunDeltaSync`, `PullManifest`, `DownloadBlob`, `UploadBlob`, `ReconcileFile`.
- Collaboration: `JoinDocument`, `SubmitUpdate`, `BroadcastAwareness`, `LeaveDocument`, `SnapshotDocument`.
- Keys/identity: `EnrollBrowser`, `ApproveDevice`, `GenerateRecoveryKey`, `RecoverFromKey`, `RotateFileKey`, `PublishPublicKeys`.
- Sharing: `CreateAccountShare`, `CreateLinkShare`, `OpenShareLink`, `RevokeShare`, `ListShares`.
- History/search: `ListVersions`, `DiffVersions`, `RestoreVersion`, `Search`, `Reindex`.

Repository interfaces (`FileRepository`, `StructureRepository`, `KeyRepository`, `ShareRepository`, `SyncRepository`, `CollabRepository`, `SearchRepository`, `VersionRepository`) live in domain; implementations live in data ([03](03-project-structure.md)).

## 1.5 Data layer components

| Component | Responsibility | Backed by |
|-----------|----------------|-----------|
| `LocalStore` | Single source of truth for structure, sync state, cached metadata/plaintext, search index | IndexedDB via **Dexie** ([04](04-local-data-model.md)) |
| `ApiClient` | Typed REST over `/api/v1`, ciphertext bodies, problem+json errors, idempotency | `fetch` + **TanStack Query** ([05](05-api-client.md)) |
| `RelayClient` | SignalR `RelayHub` connection, join/submit/awareness, reconnect | **@microsoft/signalr** ([09](09-realtime-collaboration.md)) |
| `CryptoEngine` | seal/open framed objects, hybrid-HPKE wrap/unwrap, hybrid sign/verify, content address, recovery KDF | **WebCrypto + hpke-js + libsodium + hash-wasm + a WASM PQC lib** (ML-KEM-768/ML-DSA-65) ([06](06-cryptography.md)) |
| `CrdtEngine` | Apply/encode Yjs updates, state vectors, snapshots | **Yjs** ([09](09-realtime-collaboration.md)) |
| `KeyVault` | Hold/seal the identity-key handles; gate unlock | **non-extractable WebCrypto `CryptoKey` in IndexedDB** ([07](07-key-and-device-management.md)) |
| `BlobCache` | Store/evict cached ciphertext and decrypted blobs (ink/binary) | **Cache Storage** + IndexedDB ([16](16-offline-and-pwa.md)) |
| `AuthManager` | Server access/refresh tokens, silent refresh, relay socket ticket, guest share-token | **native `fetch` + WebAuthn API** (default); **oidc-client-ts** for the enterprise Keycloak option ([14](14-authentication.md)) |

## 1.6 Workers & threading

Browsers are single-threaded on the main thread; CPU-bound crypto/CRDT/diff must not block UI:

- **Crypto Worker** — a dedicated **Web Worker** hosts the WASM (BLAKE3, Argon2id, and the PQC ML-KEM-768/ML-DSA-65 halves) and bulk AES-GCM / hybrid-HPKE work so framing, content addressing, Argon2id derivation, and large-blob seal/open run off the main thread. WebCrypto is available in workers; transfer `ArrayBuffer`s to avoid copies. **[P]**
- **Diff** for version history runs in the crypto worker (or a small diff worker) to keep large diffs smooth ([12](12-version-history.md)).
- **CRDT merge** runs on the main thread for the active editor (Yjs is fast and tightly coupled to editor bindings), but batch/catch-up merges of large logs may be offloaded; benchmark and decide per [19](19-open-questions.md).
- **Service Worker** ([16](16-offline-and-pwa.md)) handles the app-shell cache and offline navigation; it **does not** decrypt content or hold keys (it is a separate, less-trusted context — see [17](17-security.md)).
- Long sync runs use the regular event loop with cooperative chunking; **Background Sync API** (where supported) retries queued pushes after reconnect.

## 1.7 Error & connectivity model

- The UI renders from `LocalStore`; network failures degrade gracefully to "offline / N changes pending".
- A per-file **sync state machine** — canonical set `Synced`, `PendingPush`, `PendingPull`, `Downloading`, `Uploading`, `Conflicted`, `Rotating`, `Excluded`, `Error(code)` (identical to the Android client and [04](04-local-data-model.md)/[08](08-sync-engine.md)) — is persisted and surfaced as small badges.
- Server `problem+json` codes map to a typed `NyxiteApiError` (e.g. `key_generation_stale` → refetch + rotate; `share_revoked`/`link_expired` → mark dead; `excluded_content` → never retry upload; `429` → backoff with `Retry-After`). See [05 §5.4](05-api-client.md).
- The **fragment key is never lost to a reload**: guest sessions capture `location.hash` on first paint and hold it in memory only ([13](13-sharing.md), [17](17-security.md)).

## 1.8 Multi-account / session isolation

The app is **multi-account from v1.0.0** ([14 §14.7](14-authentication.md)). Per-account state must not leak across accounts:

- An **account-scoped session** (`UserSession`) owns the unlocked identity-key handles, the server's tokens, repositories, and account-scoped data sources: the account's **IndexedDB database** (named per `accountId`), its `BlobCache` partition, and its search index. It is created after login + key-unlock and **cleared on lock/logout/switch** (key handles dropped, in-memory plaintext discarded).
- No use case can touch decrypted key material before unlock; no account can read another's store. The active account is swapped by tearing down the prior session and rooting the UI at the new account's data.
- Because the bundle is instance-agnostic ([00 §0.5](00-overview.md)), each account also carries its **instance host** (API/relay/share bases, plus the OIDC authority for the enterprise Keycloak option), enabling a personal and a shared instance side by side.
