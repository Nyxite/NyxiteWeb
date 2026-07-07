# 19 — Open Questions & Resolutions (Web)

Every web-specific open item is now **ratified** (this revision). Each entry below records the **decision**, **how to validate it (spike)**, and the **pre-planned fallback** if the spike fails — so implementation can proceed while the validations run. Items that are genuinely the **server's** to decide remain "track & confirm" ([§19.6](#196-server-owned-protocol-items-track--confirm)). Project-wide decisions stay canonical in the master [`docs/OPEN-DECISIONS.md`](https://github.com/Nyxite/Nyxite); this file folds into that tracker's Web section. Where an item parallels the Android client it stays aligned ([android 19](https://github.com/Nyxite/NyxiteAndroid)).

## 19.0 Decisions ratified

| Item | Decision | Where applied |
|------|----------|---------------|
| Rendering / deployment | **Static-export SPA** — Next.js 15 App Router, `output: 'export'`, CDN/static host, no server runtime | [00 §0.5](00-overview.md), [01 §1.1](01-architecture.md), [18 §18.1](18-build-ci-testing.md) |
| Offline model | **Full installable PWA** — Workbox SW + manifest + `navigator.storage.persist()`; **IndexedDB (Dexie)** local subset + **Cache Storage** blobs | [16 §16.1](16-offline-and-pwa.md), [04](04-local-data-model.md) |
| Text CRDT engine | **Yjs** (+ y-protocols, y-codemirror.next) — reference Yrs impl, lowest-risk of the three clients | [02 §2.7](02-tech-stack-and-libraries.md), [09 §9.12](09-realtime-collaboration.md) |
| Ink in the browser | **View + basic Pointer-Events editing**; shared **deterministic CBOR** format | [00 §0.3](00-overview.md), [10](10-editors.md), [§19.7](#197-ink-parity-in-the-browser--decided-view-first) |
| Multi-account | **Yes, from v1.0.0** — per-account IndexedDB/keys/cache/index; per-account instance host | [01 §1.8](01-architecture.md), [14](14-authentication.md) |
| Library stack | shadcn/Tailwind, TanStack Query + Zustand, WebCrypto + hpke-js + libsodium + hash-wasm, CodeMirror 6, @microsoft/signalr, fetch + WebAuthn API (native auth; oidc-client-ts for enterprise Keycloak), MiniSearch, cbor-x | [02](02-tech-stack-and-libraries.md) |
| **Key storage & lock** ([§19.1](#191-in-browser-key-storage--lock-model--decided)) | **Non-extractable WebCrypto vault key** wraps the identity bundle; unlock via **WebAuthn passkey where available, else passphrase**; idle auto-lock; **persistent ("remember this browser") by default** (session-only is a user choice) | [07 §7.3–7.4](07-key-and-device-management.md), [17](17-security.md) |
| **Asymmetric crypto library — hybrid PQC** ([§19.3](#193-asymmetric-crypto-library--decided-libsodium-classical--hybrid-pqc-at-v100)) | **libsodium-wrappers-sumo (WASM)** for classical X25519/Ed25519 **+ an audited WASM PQC lib** for the ML-KEM-768/ML-DSA-65 hybrid halves (v1.0.0, NIST level 3); behind `CryptoEngine`. PQC lib choice is the one open follow-up | [02 §2.6](02-tech-stack-and-libraries.md), [06 §6.2](06-cryptography.md) |
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
- **Identity bundle** (hybrid X25519+ML-KEM-768 / Ed25519+ML-DSA-65 private keys, FK cache) is **AES-256-GCM-wrapped by the vault key** and stored as ciphertext in IndexedDB. Unwrapped material lives only in the in-memory account `UserSession` ([01 §1.8](01-architecture.md)).
- **Unlock**: **WebAuthn/passkey** (PRF / `hmac-secret` extension to derive the unlock secret) **where available**, else **passphrase → Argon2id**; both gate the vault key.
- **Persistence**: **persistent ("remember this browser") by default**; **session-only** is an explicit user choice (vault key dropped on tab close). **Idle auto-lock** (configurable, default ~10 min) evicts the in-memory session.

**Validate.** Spike across **Chromium/Firefox/Safari**: non-extractable-key persistence in IndexedDB, WebAuthn PRF availability/coverage, structured-clone of `CryptoKey`, private-mode and storage-eviction behaviour.

**Fallback.** Where non-extractable-handle persistence or WebAuthn PRF is unavailable, that browser runs **session-only, in-memory** (no at-rest key; re-unlock/re-enroll each session) rather than weakening at-rest protection.

---

## 19.2 WebCrypto & WASM performance — *DECIDED (worker offload)*

**Decision.** All crypto runs in the **crypto Web Worker** ([01 §1.6](01-architecture.md)): WebCrypto `AES-GCM` (hardware-accelerated) for bulk seal/open; **hash-wasm** BLAKE3 (streamed) + Argon2id; **hpke-js** wrap/unwrap; **libsodium** X25519/Ed25519 + a **WASM PQC lib** (ML-KEM-768/ML-DSA-65) for the hybrid halves. Transfer `ArrayBuffer`s to avoid copies. **Argon2id default m = 64 MiB, t = 3, p = 1**, off-thread with a progress indicator; chosen params persisted in `recovery_blobs.kdf_params` so any browser can reproduce on recovery.

**Validate.** Perf spike: seal/open throughput on large docs/blobs, BLAKE3 over large binaries, Argon2id wall-time on low/mid/high machines and mobile browsers; confirm 64 MiB does not OOM mobile Safari.

**Fallback.** If 64 MiB pressures the weakest target, drop to **m = 32 MiB, t = 4** (≈constant cost); chunk large-blob hashing/sealing cooperatively if worker latency spikes.

---

## 19.3 Asymmetric crypto library — *DECIDED (libsodium classical + hybrid PQC at v1.0.0)*

**Decision.** Use **libsodium-wrappers-sumo (WASM)** for the **classical** X25519 and Ed25519 halves — consistent, audited behaviour on every target and a clean building block under HPKE ([02 §2.6](02-tech-stack-and-libraries.md)). Kept behind `CryptoEngine` so a future native path can replace it without touching repositories.

**Post-quantum hybrid — resolved, hybrid at v1.0.0 (ratified 2026-07-07, [OPEN-DECISIONS](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md)).** Every asymmetric seam ships **hybrid classical + PQC** from v1.0.0: HPKE key wrap / agreement = **X25519 + ML-KEM-768** (suite `X25519MLKEM768`), signatures = **Ed25519 + ML-DSA-65**, both **NIST level 3**; symmetric primitives are unchanged ([06 §6.2](06-cryptography.md)). This is **no longer a "future migration"** — it lands with v1. **Open follow-up/dependency [P]:** WebCrypto exposes no ML-KEM/ML-DSA and libsodium does not provide them, so the PQC halves need an **audited WASM PQC library** — the specific choice is the one remaining item, kept behind `CryptoEngine` and gated on the shared vectors ([06 §6.13](06-cryptography.md), [18 §18.8](18-build-ci-testing.md)).

**Validate.** Select the WASM PQC lib; confirm the **hybrid `X25519MLKEM768` HPKE suite id + hybrid-KEM construction and the Ed25519+ML-DSA-65 signature suite match the server exactly** (classical half DHKEM(X25519,HKDF-SHA256)/HKDF-SHA256/AES-256-GCM = `0x0020`/`0x0001`/`0x0002`), and that libsodium X25519/Ed25519 **and** the PQC lib's ML-KEM-768/ML-DSA-65 outputs interop with server/desktop/Android via the shared vectors ([18 §18.6](18-build-ci-testing.md)).

**Fallback.** Native WebCrypto Ed25519/X25519 on engines that fully support it (feature-detected, classical halves only), gated behind passing the same conformance vectors; the PQC halves stay on the WASM lib (no native option exists).

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
- **PINNED** — **Hybrid HPKE suite**: `X25519MLKEM768` — classical DHKEM(X25519, HKDF-SHA256) `KEM 0x0020` concatenated with **ML-KEM-768** / HKDF-SHA256 `0x0001` / AES-256-GCM `0x0002` (NIST level 3), plus the **hybrid Ed25519 + ML-DSA-65** signature suite ([06 §6.2](06-cryptography.md)).
- **PINNED** — recovery-escrow = **AES-256-GCM under the Argon2id-derived key** (not HPKE), shape + `AAD = userId ‖ version` ([06 §6.4](06-cryptography.md), [07](07-key-and-device-management.md)).
- **PINNED** — snapshot/compaction triggers (**≥ 200 updates / 5 min / last-leave**) ([09 §9.8](09-realtime-collaboration.md)).
- **PINNED** — token lifetimes: access ~5 min; **relay socket ticket single-use 60 s**; **guest share-session 15 min** renewable ([14 §14.5](14-authentication.md)).
- **PINNED** — **CRDT wire protocol** (Yjs reference, pinned vs ydotnet/android yrs/UniFFI): byte-identical merged state, encoded updates, state-vector reconstruction ([09 §9.12](09-realtime-collaboration.md)).
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

**Decision.** Serve runtime routes via a **client-rendered optional catch-all** (`/share/[[...token]]`) resolved fully in the browser; configure the host to **fall back to the SPA shell** for unknown paths (200-rewrite to `index.html`). Resolve API base / relay hub / share base (and the OIDC authority for the enterprise Keycloak option) at boot from a **`config.json`** (per-account override), **not** baked into the build ([00 §0.5](00-overview.md), [18 §18.2](18-build-ci-testing.md)).

**Validate.** Deploy-time test that deep-linking `/share/...` and an arbitrary client route load the shell on the static host/CDN; verify `config.json` is `no-cache` and editing it re-points the app without a rebuild.

**Fallback.** Hash-based routing (`/#/share/...`) if a host cannot do SPA-fallback rewrites — ensuring the **share key fragment** scheme stays unambiguous ([13](13-sharing.md)).

---

## 19.11 Status — all web items ratified

| # | Item | Decision | Remaining (validation only) | Owner | Gate |
|---|------|----------|-----------------------------|-------|------|
| 19.1 | Key storage & lock | Vault key + WebAuthn/passphrase; persistent default; idle lock | Cross-engine spike | Web | Phase 0 |
| 19.2 | WebCrypto/WASM perf | Worker offload; Argon2id m=64 MiB,t=3,p=1 | Perf spike at editing scale | Web | Phase 0 |
| 19.3 | Asymmetric crypto (hybrid PQC) | libsodium (classical) + WASM PQC lib (ML-KEM-768/ML-DSA-65); hybrid at v1.0.0 | Select PQC lib; hybrid `X25519MLKEM768` suite-id + interop check | Web | Phase 0 |
| 19.4 | Guest data | In-memory only; fragment strip; 15-min token | Leakage audit | Web | Phase 2 |
| 19.5 | Search scope | MiniSearch local subset; title-only fallback | Index-size/quota check | Web | Phase 1 |
| 19.6 | Protocol PINNED values | Pinned to server | Conformance lock (failing-by-default tests); `/sync` payloads track | Server → Web | Per phase |
| 19.7 | Ink parity | View-first, Pointer Events + CBOR | Fidelity/palm spike; round-trip tests | Web | Before Phase 3 |
| 19.8 | Multi-tab | Web Locks + BroadcastChannel single leader | Multi-tab e2e + soak | Web | Phase 2 |
| 19.9 | Storage persistence | persist() + re-derivable | Per-browser spike (Safari ITP) | Web | Phase 0/1 |
| 19.10 | Static-export routing | Client catch-all + SPA fallback + config.json | Deploy-time routing test | Web | Phase 0 |

Every web-side fork is now **decided**; the only thing still owned elsewhere is the server's exact `/sync` payload shapes (19.6), which the client tracks and conformance-locks.

---

## 19.12 Group sharing (enterprise/family) — *web validation spikes*

Group sharing ([features/groups.md](https://github.com/Nyxite/Nyxite), [06 §6.14](06-cryptography.md), [07 §7.11](07-key-and-device-management.md), [13 §13.9](13-sharing.md)) reuses **already-decided** web machinery — the same HPKE suite, worker offload, libsodium/hpke-js stack, and `412`/`409` rotation flow — so it introduces **no new crypto fork** and **no new primitive** (a group public key is just another HPKE target). Build step **P4.4-WEB-1**, after key transparency (Phase 4.3). The remaining items are **web-specific validation spikes**, folded into the master [`docs/OPEN-DECISIONS.md`](https://github.com/Nyxite/Nyxite) group section.

### 19.12.1 In-browser group-key storage

**Decision.** Group private keys are **never persisted unwrapped**: the *wrapped* group-key grant may cache in IndexedDB ([04 §4.2](04-local-data-model.md)) exactly like a wrapped FK, and the unwrapped group private key lives **in the crypto worker only**, short-lived, dropped on lock/logout/account-switch ([06 §6.14](06-cryptography.md), [07 §7.11.3](07-key-and-device-management.md), [07 §7.9](07-key-and-device-management.md)). Unlike an FK, a group private key is **raw bytes for libsodium/HPKE unwrap** (not a non-extractable `CryptoKey`), so it takes the identity-bundle mitigations: `.fill(0)` on drop, never in `String`/`localStorage`/URL/logs ([06 §6.12](06-cryptography.md)).

**Validate.** Confirm a member of many groups/scopes keeps only bounded in-worker group-key handles (no unbounded retention), and that lock/switch zeroizes them; assert no unwrapped group key ever reaches IndexedDB, the main thread, or logs ([18 §18.5](18-build-ci-testing.md)).

**Fallback.** Re-unwrap the group key per scope on demand (from the cached wrapped grant) rather than holding handles across scopes if memory pressure shows up on the weakest target.

### 19.12.2 WASM HPKE performance for group unwrap/rotation

**Decision.** Group wrap/unwrap runs in the **crypto worker** on the existing **hpke-js + libsodium + WASM PQC** stack (the hybrid `X25519MLKEM768` suite, §19.2/§19.3), transferring `ArrayBuffer`s. Enrollment is **O(1)** (one grant); the heaviest op is **scope rotation** — re-wrap the group key to N remaining members + optional re-seal of the scope's DEKs — which is N small HPKE wraps, not content re-encryption ([07 §7.11.4](07-key-and-device-management.md), [13 §13.9](13-sharing.md)).

**Validate.** Perf spike: group-key unwrap + DEK-to-group unwrap latency on read; rotation wall-time for a large group / large scope (many DEKs) on low/mid machines and mobile Safari; confirm the browser — the **most storage- and memory-constrained** client — stays responsive and does not OOM on a big re-seal.

**Fallback.** Chunk the rotation re-wrap/re-seal cooperatively across worker turns with progress UI; for very large scopes, drive re-seal as a resumable batch queue (the §13.1/§13.9.2 pattern) rather than one synchronous pass.

### 19.12.3 Group UX in a constrained client

**Decision.** The browser hosts the full group-management screen ([13 §13.9.1](13-sharing.md)) — create/enroll/remove, transparency-verified enrollment with a clear "key not verified" rejection, reader-group attachment on a project/folder, and the **honest revocation UI** ("already-decrypted content can't be recalled"). Because the browser cannot hold the whole corpus, a **subtree grant / scope re-seal shows progress and survives reconnect** (resumable batch, §13.9.2), and group state renders from the local subset with re-fetch on demand.

**Validate.** UX spike: enrollment transparency-failure messaging is unambiguous; reader-group cascade (`inherit`/group/none) is legible across project→folder→file; the revocation caveat is surfaced non-dismissably; batch grant/rotation progress and resume behave on flaky mobile connections.

**Fallback.** Where a scope is too large to manage responsively in-browser, surface a "manage on desktop" hint (desktop is the full-corpus surface, [00 §0.3](00-overview.md)) while keeping read/enroll working in the browser.

### 19.12.4 Status

| # | Item | Decision | Remaining (validation only) | Owner | Gate |
|---|------|----------|-----------------------------|-------|------|
| 19.12.1 | Group-key storage | Wrapped grant cached; unwrapped key in-worker only, zeroized | Retention/zeroization + leakage audit | Web | Phase 4.4 |
| 19.12.2 | WASM HPKE perf | Worker offload; O(1) enroll; rotation = N small wraps | Rotation/re-seal perf spike (constrained client) | Web | Phase 4.4 |
| 19.12.3 | Group UX | Full mgmt screen; transparency + honest-revocation UI; resumable batches | UX + flaky-network spike | Web | Phase 4.4 |
| — | Enrollment trust | **Requires key transparency (Phase 4.3)** — no self-signature-only enrollment | Track Phase 4.3 landing in v1.0.0 | Server → Web | Phase 4.3 |
