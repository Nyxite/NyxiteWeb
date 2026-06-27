# 11 — Search (Client-Side)

The server holds **no index** — under full E2EE it can't read content, so there is **no server search endpoint** at all ([server 11](https://github.com/Nyxite/server)). Search lives where the keys and plaintext live: the browser, over its **local subset**. Of the three clients the web is the **weakest search surface** because a browser cannot hold the whole corpus ([00 §0.3](00-overview.md)); the **desktop** is the full-corpus reference.

## 11.1 Why search is client-side

Server-side search would require the server to read plaintext, which is incompatible with zero-knowledge. So the index is built **in the browser** from content the client has already decrypted, kept **in IndexedDB**, account-scoped, and **never uploaded** ([server 11 §11.1](https://github.com/Nyxite/server)). There is no query round-trip: a search never leaves the device.

## 11.2 Per-surface scope

| Surface | Search scope |
|---------|--------------|
| **Desktop** | **Full local corpus** — all synced files decrypted and indexed; the primary, complete search experience and the reference for "search everything" |
| **Android** | Local cache subset (client-pinned + recently opened); battery/storage-bounded |
| **Web** | **Whatever this session has decrypted/cached** (kept-in-browser + recently opened); the **weakest** — a browser can't hold the whole corpus |

This weaker reach is the direct, accepted consequence of privacy-first; the spec degrades gracefully rather than weakening encryption.

## 11.3 Server's role (no index)

The server exposes **no search endpoint** (the former `GET /api/v1/search` is removed). It supports client indexing only by serving:
- **Sync manifests** (`GET /sync/manifest`, [08](08-sync-engine.md)) — file ids, content-types, versions, hashes, policies — so the client knows what exists and what to fetch.
- **Ciphertext** (`/files/{id}/blob`, snapshots, CRDT logs) for the client to decrypt and index locally.

There is no `search_index` table, no server FTS ([server 11 §11.3](https://github.com/Nyxite/server)).

## 11.4 Web client index

- **Engine: MiniSearch** (or FlexSearch) over the decrypted local subset, **persisted to IndexedDB** in the account-scoped database ([04](04-local-data-model.md)). The serialized index is loaded into memory on session start and incrementally updated; it is **never uploaded** and is protected exactly like the rest of the at-rest store ([17](17-security.md)).
- **Document shape** — `{ fileId, title, body }` where `body` is the decrypted current-head text for markdown/plaintext. Ink files index **title only** in v1.0.0 (recognition reserved, [10 §10.5](10-editors.md)).

| Indexed | Not indexed |
|---|---|
| **Titles of all known files** (names decrypted client-side, [07](07-key-and-device-management.md)) — so every file is discoverable | Bodies of files this session never decrypted |
| **Bodies of cached / kept-in-browser files** ([16 §16.2](16-offline-and-pwa.md)) | Files evicted from the local subset (title kept, body dropped) |

So **titles are always searchable**; **bodies only for the cached/kept subset**.

## 11.5 Indexing lifecycle

- **On decrypt**: whenever a file's plaintext head is produced (open, sync pull, snapshot, CRDT merge to a quiescent point), upsert its index document (title + body).
- **On keep-in-browser**: prefetching and decrypting kept content adds its body to the index ([08](08-sync-engine.md), [16 §16.2](16-offline-and-pwa.md)).
- **On eviction**: when a not-kept file's plaintext is evicted under storage pressure, **drop its body, keep its title** — it still surfaces as a "title match only — open to search inside" result.
- **On rename / delete / rotate**: refresh or remove the document accordingly (titles re-derived from the decrypted name).
- **Rebuild**: a reindex pass rebuilds the whole index from locally available plaintext on **schema change or corruption**, and re-serializes it to IndexedDB.

## 11.6 Search UX

- A **query box** in the browse screen plus a dedicated search view; debounce input; query runs against the in-memory MiniSearch index (fast, no I/O per keystroke).
- **Results**: title + **snippet** (highlighted body match where a body is indexed) + project/folder breadcrumb + a sync/cache badge. Tapping opens the file (fetching + decrypting on demand if only the title was indexed).
- A **scope hint** is always visible — e.g. *"searching N cached files on this device"* — and an inline action to **keep more in browser** to widen the body-search scope. Clearly distinguish "found in cached content" from "title match only — open to search inside".
- **Graceful degradation**: when the local subset exceeds the browser's Storage-API quota, body indexing is bounded and the UI explains the limit and points to keep-in-browser / desktop ([16](16-offline-and-pwa.md)).

## 11.7 Implications

- **Cross-device consistency is weaker** than a server index: each device searches what it holds, and the web holds the least. **Desktop is the reference** for "search everything".
- **Shared docs** are searchable on a member's device once decrypted there; never searchable server-side.
- A future still-zero-knowledge **encrypted searchable index / blind index** on the server is a possible later hardening, **deferred** from v1.0.0 ([19](19-open-questions.md)).
