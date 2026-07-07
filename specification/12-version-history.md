# 12 — Version History

A full history of **encrypted, content-addressed snapshots** the server stores but cannot read. As with every other capability, **diffs and restore run in the browser** because the server can't read either version ([server 10](https://github.com/Nyxite/NyxiteServer)). Diffing large decrypted snapshots is CPU-heavy, so it runs in a worker with bounded memory ([§12.7](#127-web-specifics)).

## 12.1 Model

- Every file has an **append-only history** of immutable **encrypted**, content-addressed snapshots; the head is `files.currentVersionSeq`.
- History is **full** (retention/purge is a server-side admin/GC action, not a client concern). **All files** get history — E2EE is universal, there is no non-private class to exclude.

## 12.2 Listing & fetching

| Call | Returns |
|------|---------|
| `GET /files/{id}/versions` (paginated, **newest-first**) | version metadata: `seq`, `contentHash` (BLAKE3, opaque to server), `blobRef`, `sizeCipher`, `keyId`, `authorId?` (null = guest), `createdAt` |
| `GET /files/{id}/versions/{seq}` | a single version's metadata |
| `GET /files/{id}/versions/{seq}/blob` | the framed **ciphertext** snapshot |

- Cache the list as `FileVersionEntity` rows ([04](04-local-data-model.md)). The history screen shows a timeline: timestamp, author (`authorId` → display name; "guest" for null), and a size hint. No plaintext size or server summary exists.
- To open a version: unwrap the FK for that version's `keyId`/generation ([07](07-key-and-device-management.md)), `open` the blob, and **verify the BLAKE3 content address** before use ([06](06-cryptography.md)). For text a version is an encrypted **Yjs snapshot** loaded into a transient Yjs doc to render; for ink/binary it is the encrypted blob at that point.

## 12.3 How versions are produced

| Content | Snapshot trigger | Produced by |
|---------|------------------|-------------|
| Text (CRDT) | **Client compaction** of the encrypted CRDT update log into an encrypted snapshot — triggered at **≥200 updates / 5 min / last-leave** ([09](09-realtime-collaboration.md)) | Client (server can't read the log) |
| Ink / binary | **Every accepted `PUT /files/{id}/blob`** (LWW losers retained, [10 §10.4](10-editors.md)) | Client uploads ciphertext |
| Restore | Restore writes a new head ([§12.5](#125-restore-non-destructive)) | Client |

The server records `contentHash`, `blobRef`, `sizeCipher`, `keyId`, `authorId` only — no plaintext size, no diff summary.

## 12.4 Diffs (client-side)

- **There is no server diff endpoint.** The client fetches two encrypted snapshots, decrypts them locally, and computes the diff in a **worker** ([01 §1.6](01-architecture.md)).
- **Text**: render both snapshots to text and compute a line/word diff via **`diff`** (jsdiff); optionally a **Yjs structural** diff for richer results. Present additions/deletions inline or side-by-side; virtualize large diffs.
- **Ink / binary**: a **metadata-level** diff (strokes/pages added/removed, size/time) rather than a pixel diff; render thumbnails of both for comparison.

## 12.5 Restore (non-destructive)

`POST /files/{id}/restore` with body **`{ seq }`** only — **no ciphertext upload in the restore call**. The new head arrives via the **normal write path**, and prior versions are untouched.

- **Text**: the client loads the restored snapshot, **diffs it against the current Yjs doc text and applies the minimal insert/delete ops as a single Yjs transaction** so collaborators converge; the resulting CRDT update is encrypted and **submitted via the relay**, then the new head is **snapshotted** ([09](09-realtime-collaboration.md)). Best-effort textual, not structural, convergence.
- **Ink / binary**: the client uploads the restored bytes as a new ciphertext head via **`PUT /files/{id}/blob`** = new version.
- The restore endpoint **records the restore** (audit), linking the new head to the source `seq`. UX confirms before restoring and explains it creates a **new** version (nothing is overwritten).

## 12.6 Deduplication & LWW recovery

- Snapshots are content-addressed by **plaintext BLAKE3**; re-saving unchanged content within a file resolves to the **same address** (a `file_versions` row is still recorded for the timeline). **Intra-file** dedup only — per-file keys preclude cross-file dedup, and it isn't a goal.
- **Guarantees**: **no destructive overwrite** (LWW losers retained as versions), **encrypted history** (a server-side backup without a member's key yields nothing), and history for **all files**. The history screen is therefore also where a user recovers a "lost" concurrent ink/binary edit: flag concurrent versions and offer "restore this one" / "keep both".

## 12.7 Web specifics

- **Run diffs in a worker.** Decrypting and diffing two large snapshots on the main thread would jank the UI; the diff (and the snapshot decrypt that feeds it) runs in the crypto/diff worker ([01 §1.6](01-architecture.md)).
- **Stream and bound memory.** Fetch snapshot ciphertext as a stream, decrypt in chunks, and cap the diff working set; for very large blobs fall back to a metadata-level summary rather than a full text diff.
- **Verify the content address** on every fetched snapshot before rendering or diffing — recompute BLAKE3 of the decrypted plaintext and reject on mismatch ([06](06-cryptography.md)).
- Fetched version blobs land in the **evictable BlobCache**, not kept-in-browser storage ([16](16-offline-and-pwa.md)); the lightweight version-list metadata is cached in IndexedDB.
