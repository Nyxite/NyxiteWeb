# 08 — Sync Engine

Reconciles this browser's local state ([04](04-local-data-model.md)) with the server for every readable file, honoring per-file policy and the right model per content type — **all over ciphertext** ([server 06](https://github.com/Nyxite/server)). The server moves bytes, sequences encrypted updates, and tracks versions/policies; the browser does all crypto and merge. The web client is the **most constrained peer** ([00 §0.3](00-overview.md)): it holds structure/manifests for everything readable but caches and indexes only its **local subset**, so the sync engine is built around *lazy, on-demand* fetch with an opt-in proactive tier.

The engine is the `SyncRepository` implementation in the data layer, orchestrating `LocalStore`, `ApiClient`, `RelayClient`, `CrdtEngine`, `CryptoEngine` and `BlobCache` ([01 §1.5](01-architecture.md)). Plaintext never reaches `ApiClient`; every outbound content payload passes `CryptoEngine.seal(...)`, every inbound one `CryptoEngine.open(...)` ([01 §1.2](01-architecture.md)).

## 8.1 Content-type → sync model

| `contentType` | Phase | Sync model | Channel |
|---|---|---|---|
| `markdown`, `plaintext`, `sourcecode` | 1 / 5 | **CRDT** (client-merged, Yjs) | Encrypted relay ([09](09-realtime-collaboration.md)) + REST `crdt/log` fallback |
| `ink` | 3 | **LWW / version-vector** | `PUT/GET /files/{id}/blob` (encrypted blob) |
| `office`, `image`, `binary` | 5 | **LWW / version-vector** | encrypted blob, chunked for large |

`contentType` is immutable after creation and selects the decoder/editor ([10](10-editors.md)) and the sync path. CRDT-vs-LWW assignment is pinned to the server ([server 06 §6.1](https://github.com/Nyxite/server)).

## 8.2 Sync policies

The server `syncPolicy` enum is **only `{ server-default, excluded }`** — `pinned-local` does not exist on the server. Offline pinning is the separate client-local `keepInBrowser` field ([§8.7](#87-on-demand-download--caching), [16 §16.2](16-offline-and-pwa.md)).

| Policy | Server stores? | Client fetches? | Keeps plaintext in browser? |
|--------|---|---|---|
| `server-default` | yes (ciphertext) | on-demand (open) or proactively if kept | depends on `keepInBrowser` |
| `excluded` | **no** | never | browser-only (never uploaded) |

Transitions: `excluded → server-default` triggers an initial ciphertext upload from this browser; `server-default → excluded` stops syncing, drops local plaintext from `LocalStore`/`BlobCache`, removes it from the search index, and the server schedules content purge. Uploading content for an `excluded` file is rejected (`409 excluded_content`) — **never retried** ([§8.9](#89-server-error--state-mapping), [05](05-api-client.md)).

**`keepInBrowser` is never sent to the server.** It is a client-local, per-browser-profile control (`keep`/`dontKeep`/`inherit`, set at file/folder/project with cascade) over which `server-default` files are proactively downloaded, decrypted, and indexed. A kept file is just a `server-default` file this browser chooses to prefetch; a not-kept one is fetched on open. `excluded` remains the distinct "never upload at all" choice. Given browser storage limits, the default keep tier is small and quota-aware ([16 §16.5](16-offline-and-pwa.md)).

## 8.3 Two-tier reconciliation

### Manifest reconcile (periodic / on project open)
`GET /sync/manifest?projectId=` returns per-file structural/opaque records:

| Field | Use locally |
|---|---|
| `fileId` | key into `LocalStore` |
| `contentType` | selects sync model + editor |
| `syncPolicy` | `server-default` \| `excluded` |
| `currentVersionSeq` | head version watermark for pull/up-to-date check |
| `contentHash` | opaque; conditional-fetch tag (`If-None-Match`) |
| `keyGeneration` | drives FK rotation refetch ([07 §7.6](07-key-and-device-management.md)) |
| `updatedAt` | freshness / ordering |
| `deletedAt` | soft-delete propagation |

No names, no content. The client diffs each record against `LocalStore` to decide **pull / push / delete** and to drive the **local search index** ([11](11-search.md)). Manifest reconcile is the authority for "what exists"; it runs on project open and opportunistically (visibility/focus, [§8.8](#88-execution-model-p)).

### Delta sync (incremental, cursor-based)
`POST /sync/changes` with `{ since: cursor, projectId, localChanges[] }` → `{ serverChanges[], nextCursor }`. Change `kind ∈ structure | blob | crdt | delete | keyrotate`; each carries a `ref` (structure → encrypted-name/metadata ref; blob → `blob_ref` to pull; crdt → log cursor ref; keyrotate → new `keyGeneration`).

The opaque `nextCursor` (base64url, monotonic, tied to the server change-log seq) is persisted **verbatim** in `LocalStore` and **never parsed by the client**; it is **resumable** across interruptions and tab closes. The client uploads its own pending local changes from the **outbox** ([§8.8](#88-execution-model-p)) in the same call and applies server changes:

- `structure` → upsert project/folder/file metadata (decrypt `nameEnc`).
- `blob` → mark file `PendingPull`; fetch ciphertext on open or proactively (kept).
- `crdt` → fetch encrypted updates after the local cursor (or rely on the live relay).
- `delete` → propagate `deletedAt`.
- `keyrotate` → refetch the wrapped key for the new generation ([07 §7.6](07-key-and-device-management.md)).

## 8.4 Text sync (CRDT)

- **Live**: via the relay ([09](09-realtime-collaboration.md)).
- **Offline / catch-up**:
  1. `GET /files/{id}/crdt/log?since=<localSeq>` → encrypted updates.
  2. Fetch the **latest encrypted snapshot** (bootstrap) and decrypt it.
  3. `CryptoEngine.open` the snapshot + updates; reconstruct the local Yjs **state vector**; merge (order-independent).
  4. Submit local encrypted updates from the outbox via `POST /files/{id}/crdt/log` when the relay is unavailable.
- Convergence is guaranteed by the CRDT regardless of server ordering; the server's `seq` is only a durable cursor / snapshot watermark — the server computes **no** state-vector diff (it can't read the doc).
- **Reconciliation under rotation (offline catch-up).** Each outbound CRDT update row records the `keyId` it was sealed under. On reconnect the outbox is submitted in order; the server assigns a `seq` and acks each, matched back to the outbox row by the update's **stable client id**, then marked acked with the stored `seq`. Updates sealed under an **old** `keyId` after a rotation are rejected `412 key_generation_stale`; the engine refetches the wrapped key, re-seals under the new FK, and resubmits. Resubmission is **idempotent via the stable client id**, so a re-seal never double-applies.

## 8.5 Ink/binary sync (LWW / version-vector)

- **Version-vector shape**: the client keeps a per-file `{ deviceId → counter }` map inside the **encrypted metadata** (`metadata_enc`); the server cannot read it. **Increment rule**: on each local committed ink/binary edit, increment this browser's own component by 1. **Comparison** (component-wise over the union of ids, missing = 0): `A == B` if all equal; `A` is an **ancestor** of `B` if every component ≤ and at least one strictly less (descendant is the converse); otherwise the two are **concurrent** (a real conflict).
- **Upload**: the client sends the parent version's `seq` as **`If-Match: <parentSeq>`** (or body `parentSeq`) on `PUT /files/{id}/blob` (ciphertext). 
  - head `== parentSeq` → accept the ciphertext as a new **content-addressed** blob, advance head, record a `file_versions` row.
  - head `!= parentSeq` → **`409 conflict`** returning the **winning** version metadata; the rejected bytes are **retained as a sibling (losing) version row**, not head, and surfaced in history (no data loss).
- **True-concurrent tie-break = last-write-wins by server-received timestamp.** The server detects conflicts purely on head `seq`; the version-vector in encrypted metadata is the **client's** classifier (equal / ancestor / concurrent), not the server's.
- **Conflict reconcile**: on `409` the client records both versions (winner head, loser sibling), classifies via the version-vector, sets the file `Conflicted`, and surfaces a non-destructive "two versions exist" affordance in history ([12](12-version-history.md)). Ink has no automatic merge — the user keeps/merges manually.
- **Download**: metadata eagerly via manifest; ciphertext on-demand (`server-default`, not kept) or proactively (kept). Use `If-None-Match: <contentHash>` to skip unchanged blobs.

## 8.6 Deletes

Soft-delete via `deletedAt`, propagated through manifest/delta; all peers converge. `excluded` files never reached the server, so deleting them is local-only. Hard delete/purge is admin-only and not initiated from the normal client.

## 8.7 On-demand download & caching

- The browser holds **structure/manifests for all readable files** but fetches ciphertext **lazily**.
- Opening a file fetches its ciphertext, decrypts in the crypto worker, verifies the **BLAKE3 content address** before trusting bytes ([06 §6.6](06-cryptography.md)), and renders.
- `keepInBrowser` files are fetched **proactively**, decrypted, kept in `BlobCache`/`LocalStore`, and **indexed** for offline search ([11](11-search.md)). This is the only tier guaranteed available offline.
- **Conditional fetch** with `If-None-Match: <contentHash>` avoids re-downloading unchanged ciphertext; a `304` keeps the cached copy.
- `BlobCache` eviction respects Storage-API quota; evicting a not-kept file drops it from cache and index but leaves its manifest record ([16 §16.5](16-offline-and-pwa.md)).

## 8.8 Execution model **[P]**

There is **no WorkManager** in a browser. Sync runs in JavaScript in an open tab and is driven by web lifecycle signals:

| Mechanism | Trigger | Use |
|---|---|---|
| **Foreground loop** | tab open + visible (Page Visibility, focus) | manifest + delta reconcile, drain outbox, proactive prefetch of kept files |
| **`online` / `visibilitychange` / `focus`** | reconnect / tab returns to foreground | opportunistic reconcile + outbox flush |
| **Background Sync API** | browser fires registered sync event after reconnect | flush the outbox even if no tab is in foreground (where supported) |
| **Periodic Background Sync** | UA-scheduled, install-gated | best-effort periodic reconcile (Chromium only; optional) |

Where the Background Sync APIs are unavailable (Firefox, Safari) the engine falls back to **purely opportunistic** sync on visibility/focus/`online`. The **outbox** — queued local changes awaiting push — lives in IndexedDB ([04](04-local-data-model.md)) so it survives reloads and tab closes; the service worker ([16](16-offline-and-pwa.md)) holds keys/plaintext, so Background Sync only **triggers** a flush of already-sealed ciphertext records (the SW does not decrypt).

**Multi-tab coordination [P].** Multiple tabs of the same account must not double-sync. One tab is **elected the syncing tab** via the **Web Locks API** (`navigator.locks.request('nyxite-sync:{accountId}')`) with **BroadcastChannel** as the coordination/heartbeat fallback. The holder runs reconcile and outbox drain; other tabs observe results through `LocalStore` change events and re-elect if the holder closes. This pairs with the single-relay-connection election in [09 §9.10](09-realtime-collaboration.md).

Long reconciles use cooperative chunking on the event loop; heavy decrypt/merge offloads to the crypto worker ([01 §1.6](01-architecture.md)). All network retries are exponential backoff with jitter and honor `Retry-After`.

## 8.9 Per-file sync state machine

Canonical state set, identical to [01 §1.7](01-architecture.md) and [04 §4.4](04-local-data-model.md):

`Synced`, `PendingPush`, `PendingPull`, `Downloading`, `Uploading`, `Conflicted`, `Rotating`, `Excluded`, `Error(code)`.

Typical flow `Synced → PendingPush/PendingPull → Uploading/Downloading → Synced`, with `Conflicted` (LWW), `Rotating`, `Excluded`, and `Error(code)` as branch/terminal states. Persisted on the file record and surfaced as compact badges; clicking a badge explains the state and offers actions (retry, view conflict, etc.).

### Server `problem+json` → transition mapping

| Code / status | Transition | Behavior |
|---|---|---|
| `key_generation_stale` (`412`) | → `Rotating` | refetch wrapped key for new generation, re-seal, resubmit (idempotent); then `Synced` |
| `excluded_content` (`409`) | → `Excluded` | **never retry**; drop pending upload |
| `version_conflict` / `409 conflict` (blob) | → `Conflicted` | record winner head + losing sibling, classify via version-vector, surface in history |
| `share_revoked` / `link_expired` | → `Error(code)` | mark dead; prompt re-auth/refetch where applicable |
| `429` | stays, backoff | exponential backoff honoring `Retry-After`, then resume |
| other 5xx/network | → `Error(code)` transient | retry with backoff |

These map through the typed `NyxiteApiError` ([05 §5.4](05-api-client.md)) and are conformance-locked ([18](18-build-ci-testing.md)).

## 8.10 Conflict philosophy

- **Text**: **no conflicts** — CRDT merge happens on clients; the relay only orders/persists ([09](09-realtime-collaboration.md)).
- **Ink/binary**: **LWW head**, but **history retains every version**, so an LWW "loss" is fully recoverable from version history ([12](12-version-history.md)) — the full-history guarantee holds under E2EE.

## 8.11 Integrity & trust

After each download, decrypt and **verify the BLAKE3 address** before trusting bytes ([06 §6.6](06-cryptography.md)). The client never trusts the server to have validated content; only structure, ACL, and write-once ordering are server-guaranteed.
