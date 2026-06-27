# 05 — REST API Client

`ApiClient` implements the server REST surface ([server 04](https://github.com/Nyxite/server)) under `src/data/api` ([03 §3.1](03-project-structure.md)). It speaks **JSON for structure/metadata** and **binary streams (`Uint8Array`) for ciphertext**, and it has **no reference to `CryptoEngine`** — it can never serialize plaintext ([01 §1.2](01-architecture.md)). Repositories decrypt; `ApiClient` only moves bytes and structural DTOs. It is the web counterpart of the Android Retrofit client ([android 05](https://github.com/Nyxite/android)).

## 5.1 Conventions

- **Base path:** `https://{host}/api/v1`, where `{host}` is resolved from the **per-account instance config** (`config.json` / per-account host, [01 §1.8](01-architecture.md), [14](14-authentication.md)); default `nyxite.app`. The OIDC authority, API base, relay hub, and public-share base are all runtime-resolved.
- **Auth:** `Authorization: Bearer <OIDC access token>` on `/api/v1/**` except public share endpoints. Guests use a short-lived **share-session token** ([14](14-authentication.md)). Lifetimes match the server: access ~5 min; guest share-session 15 min (renewable); relay socket ticket single-use, 60 s.
- **Content type:** `application/json` for structure/metadata; raw binary (`Uint8Array` / `ReadableStream`) for ciphertext blobs (`GET/PUT .../blob`, snapshots, version blobs).
- **IDs:** UUIDv7, client-allocated where the API requires (`crdtDocId`, file `id` at create, share/key surrogate ids) via `@/lib/uuidv7`. **Timestamps:** RFC 3339 UTC strings.
- **Pagination:** `?cursor=<opaque>&limit=<n>` (default 50, max 200); response `{ items, nextCursor }`. The cursor is an **opaque base64url** server token — stored verbatim (`syncCursor.cursor`) and **never parsed** ([04 §4.3](04-local-data-model.md)).
- **Idempotency:** `Idempotency-Key` (a UUIDv7) is **REQUIRED on POST creates**, taken from the originating `outbox` row so retries reuse it; (user, endpoint)-scoped for 24 h server-side. Replay → original response; same key + different body → `409 idempotency_conflict`.
- **bytea encoding [P]:** `nameEnc` / `metadataEnc` / `wrappedKey` / `linkTokenHash` and other `bytea` fields are carried inside JSON DTOs as **base64 (standard, not base64url) strings**, encoded/decoded at the `ApiClient` edge via `@/lib/base64url`'s base64 helpers. Raw ciphertext **blobs** are never base64-wrapped — they stream as binary bodies. (This base64-in-JSON choice is web-side; the server treats them as opaque `byte[]`.)

## 5.2 Endpoints

Bodies marked *(cipher)* are opaque binary streams; *(json)* are DTOs whose `*Enc` fields are base64 per [§5.1](#51-conventions). Grouped to match the server resource map ([server 04 §4.2](https://github.com/Nyxite/server)).

| Group | Method · Path | Notes |
|-------|---------------|-------|
| **Structure** | `GET/POST /projects`, `GET/PATCH/DELETE /projects/{id}` *(json)* | `nameEnc`, `metadataEnc` |
| | `GET /projects/{id}/folders` | tree/flat |
| | `POST/GET/PATCH/DELETE /folders`, `/folders/{id}` *(json)* | `nameEnc`, `parentFolderId` (move) |
| | `GET /folders/{id}/files` | list |
| | `POST/GET/PATCH/DELETE /files`, `/files/{id}` *(json)* | `POST` sends client-allocated `id` + `crdtDocId` (text only); `contentType` **immutable** (never PATCHed); `PATCH` = `nameEnc`/`syncPolicy`/`parentFolderId`/`metadataEnc` only |
| **Content** | `GET /files/{id}/blob` *(cipher)* | `If-None-Match: <contentHash>` → 304 |
| | `PUT /files/{id}/blob` *(cipher)* | ink/binary LWW; `If-Match: <seq>` (parent seq, [server 06 §6.5](https://github.com/Nyxite/server)) |
| | `GET /files/{id}/crdt/log?since={seq}` *(cipher)* | encrypted updates — REST fallback for relay |
| | `POST /files/{id}/crdt/log` *(cipher)* | submit encrypted update(s) — REST fallback |
| | `GET /files/{id}/snapshot` *(cipher)* | latest encrypted snapshot pointer/stream |
| **File keys** | `GET /files/{id}/keys` *(json)* | caller's wrapped FK(s) to unwrap locally |
| | `POST /files/{id}/keys` *(json)* | upload wrapped FK(s) for members |
| | `POST /files/{id}/keys/rotate` *(json)* | `{ newKeyId, generation, wrappedKeys[], newHeadRef }`; commit iff `generation == current+1`, else `409 conflict` |
| | `POST /files/keys:batch` *(json)* | subtree share fan-out; idempotent, per-item partial success |
| **Keys/devices/recovery** | `GET /keys/directory?userId=` \| `?email=` *(json)* | public keys to wrap shares |
| | `PUT /keys` *(json)* | publish/rotate own public identity keys |
| | `GET/DELETE /devices`, `/devices/{id}` | list / revoke |
| | `POST /devices` *(json)* | `{ label, pubkey }` → `{ deviceId, status:"pending", pairingCode, qrPayload }` |
| | `POST /devices/{id}/approve` *(json)* | `{ wrappedIdentityKey }` = HPKE-seal to pending device pubkey |
| | `GET /devices/me/enrollment` *(json)* | `{ wrappedIdentityKey }` once approved — **single-use** |
| | `GET/PUT /recovery` *(json)* | server-opaque recovery blob + `kdf` (AES-256-GCM under Argon2id key) |
| **Versions** | `GET /files/{id}/versions` *(json, paged)* | newest first |
| | `GET /files/{id}/versions/{seq}` *(json)* | metadata |
| | `GET /files/{id}/versions/{seq}/blob` *(cipher)* | version ciphertext |
| | `POST /files/{id}/restore` *(json)* | body `{ seq }` **only**; new head via normal write path |
| **Shares** | `GET /shares?targetType=&targetId=` *(json)* | list on target |
| | `POST /shares` *(json)* | account: `wrappedKeys[]`; link: `linkTokenHash` only (key stays in fragment) |
| | `PATCH /shares/{id}` *(json)* | permission / expiry |
| | `DELETE /shares/{id}` | revoke (client rotates key separately) |
| **Sync** | `POST /sync/changes` *(json)* | delta; cursor in/out |
| | `GET /sync/manifest?projectId=` *(json)* | ids, contentType, versions, hashes, policies for reconcile/index |
| **Me** | `GET /me`; `GET/PUT /me/settings` *(json)* | settings body is **ciphertext** (`settingsEnc`) |
| **Public share** (no bearer) | `GET /share/{token}`; `GET /share/{token}/blob` *(cipher)* | guest bootstrap + ciphertext; relay WS `/share/{token}/ws` handled by `RelayClient` ([09](09-realtime-collaboration.md)) |

> The client implements **no `/search`, `/diff`, `/content` (decoded), or `/crdt/state`** calls — none exist. Search and diff are local ([11](11-search.md), [12](12-version-history.md)).

## 5.3 DTO types (TS translation of the server C# records)

All `bytea` → `Uint8Array` in TS, carried as base64 strings over JSON ([§5.1](#51-conventions)). DTOs carry the `Dto` suffix and **never escape `data/`** ([03 §3.4](03-project-structure.md)).

```ts
interface FileDto {
  id: string; projectId: string; folderId?: string; ownerId: string;
  nameEnc: Uint8Array; contentType: ContentType; syncPolicy: 'server-default' | 'excluded';
  currentVersionSeq?: number;            // = file_versions.seq of current head
  crdtDocId?: string; keyGeneration: number;
  createdAt: string; updatedAt: string;  // RFC3339
}

interface CreateFileRequest {
  id: string;                            // client-allocated UUIDv7
  projectId: string; folderId?: string; nameEnc: Uint8Array;
  contentType: ContentType; syncPolicy?: 'server-default' | 'excluded';
  metadataEnc?: Uint8Array;
  crdtDocId?: string;                    // REQUIRED for text types, null otherwise
  ownerWrappedKey: WrappedKey;           // owner's FK wrapped to themselves
}

interface WrappedKey { keyId: string; memberId?: string; ciphertext: Uint8Array; generation: number; }

interface CreateShareRequest {
  targetType: 'file' | 'folder' | 'project'; targetId: string;
  kind: 'user_grant' | 'link';
  granteeId?: string; permission: 'read' | 'write';
  wrappedKeys?: WrappedKey[];            // account share
  linkTokenHash?: Uint8Array;            // link share: hash only; KEY stays in URL fragment
  expiresAt?: string;
}

interface PublicKeyDto { keyId: string; x25519: Uint8Array; ed25519: Uint8Array; generation: number; }

interface RecoveryBlobDto { version: number; kdf: KdfParams; nonce: Uint8Array; ciphertext: Uint8Array; tag: Uint8Array; }
interface KdfParams { alg: 'argon2id'; m: number; t: number; p: number; salt: Uint8Array; }

interface BatchWrapRequest { grants: BatchGrant[]; }
interface BatchGrant { fileId: string; keyId: string; memberId: string; wrappedKey: Uint8Array; }

interface Paged<T> { items: T[]; nextCursor?: string; }  // nextCursor opaque, never parsed
```

(Plus `ProjectDto`, `FolderDto`, `ShareDto`, `DeviceDto`, `VersionDto`, `DeviceEnrollResponse { deviceId; status; pairingCode; qrPayload }`, and `RotateRequest { newKeyId; generation; wrappedKeys; newHeadRef }`, translated the same way.)

## 5.4 Error model → typed `NyxiteApiError`

RFC 9457 `application/problem+json` is parsed into a discriminated-union `NyxiteApiError` (`type NyxiteApiError = { kind: 'KeyGenerationStale'; ... } | ...`) and mapped to a client action:

| HTTP / code | `kind` | Client action |
|---|---|---|
| 400 `validation_failed`, `bad_sync_policy` | `Validation` | Surface; do not blind-retry. |
| 401 `unauthenticated`, `token_expired` | `Unauthenticated` | **Silent refresh** once via `AuthManager`, retry; else re-login ([14](14-authentication.md)). |
| 403 `acl_denied` | `Forbidden` | Surface "no access". |
| 403 `2fa_required` | `TwoFactorRequired` | Route to TOTP step. |
| 404 `not_found` | `NotFound` | Mark missing/removed. |
| 409 `conflict`, `version_conflict` | `Conflict` | LWW conflict flow ([08 §8.5](08-sync-engine.md)). |
| 409 `excluded_content` | `ExcludedRejected` | **Never retry** upload of excluded files. |
| 409 `address_exists` | `AddressExists` | Treat as success (write-once idempotent). |
| 409 `idempotency_conflict` | `IdempotencyConflict` | Same key + different body — surface as bug, do not retry. |
| 410 `share_revoked`, `link_expired` | `ShareDead` | Mark share/link **dead** ([13](13-sharing.md)). |
| 412 `key_generation_stale` | `KeyGenerationStale` | **Refetch keys + rotate/re-encrypt** with current generation, retry ([07](07-key-and-device-management.md)). |
| 413 `payload_too_large` | `TooLarge` | Reject (chunking is Phase 5). |
| 429 `rate_limited` | `RateLimited(retryAfter)` | **Backoff per `Retry-After`**. |
| 5xx `internal`, `unavailable` | `ServerError` | Backoff + retry from outbox. |

## 5.5 Fetch pipeline, retry, cancellation

`ApiClient` wraps `fetch` with a small ordered middleware chain (no heavy HTTP dep):

1. **Auth** — attach bearer/share token; on `401 token_expired` trigger a single `AuthManager` refresh + retry.
2. **Idempotency** — attach `Idempotency-Key` for creates (from the outbox row).
3. **Problem+json** — parse error bodies into `NyxiteApiError`.
4. **Retry/backoff** — exponential backoff + jitter on `429`/`5xx`, honoring `Retry-After`; never retries non-idempotent calls without an idempotency key.
5. **(dev) logging** — via the scrubbing `Logger`: ciphertext sizes and opaque IDs may be logged; **never** plaintext, keys, tokens, or URL fragments ([17](17-security.md)).

- **Cancellation:** every request takes an `AbortSignal`; TanStack Query passes its `signal` so navigation/unmount aborts in-flight fetches. Streamed blob reads are abortable mid-stream.
- **Reliability:** all mutations flow through the **outbox** ([04 §4.3](04-local-data-model.md)) with stable idempotency keys; content-addressed blob writes are idempotent (`address_exists` → success); conditional GET (`If-None-Match: contentHash`) skips unchanged ciphertext.

## 5.6 TanStack Query integration

- Query/mutation functions call **repositories**, not `ApiClient` directly; repositories fetch ciphertext via `ApiClient` then `CryptoEngine.open(...)`, so **query caches hold decrypted view models** for the UI ([01 §1.3](01-architecture.md)).
- **Never persist the Query cache to disk.** No TanStack Query persister is wired to IndexedDB/localStorage — decrypted plaintext lives only in memory and in the gated `LocalStore`/`BlobCache` ([04](04-local-data-model.md), [17](17-security.md)). Offline reads come from `LocalStore`, not a persisted Query cache.
- Query keys are scoped by `accountId` so cache cannot leak across accounts on switch ([01 §1.8](01-architecture.md)).

## 5.7 Validation (zod)

Every decoded JSON DTO is validated with a **zod** schema at the `ApiClient` edge before it becomes a typed DTO ([02 §2.3](02-tech-stack-and-libraries.md)); base64 `bytea` fields decode to `Uint8Array` via a zod transform. Validation failure → `Validation` error (treated as a server/contract bug, surfaced, not retried). The zod schemas are checked against `/openapi/v1.json` in CI ([18](18-build-ci-testing.md)).

## 5.8 What the client enforces locally (server can't)

Because the server validates only structure/ACL/write-once and **cannot read plaintext** ([server 04 §4.7](https://github.com/Nyxite/server)):

- Compute the **BLAKE3 content address honestly** and verify downloaded ciphertext decrypts and matches its claimed address before trusting it.
- Don't rely on the server to validate CRDT updates — malformed updates simply fail to apply locally ([09](09-realtime-collaboration.md)).
- Check the 100 MB inline size limit client-side before upload to avoid `413`.

## 5.9 TLS / Cloudflare

- Standard TLS validation against the public chain (Cloudflare-fronted `nyxite.app`); the browser handles trust. There is no app-level certificate pinning in a browser (the platform owns the trust store).
- **Self-hoster caveat:** an instance hitting its **origin directly** behind Cloudflare may present a **Cloudflare Origin CA** certificate, which browsers do not trust by default — such hosts must terminate with a publicly trusted certificate (or front via Cloudflare) for the web client to connect. Out of scope for normal users ([17](17-security.md)).
