# 09 — Real-time Collaboration

Live multi-user editing of text documents over an **encrypted relay**: the server stores and forwards encrypted CRDT updates but never merges or reads them; **each browser runs the Yjs engine and merges locally** ([server 05](https://github.com/Nyxite/server)). Yjs is the **reference** Yrs-family implementation, so the web binding is the lowest-risk of the three clients ([02 §2.7](02-tech-stack-and-libraries.md)) — but the wire protocol is still conformance-locked against ydotnet (desktop) and ykt (Android) ([§9.12](#912-wire-protocol-conformance-critical), [18](18-build-ci-testing.md)).

## 9.1 Model

- **Client-authoritative, server-relayed.** Each browser holds the Yjs document and merges updates locally. The server stores the encrypted update log and broadcasts encrypted updates to the room; convergence is guaranteed by the CRDT, independent of the server.
- **The server cannot read collaboration traffic.** Updates are sealed with the file key ([06](06-cryptography.md)) before they leave the browser. The server sees update **size and ordering**, never content.
- **Applies to text** types (`markdown`, `plaintext`, later `sourcecode`). Ink/binary sync via LWW ([08 §8.5](08-sync-engine.md)), also encrypted.

## 9.2 Components

- **`CrdtEngine`** (Yjs): holds the per-document `Y.Doc`, applies/encodes updates, computes state vectors, serializes snapshots; bound to the CodeMirror 6 editor via **y-codemirror.next** ([10](10-editors.md)).
- **`RelayClient`** (`@microsoft/signalr`): connects to `RelayHub`, joins per-document groups, submits/receives **encrypted** updates and awareness, manages reconnection. Speaks ciphertext only — it holds no reference to `CryptoEngine` ([01 §1.2](01-architecture.md)).
- **`CollabRepository`**: orchestrates join → bootstrap → live loop → snapshot → leave, bridging `CryptoEngine ⇄ CrdtEngine ⇄ RelayClient`.
- **Awareness** via **y-protocols** awareness, encrypted before relay.

## 9.3 Transport

- **SignalR over WebSocket**, a single hub `RelayHub`, with per-document SignalR **groups keyed `file:{fileId}`**. The official `@microsoft/signalr` JS client is used with the **binary (MessagePack) protocol** so `EncryptedUpdate` ciphertext travels as bytes, not base64.
- **Upgrade auth** (one of):
  - **the server's access token (bearer)** for signed-in users ([14](14-authentication.md));
  - a **single-use socket ticket** (60 s lifetime) minted by the API and exchanged on connect (avoids putting the bearer on the WS URL);
  - a **guest share-session token** (15 min, renewable) for link-share guests.
- The token authorizes **relay access only**; the **decryption key never comes from the server** — it is the user's unwrapped FK, or the guest's **fragment key from the URL** (`#k=…`, [13](13-sharing.md)).
- **Fallback**: when the socket is unavailable, use the REST encrypted-update endpoints ([08 §8.4](08-sync-engine.md)).

## 9.4 Hub contract (client side)

Mirror the server `IRelayHub`. All payloads are ciphertext.

```ts
// server → client callbacks
interface RelayCallbacks {
  OnUpdates(fileId: string, updates: EncryptedUpdate[]): void; // everything after the client's cursor (join bootstrap)
  OnUpdate(fileId: string, update: EncryptedUpdate): void;     // a peer's relayed update
  OnAwareness(fileId: string, awarenessCipher: Uint8Array): void; // encrypted ephemeral presence/cursors
  OnPresence(fileId: string, peers: PresenceDto[]): void;      // join/leave roster (identities only)
  OnError(fileId: string, code: string, message: string): void;
}

// client → server (hub methods)
JoinDocument(fileId: string, sinceSeq: number): Promise<void>; // server replies OnUpdates with everything after sinceSeq
SubmitUpdate(fileId: string, update: EncryptedUpdate): Promise<void>; // server persists + relays; never decrypts
SubmitAwareness(fileId: string, awarenessCipher: Uint8Array): Promise<void>; // relayed, not persisted
LeaveDocument(fileId: string): Promise<void>;

type EncryptedUpdate = { seq: number; ciphertext: Uint8Array; keyId: string };
```

`PresenceDto` carries identity only (the account display name or `"guest"`), never content.

## 9.5 Join handshake

1. Open the (authenticated) socket ([§9.3](#93-transport)); call `JoinDocument(fileId, localSinceSeq)`.
2. Server checks **ACL** — **read** to receive, **write** to `SubmitUpdate` (guests' permission comes from the link share).
3. Server returns, via `OnUpdates`, all encrypted updates with `seq > sinceSeq`, and **points the client at the latest encrypted snapshot blob** to bootstrap from. The server computes **no** state-vector diff (it has no readable document).
4. Client bootstraps the `Y.Doc`: `CryptoEngine.open` the snapshot and apply it, `open` + apply each update, reconcile via the **locally reconstructed Yjs state vector**.
5. Client is added to the `file:{fileId}` group; `OnPresence` delivers the roster.

## 9.6 Update flow

1. **Local edit** → Yjs emits an update → `CryptoEngine.seal(update, fk, { kind: crdt, fileId })` → `SubmitUpdate(fileId, { seq: 0, ciphertext, keyId })`.
2. **Server**: authorize (write) → assign monotonic `seq` → **persist** the ciphertext → broadcast `OnUpdate` to the rest of the group. **No decryption, no merge, no content validation.**
3. **Inbound** `OnUpdate`/`OnUpdates` → `CryptoEngine.open` → `Y.applyUpdate` → editor text recomputed via the y-codemirror binding. Merge is deterministic and order-independent.
4. **Outbox**: locally generated updates are persisted in IndexedDB ([04](04-local-data-model.md), [08 §8.8](08-sync-engine.md)) until acked, so edits survive disconnects and are resubmitted on reconnect or via REST fallback.

## 9.7 Awareness & presence

- **Awareness** (cursor, selection, display label/color) is produced by the y-protocols awareness protocol, **sealed** with the FK (`kind = awareness`), relayed via `SubmitAwareness`/`OnAwareness`, and **never persisted**. Rendered as remote carets/selections in the editor ([10](10-editors.md)).
- **Presence roster** (`OnPresence`) shows who is in the room by account display name or `"guest"` — no content. Rendered as avatar chips.
- Throttle awareness emission (coalesce cursor moves to ~20–30 Hz max) to limit relay traffic.

## 9.8 Snapshotting & compaction (client-driven)

The server cannot compact an encrypted log it can't read, so **the client snapshots**: serialize the merged `Y.Doc` (`Y.encodeStateAsUpdate`), `seal` it with the FK (`kind = snapshot`), and upload it as a **content-addressed blob + a `file_versions` row** ([12](12-version-history.md)). This is what produces version history for text.

**Snapshot triggers (PINNED, client-side):**

- **≥ 200 CRDT updates** since the last snapshot, **OR**
- **≥ 5 min** of accumulated edits, **OR**
- the **last participant leaving** the room.

Any participating client may snapshot. The **server prune watermark** (`PruneAfterSnapshotSeq`, with its safety tail) is entirely server-side ([server 05 §5.6](https://github.com/Nyxite/server)); the client only snapshots and uploads. In a browser, snapshot serialization/seal runs in the crypto worker ([01 §1.6](01-architecture.md)) to avoid blocking the editor.

## 9.9 Guest sessions (this browser joining a link share)

- **Entry**: the app opens `…/share/{token}#k=<fragmentKey>` ([13 §13.3](13-sharing.md)). The **fragment key is parsed locally and never sent** to the server; it is captured on first paint and held in memory only ([01 §1.7](01-architecture.md), [17](17-security.md)).
- `GET /share/{token}` bootstraps; the relay socket connects at **`/share/{token}/ws`** with a short-lived **guest share-session token** (15 min, renewable) authorizing relay access only.
- The FK comes from the fragment; the guest decrypts/edits exactly as a member would.
- **Read-only links**: the guest receives `OnUpdate`/`OnAwareness` but is rejected via `OnError` on `SubmitUpdate`; the editor enters view-only mode.
- **Guest-authored** updates/snapshots store `author_id = null` server-side.
- A guest runs **without the per-account store**: content lives in an **ephemeral in-memory cache**, never written to an account IndexedDB database, and discarded when the guest session ends ([14 §14.5](14-authentication.md)).

## 9.10 Reconnection, lifecycle & multi-tab

- The relay connection **lives in the open tab**. On tab close or hide (`visibilitychange` → hidden / `pagehide`) call `LeaveDocument(fileId)` and tear down; on return, rejoin.
- **Reconnection** uses exponential backoff with jitter; on reconnect, re-`JoinDocument` with the current `sinceSeq` and **drain the outbox**, then resume the live loop. Connection state is surfaced in the editor (live / reconnecting / offline-editing).
- **Multi-tab** (ratified, [19 §19.8](19-open-questions.md)): avoid duplicate sockets by electing a **single relay connection per account** via the **Web Locks API** with **BroadcastChannel** for coordination/fan-out — the holder owns the socket and rebroadcasts inbound updates/awareness/presence to peer tabs, which apply them to their local `Y.Doc`; submits from peer tabs are funneled to the holder. This mirrors the syncing-tab election in [08 §8.8](08-sync-engine.md). If the holder closes, a peer re-elects and re-joins from its `sinceSeq`.
- **Background-tab throttling caveat**: hidden tabs are timer-throttled by the browser, so a backgrounded holder may stall awareness/heartbeats; the election prefers a **visible** tab, and a hidden holder yields the lock when a visible tab is available.
- **Key rotation**: an `OnError` with `key_generation_stale` (or `412` on REST) → refetch the wrapped key for the new generation, re-seal pending updates under the new FK, and resubmit (idempotent) ([07 §7.6](07-key-and-device-management.md), [08 §8.9](08-sync-engine.md)).

## 9.11 Ordering & durability

- The server assigns a **monotonic `seq` per doc** for the encrypted log; CRDT convergence does not depend on order, but `seq` gives the relay a durable cursor and snapshot watermark.
- A write is **acked after it is persisted** (durable) — even though the server can't read it, it won't lose it. The outbox holds each update until its ack.

## 9.12 Wire-protocol conformance (critical)

Text edited on web (Yjs) must merge identically on desktop (ydotnet) and Android (ykt). The client ships a **`CrdtConformance` suite** that replays the shared Yrs wire-protocol vectors and asserts identical merged state and identical encoded updates across bindings ([18 §18.5](18-build-ci-testing.md)). Because the relay model keeps no CRDT engine in the live server path, binding-maturity risk lives entirely in the clients; Yjs is the **reference**, so the web vectors are authoritative for the protocol the other clients are checked against.
