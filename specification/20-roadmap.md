# 20 — Roadmap (Web)

Phase order mirrors the server ([server 15](https://github.com/Nyxite/server)) and the Android client ([android 20](https://github.com/Nyxite/android)) so all clients and the backend land each capability together. v1.0.0 is the complete E2EE web client (Phases 0–6); phases are **build order, not separate products**. **E2EE is foundational from Phase 0** and never retrofitted ([00 §0.4](00-overview.md)). The phase map is summarized in [00 §0.9](00-overview.md).

Before Phase 1 logic is committed, run the **early spikes** ([18 §18.8](18-build-ci-testing.md)): crypto/WASM perf, hpke-js suite-id check, in-browser key-storage/lock, SignalR-in-browser, multi-tab coordination, storage persistence.

Every phase is gated by the **cross-client conformance vectors** ([18 §18.6](18-build-ci-testing.md)) — crypto KATs/interop and the Yjs↔ydotnet↔yrs/UniFFI (Android) CRDT wire vectors must stay green; any drift from the server ledger fails CI.

## Phase 0 — Foundations
**Deliver**: app shell + client-side navigation as a **static-export SPA** + **PWA install** (Workbox SW, manifest, `navigator.storage.persist()`) ([00 §0.5](00-overview.md), [16](16-offline-and-pwa.md)); **account-scoped session + per-account IndexedDB/keys/cache** as the multi-account foundation ([01 §1.8](01-architecture.md), [04](04-local-data-model.md), [14](14-authentication.md)); native login (password+TOTP or passkey; enterprise Keycloak OIDC + PKCE pluggable) ([14](14-authentication.md)); **browser enrollment**, identity keypair (X25519 + Ed25519) generation, and the **recovery-key UX** (BIP39-24 → Argon2id → AES-256-GCM escrow) ([07](07-key-and-device-management.md)); key directory publish/lookup; the **non-extractable WebCrypto vault key wrapping the identity bundle** in IndexedDB ([07](07-key-and-device-management.md), [19 §19.1](19-open-questions.md)); structure CRUD with **encrypted names** decrypted in-browser ([05](05-api-client.md)); crypto engine in the Web Worker with conformance vectors ([06](06-cryptography.md), [18 §18.6](18-build-ci-testing.md)); runtime `config.json` instance resolution ([19 §19.10](19-open-questions.md)).
**Done when**:
- A user can log in (native password+TOTP or passkey; or enterprise OIDC + PKCE + TOTP), enroll the browser, set a recovery key, and **recover in a second browser** from the phrase.
- The app **installs as a PWA** and the encrypted project/folder/file structure browses offline-first with names decrypted in-browser.
- The **non-extractable vault key** wraps the identity bundle; idle lock + re-unlock work across Chromium/Firefox/Safari.
- Crypto KATs + cross-client vectors (AES-GCM framing, HPKE, Ed25519/X25519, BLAKE3, Argon2id) pass.

## Phase 1 — Notes that sync (single user)
**Deliver**: **markdown + plaintext** editors with view/edit modes — CodeMirror 6 bound to the **encrypted Yjs document**, react-markdown view ([10](10-editors.md)); encrypted CRDT backbone with offline catch-up (relay optional here) ([08 §8.4](08-sync-engine.md), [09](09-realtime-collaboration.md)); ciphertext **blob sync** + **on-demand download**; the server **sync policies** (`server-default`/`excluded`) + the client-local **keepInBrowser** pinning ([08 §8.2](08-sync-engine.md), [16 §16.2](16-offline-and-pwa.md)); **in-browser search** (MiniSearch) over the **local subset** with title-only fallback ([11](11-search.md), [19 §19.5](19-open-questions.md)).
**Done when**:
- A single user edits notes in the browser; edits sync to another device through the encrypted relay/REST, and **offline edits reconcile** on reconnect.
- **keepInBrowser** pinning and on-demand download behave; not-kept files fetch on open.
- **Local search** works over the cached subset; eviction drops bodies but keeps titles.
- CRDT wire vectors (Yjs reference) pass.

## Phase 2 — Collaboration & sharing
**Deliver**: **live relay collaboration** (client-side Yjs merge) with presence/awareness over `@microsoft/signalr` ([09](09-realtime-collaboration.md)); **account shares** (HPKE-wrapped FKs) + **link/guest shares** (URL-fragment keys) ([13](13-sharing.md)); **guest mode as the primary surface** — fragment capture + replaceState + scoped 15-min share-session token ([13](13-sharing.md), [19 §19.4](19-open-questions.md)); **rotation-based revocation** ([07](07-key-and-device-management.md)); **version history** with **client-side diffs + restore** ([12](12-version-history.md)); **multi-tab coordination** (single relay/syncing tab) ([19 §19.8](19-open-questions.md)).
**Done when**:
- Two users **and an anonymous guest via link** co-edit a document live in the browser; the guest's key comes only from the URL fragment, which never reaches a server.
- **Revoking a share** cuts off access and **rotates the key**.
- **History diffs and restore** run entirely in-browser.
- Multi-tab: exactly one relay connection; edits propagate across tabs.
- CRDT conformance with desktop (ydotnet) and Android (yrs/UniFFI) passes.

## Phase 3 — Handwriting
**Deliver**: **Pointer-events ink capture** (`pressure`/`tiltX`/`tiltY`, `getCoalescedEvents()`, palm rejection) on `<canvas>` with low-latency rendering ([10 §10.4](10-editors.md)); the **shared deterministic-CBOR** ink vector format, byte-stable for content addressing ([10 §10.5](10-editors.md)); **LWW / version-vector** ink sync as encrypted blobs ([08 §8.5](08-sync-engine.md)); **viewing first**, then basic editing ([19 §19.7](19-open-questions.md)).
**Done when**:
- Handwritten notes **render faithfully** in the browser and capture basic strokes where the device exposes pressure/tilt.
- Ink stores encrypted, syncs via LWW (losers retained in history), and **round-trips with the desktop/Android format** (stable BLAKE3 address).
- Ink format round-trip + stable-address conformance tests pass.

## Phase 4 — Admin & polish
**Deliver**: rich per-user/per-file **client-encrypted settings** + config ([05](05-api-client.md)); **device/key management UI** (revoke a browser, re-issue recovery, identity rotation) ([07](07-key-and-device-management.md)); **multi-account switcher** (add/switch/remove accounts, per-account instance host) over the Phase-0 scoping ([14](14-authentication.md), [15 §15.1](15-ui-and-navigation.md)); storage/quota controls + `persist()` surfacing ([16 §16.5](16-offline-and-pwa.md)); **audit surfacing** where applicable; **security settings** (app/idle lock, WebAuthn unlock) ([17](17-security.md)); **PWA hardening** (SW update flow, CSP/headers) ([18 §18.2](18-build-ci-testing.md)); **accessibility pass** ([15](15-ui-and-navigation.md)).
**Done when**:
- Settings/config are complete and **encrypted**; the multi-account switcher manages accounts across instances.
- The **Lighthouse/PWA + accessibility** budgets pass in CI ([18 §18.7](18-build-ci-testing.md)); SW update prompt works without disrupting an open edit.

## Phase 5 — Format expansion (optional in v1.0.0 scope)
**Deliver**: **office docs, source-code text types, and images as encrypted blobs**; source-code on the CRDT with a syntax view; **chunked/resumable ciphertext upload** for large binaries ([05 §5.6](05-api-client.md), [16 §16.5](16-offline-and-pwa.md)). Any processing (thumbnails/extraction) is **client-side** in the worker.
**Done when**: large encrypted binaries upload/download in chunks within the browser's memory/quota limits; new content types view/edit where applicable.

## Phase 6 — Advanced hardening (optional)
**Deliver**: **key-transparency / safety-number verification UI** for the key directory ([13](13-sharing.md)); optional **blind-index hardening** for client-side search if the server adds a leak-free scheme ([11](11-search.md)); metadata-graph-hiding support if the server adds it.
**Done when**: a user can verify a peer's safety number before sharing; any added index hardening leaks nothing to the server.

> **Multi-account / instance switching is in v1.0.0** (foundation in Phase 0, switcher UI in Phase 4), not deferred — see [14](14-authentication.md).

## Cross-cutting / later
- Samsung Notes `.sdoc` import (separate, non-trivial migration item — proprietary ink format).
- Resilience to a Redis-backplane multi-node relay (transparent to the client; hub contract unchanged).
- Desktop remains the **full-corpus** search/offline surface; the browser stays a constrained peer by design ([00 §0.3](00-overview.md)).

## Versioning
SemVer for the app; pre-1.0 milestone tags track phases. The API is consumed at `/api/v1`; the **crypto frame and CRDT wire protocol are pinned by conformance vectors** so primitives/keys can rotate without a format break ([06 §6.3](06-cryptography.md), [18 §18.6](18-build-ci-testing.md)). The **static bundle is versioned independently of instance config** (`config.json`) so a self-hoster upgrades by swapping assets without a rebuild ([18 §18.2](18-build-ci-testing.md)).
