# 19 — Open Questions & Resolutions (Web)

Every web-specific open item is now **ratified** (this revision). Each entry below records the **decision**, **how to validate it (spike)**, and the **pre-planned fallback** if the spike fails — so implementation can proceed while the validations run. Items that are genuinely the **server's** to decide remain "track & confirm" ([§19.6](#196-server-owned-protocol-items-track--confirm)). Project-wide decisions stay canonical in the master [`docs/OPEN-DECISIONS.md`](https://github.com/Nyxite/Nyxite); this file folds into that tracker's Web section. Where an item parallels the Android client it stays aligned ([android 19](https://github.com/Nyxite/android)).

## 19.0 Decisions ratified

| Item | Decision | Where applied |
|------|----------|---------------|
| Rendering / deployment | **Static-export SPA** — Next.js 15 App Router, `output: 'export'`, CDN/static host, no server runtime | [00 §0.5](00-overview.md), [01 §1.1](01-architecture.md), [18 §18.1](18-build-ci-testing.md) |
| Offline model | **Full installable PWA** — Workbox SW + manifest + `navigator.storage.persist()`; **IndexedDB (Dexie)** local subset + **Cache Storage** blobs | [16 §16.1](16-offline-and-pwa.md), [04](04-local-data-model.md) |
| Text CRDT engine | **Yjs** (+ y-protocols, y-codemirror.next) — reference Yrs impl, lowest-risk of the three clients | [02 §2.7](02-tech-stack-and-libraries.md), [09 §9.12](09-realtime-collaboration.md) |
| Ink in the browser | **View + basic Pointer-Events editing**; shared **deterministic CBOR** format | [00 §0.3](00-overview.md), [10](10-editors.md), [§19.7](#197-ink-parity-in-the-browser--decided-view-first) |
| Multi-account | **Yes, from v1.0.0** — per-account IndexedDB/keys/cache/index; per-account instance host | [01 §1.8](01-architecture.md), [14](14-authentication.md) |
| Library stack | shadcn/Tailwind, TanStack Query + Zustand, WebCrypto + hpke-js + libsodium + hash-wasm, CodeMirror 6, @microsoft/signalr, oidc-client-ts, MiniSearch, cbor-x | [02](02-tech-stack-and-libraries.md) |
| **Key storage & lock** ([§19.1](#191-in-browser-key-storage--lock-model--decided)) | **Non-extractable WebCrypto vault key** wraps the identity bundle; unlock via **WebAuthn passkey where available, else passphrase**; idle auto-lock; **persistent ("remember this browser") by default** (session-only is a user choice) | [07 §7.3–7.4](07-key-and-device-management.md), [17](17-security.md) |
| **X25519 / Ed25519 library** ([§19.3](#193-x25519--ed25519-library--decided-libsodium)) | **libsodium-wrappers-sumo (WASM)** — consistent across all browsers; behind `CryptoEngine` for a future native swap | [02 §2.6](02-tech-stack-and-libraries.md), [06 §6.2](06-cryptography.md) |
| **Argon2id params (default)** ([§19.2](#192-webcrypto--wasm-performance--decided-worker-offload)) | **m = 64 MiB, t = 3, p = 1** (matches server/Android; per-blob params in `kdf_params`) | [06 §6.8](06-cryptography.md), [07 §7.6](07-key-and-device-management.md) |
| **Crypto execution** ([§19.2](#192-webcrypto--wasm-performance--decided-worker-offload)) | **All crypto offloaded to a Web Worker** (WebCrypto AES-GCM + WASM) | [01 §1.6](01-architecture.md), [06 §6.9](06-cryptography.md) |
| **Storage persistence** ([§19.9](#199-storage-persistence--eviction--decided-persist--re-derivable)) | **`navigator.storage.persist()` + treat keys/subset as re-derivable**; encourage PWA install | [16 §16.5](16-offline-and-pwa.md), [17](17-security.md) |
| **Client-side search scope** ([§19.5](#195-client-side-search-scope--decided-local-subset-index)) | **MiniSearch over the local subset**, titles for all known files, **body index capped → title-only fallback** | [11](11-search.md) |
| **Multi-tab coordination** ([§19.8](#198-multi-tab-coordination--decided-single-leader-tab)) | **Single leader tab via Web Locks + BroadcastChannel** (one relay socket; followers mirror) | [01 §1.8](01-architecture.md), [09 §9.10](09-realtime-collaboration.md) |
| **Guest local data** ([§19.4](#194-guest-session--data-model--decided-in-memory-only)) | **In-memory only** — guest fragment key + content never persisted; cleared on tab close | [13 §13.4](13-sharing.md), [17](17-security.md) |
| **Static-export routing** ([§19.10](#1910-static-export-routing--runtime-config--decided-client-catch-all--spa-fallback)) | **Client catch-all (`/share/[[...token]]`) + host SPA-fallback rewrite**; runtime `config.json` | [15 §15.1](15-ui-and-navigation.md), [18 §18.2](18-build-ci-testing.md) |

The remaining work below is **validation spikes**, not undecided forks.

---

## 19.1 In-browser key storage & lock model — *DECIDED*

**Decision.** The browser has no platform keystore, so:

- **Vault key**: a **non-extractable WebCrypto `CryptoKey`** (AES-256-GCM); the handle itself is stored in IndexedDB (structured-clone). Its bits can't be read back as bytes — the nearest browser analogue to a keystore wrapping key.
- **Identity bundle** (X25519 + Ed25519 private keys, FK cache) is **AES-256-GCM-wrapped by the vault key** and stored as ciphertext in IndexedDB. Unwrapped material lives only in the in-memory account `UserSession` ([01 §1.8](01-architecture.md)).
- **Unlock**: **WebAuthn/passkey** (PRF / `hmac-secret` extension to derive the unlock secret) **where available**, else **passphrase → Argon2id**; both gate the vault key.
- **Persistence**: **persistent ("remember this browser") by default**; **session-only** is an explicit user choice (vault key dropped on tab close). **Idle auto-lock** (configurable, default ~10 min) evicts the in-memory session.

**Validate.** Spike across **Chromium/Firefox/Safari**: non-extractable-key persistence in IndexedDB, WebAuthn PRF availability/coverage, structured-clone of `CryptoKey`, private-mode and storage-eviction behaviour.

**Fallback.** Where non-extractable-handle persistence or WebAuthn PRF is unavailable, that browser runs **session-only, in-memory** (no at-rest key; re-unlock/re-enroll each session) rather than weakening at-rest protection.

---

## 19.2 WebCrypto & WASM performance — *DECIDED (worker offload)*

**Decision.** All crypto runs in the **crypto Web Worker** ([01 §1.6](01-architecture.md)): WebCrypto `AES-GCM` (hardware-accelerated) for bulk seal/open; **hash-wasm** BLAKE3 (streamed) + Argon2id; **hpke-js** wrap/unwrap; **libsodium** X25519/Ed25519. Transfer `ArrayBuffer`s to avoid copies. **Argon2id default m = 64 MiB, t = 3, p = 1**, off-thread with a progress indicator; chosen params persisted in `recovery_blobs.kdf_params` so any browser can reproduce on recovery.

**Validate.** Perf spike: seal/open throughput on large docs/blobs, BLAKE3 over large binaries, Argon2id wall-time on low/mid/high machines and mobile browsers; confirm 64 MiB does not OOM mobile Safari.

**Fallback.** If 64 MiB pressures the weakest target, drop to **m = 32 MiB, t = 4** (≈constant cost); chunk large-blob hashing/sealing cooperatively if worker latency spikes.

---

## 19.3 X25519 / Ed25519 library — *DECIDED (libsodium)*

**Decision.** Use **libsodium-wrappers-sumo (WASM)** for X25519 and Ed25519 — consistent, audited behaviour on every target and a clean building block under HPKE ([02 §2.6](02-tech-stack-and-libraries.md)). Kept behind `CryptoEngine` so a future native path can replace it without touching repositories.

**Validate.** Confirm **hpke-js suite IDs match the server exactly** (DHKEM(X25519,HKDF-SHA256)/HKDF-SHA256/AES-256-GCM = `0x0020`/`0x0001`/`0x0002`) and that libsodium X25519/Ed25519 outputs interop with server/desktop/Android via the shared vectors ([18 §18.6](18-build-ci-testing.md)).

**Fallback.** Native WebCrypto Ed25519/X25519 on engines that fully support it (feature-detected), gated behind passing the same conformance vectors.

---

## 19.4 Guest session & data model — *DECIDED (in-memory only)*

**Decision.** Anonymous guests persist **nothing**. The fragment key (`#k=…`) and decrypted content live **in memory only**; on first paint of a `/share/...` route the app **captures `location.hash` then `history.replaceState`s** to strip it from the visible URL/history ([13 §13.4](13-sharing.md), [17](17-security.md)). Relay/REST access uses a **short-lived guest share-session token** (15 min renewable, [§19.6](#196-server-owned-protocol-items-track--confirm)) scoped to the shared object only. Closing the tab clears everything — strongest privacy for links opened on shared devices.

**Validate.** Leakage audit: the fragment never appears in any network request, `Referer`, `document.referrer`, history entry, SW cache, IndexedDB, or scrubbed logs; e2e guest test asserts this ([18 §18.5](18-build-ci-testing.md)).

**Fallback.** If `replaceState` is constrained, navigate to a fragment-stripped URL after in-memory capture; expire the in-memory key on idle.

---

## 19.5 Client-side search scope — *DECIDED (local-subset index)*

**Decision.** A **MiniSearch** index over the **local subset** (keep-in-browser + recently opened), persisted to IndexedDB and **never uploaded** ([11](11-search.md)). Index **titles for all known files** + **bodies for cached files only**; the body index is **capped**, and over quota it **degrades to title-only** (titles stay discoverable; re-fetch to read). Desktop remains the full-corpus search surface.

**Validate.** Measure index size vs `navigator.storage.estimate()` quota for a representative kept set; set the cap/eviction threshold where the body index falls back to title-only.

**Fallback.** Title-only search when the corpus/index exceeds the storage budget.

---

## 19.6 Server-owned protocol items (track & confirm)

These track the **server's canonical ledger**; the client must match each exactly and **lock it via shared conformance vectors** ([18 §18.6](18-build-ci-testing.md)), failing CI on any drift. Each has a **failing-by-default conformance test** that turns green when confirmed.

- **PINNED** — frame `magic` = `"NYXC"`, `version` = `0x01`, AAD = `magic ‖ version ‖ key_id ‖ file_id ‖ object_kind`, with the `object_kind` enum ([06 §6.3](06-cryptography.md)).
- **PINNED** — **HPKE suite IDs**: DHKEM(X25519, HKDF-SHA256)/HKDF-SHA256/AES-256-GCM = `KEM 0x0020`, `KDF 0x0001`, `AEAD 0x0002` ([06 §6.2](06-cryptography.md)).
- **PINNED** — recovery-escrow = **AES-256-GCM under the Argon2id-derived key** (not HPKE), shape + `AAD = userId ‖ version` ([06 §6.4](06-cryptography.md), [07](07-key-and-device-management.md)).
- **PINNED** — snapshot/compaction triggers (**≥ 200 updates / 5 min / last-leave**) ([09 §9.8](09-realtime-collaboration.md)).
- **PINNED** — token lifetimes: access ~5 min; **relay socket ticket single-use 60 s**; **guest share-session 15 min** renewable ([14 §14.5](14-authentication.md)).
- **PINNED** — **CRDT wire protocol** (Yjs reference, pinned vs ydotnet/ykt): byte-identical merged state, encoded updates, state-vector reconstruction ([09 §9.12](09-realtime-collaboration.md)).
- *Track & confirm* — the only item still genuinely loose server-side: exact `POST /sync/changes` / `GET /sync/manifest` payload shapes and `ref` formats ([08](08-sync-engine.md)). The client codes defensively against the server spec and locks them once the server pins them.

**Action**: one tracking item per bullet against the server spec; the failing-by-default test for each turns green when the server pins the value.

---

## 19.7 Ink parity in the browser — *DECIDED (view-first)*

**Decision.** **View-first, basic editing.** Capture with **Pointer Events** on `<canvas>` using `getCoalescedEvents()` for sub-frame samples, `pressure`/`tiltX`/`tiltY` where exposed, and `pointerType` for palm rejection; render on a `desynchronized` 2D context for low latency ([02 §2.8](02-tech-stack-and-libraries.md), [10](10-editors.md)). Persist in the **shared deterministic CBOR** format so files round-trip and the BLAKE3 address is reproducible with desktop/Android ([10 §10.5](10-editors.md)).

**Validate.** Cross-engine spike on coalesced/predicted-event availability, pressure/tilt fidelity, and palm-rejection reliability on real touch/stylus hardware; round-trip + stable-address conformance tests ([18 §18.4](18-build-ci-testing.md)).

**Fallback.** **View-only** ink on engines/devices where capture fidelity or palm rejection is inadequate; editing remains a desktop/Android showcase.

---

## 19.8 Multi-tab coordination — *DECIDED (single leader tab)*

**Decision.** Elect a **single leader tab** that owns the relay connection and sync loop, using the **Web Locks API** (held lock = leader) with **BroadcastChannel** to relay state/edits to follower tabs; on leader close the lock releases and a follower is elected. Followers render from the shared IndexedDB and receive live updates over BroadcastChannel ([01 §1.8](01-architecture.md), [09 §9.10](09-realtime-collaboration.md)).

**Validate.** Multi-tab e2e ([18 §18.5](18-build-ci-testing.md)): edits propagate across tabs, exactly one relay connection exists, election survives leader close; soak-test for split-brain.

**Fallback.** BroadcastChannel-only heartbeat election where Web Locks is unavailable; worst case, a "this account is open in another tab" banner that suspends sync in non-leaders.

---

## 19.9 Storage persistence / eviction — *DECIDED (persist() + re-derivable)*

**Decision.** Call **`navigator.storage.persist()`** at enrollment/first-use and surface the granted/denied state; monitor `navigator.storage.estimate()`. Treat keys and the cached subset as **re-derivable** (re-unlock + re-download) so eviction is recoverable, never data loss given the recovery key. Encourage **PWA install** (improves persistence) ([16 §16.5](16-offline-and-pwa.md)).

**Validate.** Per-browser spike: persist() grant rates, quota ceilings, Safari ITP behaviour after inactivity, eviction-recovery path (re-unlock + re-fetch).

**Fallback.** If persistence is denied (common on Safari without install), run **session-scoped** with explicit "not saved on this browser" UI; rely on re-download/re-unlock.

---

## 19.10 Static-export routing & runtime config — *DECIDED (client catch-all + SPA fallback)*

**Decision.** Serve runtime routes via a **client-rendered optional catch-all** (`/share/[[...token]]`) resolved fully in the browser; configure the host to **fall back to the SPA shell** for unknown paths (200-rewrite to `index.html`). Resolve API base / OIDC authority / relay hub / share base at boot from a **`config.json`** (per-account override), **not** baked into the build ([00 §0.5](00-overview.md), [18 §18.2](18-build-ci-testing.md)).

**Validate.** Deploy-time test that deep-linking `/share/...` and an arbitrary client route load the shell on the static host/CDN; verify `config.json` is `no-cache` and editing it re-points the app without a rebuild.

**Fallback.** Hash-based routing (`/#/share/...`) if a host cannot do SPA-fallback rewrites — ensuring the **share key fragment** scheme stays unambiguous ([13](13-sharing.md)).

---

## 19.11 Status — all web items ratified

| # | Item | Decision | Remaining (validation only) | Owner | Gate |
|---|------|----------|-----------------------------|-------|------|
| 19.1 | Key storage & lock | Vault key + WebAuthn/passphrase; persistent default; idle lock | Cross-engine spike | Web | Phase 0 |
| 19.2 | WebCrypto/WASM perf | Worker offload; Argon2id m=64 MiB,t=3,p=1 | Perf spike at editing scale | Web | Phase 0 |
| 19.3 | X25519/Ed25519 | libsodium (WASM) | hpke-js suite-id + interop check | Web | Phase 0 |
| 19.4 | Guest data | In-memory only; fragment strip; 15-min token | Leakage audit | Web | Phase 2 |
| 19.5 | Search scope | MiniSearch local subset; title-only fallback | Index-size/quota check | Web | Phase 1 |
| 19.6 | Protocol PINNED values | Pinned to server | Conformance lock (failing-by-default tests); `/sync` payloads track | Server → Web | Per phase |
| 19.7 | Ink parity | View-first, Pointer Events + CBOR | Fidelity/palm spike; round-trip tests | Web | Before Phase 3 |
| 19.8 | Multi-tab | Web Locks + BroadcastChannel single leader | Multi-tab e2e + soak | Web | Phase 2 |
| 19.9 | Storage persistence | persist() + re-derivable | Per-browser spike (Safari ITP) | Web | Phase 0/1 |
| 19.10 | Static-export routing | Client catch-all + SPA fallback + config.json | Deploy-time routing test | Web | Phase 0 |

Every web-side fork is now **decided**; the only thing still owned elsewhere is the server's exact `/sync` payload shapes (19.6), which the client tracks and conformance-locks.
