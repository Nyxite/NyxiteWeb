# 18 — Build, CI & Testing

The web client ships as a **static bundle** with **no server runtime** ([00 §0.5](00-overview.md), [01 §1.1](01-architecture.md)), so "build" means *static export + service-worker generation + WASM bundling*, and "deploy" means *publish immutable assets to a CDN/static host*. Because the browser is a **full cryptographic peer** ([00 §0.2](00-overview.md)), the non-negotiable gate is **conformance** — the same crypto and CRDT bytes as server, desktop, and Android ([§18.6](#186-conformance-vectors-the-critical-gate)). This doc mirrors the Android build/test spec ([android 18](https://github.com/Nyxite/NyxiteAndroid)) for the browser.

## 18.1 Build setup

- **pnpm** workspace; **Node 20+ build-time only** (nothing runs server-side in prod — [02 §2.1](02-tech-stack-and-libraries.md)). Lockfile committed; CI runs `pnpm install --frozen-lockfile`.
- **Next.js 15 App Router** with **`output: 'export'`** → a static `out/` of HTML/JS/CSS/WASM. SSR, route handlers, middleware, and image optimization are disabled by config — a build that introduces a server-only API must **fail** ([02 §2.1](02-tech-stack-and-libraries.md)).
- **TypeScript `strict`**; `tsc --noEmit` is a build-blocking step, not advisory.
- Asset hashing: all JS/CSS/WASM emitted with **content-hashed filenames** (cache-busting) so the CDN can serve them `immutable` and the service worker can precache by revision ([§18.2](#182-distribution--deployment)).
- **WASM bundling**: `hash-wasm` (BLAKE3, Argon2id), `libsodium-wrappers-sumo`, the `hpke-js` build, and the CBOR codec ship as **self-hosted, content-hashed `.wasm`/JS assets** — never fetched from a third-party CDN (CSP forbids it, [17](17-security.md)). WASM is loaded inside the **crypto Web Worker** ([01 §1.6](01-architecture.md)); verify the bundler emits worker chunks + their WASM as same-origin assets.
- **Service worker** built with **Workbox** ([02 §2.9](02-tech-stack-and-libraries.md)): precache manifest = the hashed app-shell assets; runtime routes for navigation fallback. The SW build is part of the export, after asset hashing, so the precache revisions match.

### Build outputs

| Output | Contents |
|--------|----------|
| `out/` static site | Hashed JS/CSS/WASM, HTML shells, `manifest.webmanifest`, icons |
| `sw.js` + precache manifest | Workbox SW + revisioned app-shell list ([16 §16.1](16-offline-and-pwa.md)) |
| `config.json` (NOT baked in) | Runtime instance config — supplied per deployment ([§18.2](#182-distribution--deployment)) |
| Source maps | Generated, uploaded to the (self-hosted) error sink only; **not** publicly served in release |

### Bundle-size budget **[P]**

| Bundle | Budget (gzip) | Rationale |
|--------|---------------|-----------|
| Initial route + app shell JS | ≤ 250 KB | First paint / install on mobile |
| Crypto worker (excl. WASM) | ≤ 120 KB | Loaded async after shell |
| WASM total (sodium+hash-wasm+hpke) | tracked, no hard cap | Required; lazy-loaded in worker |
| Editor chunk (CodeMirror + Yjs + binding) | lazy, ≤ 350 KB | Only on opening a file |

CI fails on a budget breach (`size-limit` or the bundler analyzer with thresholds). Heavy deps (CodeMirror, Yjs, CBOR, markdown) are **code-split** and lazy-loaded per route.

## 18.2 Distribution & deployment

- **Static host / CDN.** The `out/` bundle is published as immutable assets through **NyxiteDeploy** (the shared deploy repo) to a static host/CDN. There is no app server; the same bundle a self-hoster downloads is the one we ship.
- **Instance-agnostic bundle + runtime `config.json`.** The API base (`/api/v1`), relay hub URL, public-share base, and (for the enterprise Keycloak option) the OIDC authority are **not baked into the build** — they are fetched at runtime from a small `config.json` at app boot, with per-account override for multi-instance ([00 §0.5](00-overview.md), [01 §1.8](01-architecture.md), [14](14-authentication.md)). A self-hoster serves the identical static bundle pointed at their instance by editing only `config.json`. CI builds **once**; deployment is config, not a rebuild.
- **HTTPS + headers.** Served only over HTTPS (secure context required for WebCrypto/SW — [00 §0.6](00-overview.md)). The host/CDN must emit the **CSP and security headers from [17](17-security.md)** (script-src self, `connect-src` limited to the configured instance API + relay, plus the OIDC authority under the enterprise Keycloak option, `frame-ancestors 'none'`, COOP/COEP as needed for worker isolation). Header config travels with the deploy, not the bundle.
- **Cloudflare note.** Cloudflare may front public traffic (matching the server's edge, [server 14 §14.6](https://github.com/Nyxite/NyxiteServer)); ensure it does **not** rewrite/strip the CSP or the URL **fragment** (the fragment never reaches any server, but proxies must not interfere with client-side routing of `/share/...`). Disable any "auto-minify"/script-injection that would violate SRI/CSP.
- **Cache strategy.** Hashed assets → `Cache-Control: immutable, max-age=1y`. `index.html`/entry shells and `config.json` → `no-cache` (must revalidate) so a deploy is picked up.

### PWA versioning & SW update

- App version stamped at build (git SHA + SemVer). The Workbox SW precache is keyed by asset revision; a new deploy yields a new SW.
- **Update flow**: SW installs in the background, then prompts "Update available — reload" (skip-waiting only on user action, to avoid swapping code mid-edit). On reload the new precache activates and stale caches are pruned ([16 §16.1](16-offline-and-pwa.md)).
- The SW **never** decrypts or caches plaintext content or keys ([01 §1.6](01-architecture.md), [17](17-security.md)); it precaches only the app shell.

## 18.3 Static analysis & architecture enforcement

- **`tsc --noEmit`** (strict) — type gate.
- **ESLint** + **Prettier** — style/lint gate.
- **`eslint-plugin-boundaries`** enforces the layering and the **crypto boundary** ([01 §1.2](01-architecture.md)): network modules (`ApiClient`, `RelayClient`) must not import `CryptoEngine` or any domain content type; a type carrying a plaintext field must not reach a network client; presentation must not import data internals. This is the web analogue of the Android `konsist` architecture test ([android 18 §18.3](https://github.com/Nyxite/NyxiteAndroid)). A boundary violation **fails CI**.
- Dependency policy lint ([02 §2.12](02-tech-stack-and-libraries.md)): no new dependency that loads remote code or phones home; license check (PolyForm-redistributable).

## 18.4 Test strategy (critical surfaces first)

| Layer | Tools | Focus |
|-------|-------|-------|
| **Crypto conformance** | Vitest + shared KAT/cross-client vectors | **AES-GCM framing, HPKE wrap/unwrap, Ed25519, X25519, BLAKE3, Argon2id** interop byte-for-byte with server/desktop/Android. Wrap in browser → unwrap on server impl and vice-versa. **Highest priority** ([06 §6.9](06-cryptography.md), [§18.6](#186-conformance-vectors-the-critical-gate)). |
| **CRDT conformance** | Vitest + shared Yrs wire vectors | Yjs is the **reference**, but still pinned vs ydotnet (desktop) and yrs/UniFFI (Android): identical merged state, identical encoded updates, state-vector reconstruction ([09 §9.12](09-realtime-collaboration.md)). |
| Domain | Vitest | Use cases, policy, conflict/sync state machine — **pure TS**, no DOM/IO ([01 §1.4](01-architecture.md)). |
| Data (repos) | Vitest + **fake-indexeddb** + a fake/mocked server | Dexie schema + migrations (asserted), API mapping, problem+json error mapping, outbox/idempotency, delta/manifest reconcile ([04](04-local-data-model.md), [05](05-api-client.md), [08](08-sync-engine.md)). |
| Crypto engine (real) | Vitest in **jsdom/Node with real WebCrypto** + WASM | Real `SubtleCrypto` (Node 20 `crypto.subtle`), real WASM (sodium/hash-wasm/hpke); KATs run against the actual libs, not mocks. |
| Stores/hooks | React Testing Library | Zustand stores, query hooks (decrypt-then-render), loading/empty/error/locked states. |
| UI components | React Testing Library (+ jsdom) | Screens, editor mode toggles, dialogs, share sheet, deep-link routing ([15](15-ui-and-navigation.md)). |
| Ink | Vitest + synthetic Pointer Event streams | Stroke capture (pressure/tilt/coalesced), CBOR round-trip + **stable BLAKE3 address**, render ([10 §10.5](10-editors.md)). |
| E2E | **Playwright** against a test server | Full flows ([§18.5](#185-end-to-end-flows-playwright)). |

**jsdom + real crypto policy**: prefer **real WebCrypto and real WASM** in tests (available in Node 20 and the Vitest browser/jsdom env) over mocks, so conformance and framing are exercised for real. Only network and timers are faked. WASM modules are loaded in-test via the same worker-less code path where possible.

## 18.5 End-to-end flows (Playwright)

Run against a test server (Testcontainers-backed server stack or a staging instance, [server 14 §14.8](https://github.com/Nyxite/NyxiteServer)). Required scenarios:

1. **Login** — native auth (password+TOTP, passkey) and the enterprise Keycloak OIDC + PKCE option, token acquisition, silent refresh ([14](14-authentication.md)).
2. **Enroll + recovery** — browser enrollment, generate recovery (BIP39-24), recover in a second browser context.
3. **Create / edit / sync** — create project/folder/file with encrypted names, edit markdown on the Yjs doc, blob sync to a second context, offline edit then reconcile.
4. **Share-link GUEST flow** — create a link share, open it in a **clean (no-account) context**, key from URL fragment, guest edits over the relay via a share-session token; verify the fragment never appears in any network request ([13](13-sharing.md), [17](17-security.md)).
5. **Offline / PWA** — install, go offline, read/edit cached subset, reconnect and sync; SW update prompt ([16](16-offline-and-pwa.md)).
6. **Multi-tab relay** — two tabs of the same account: a single relay/syncing tab elected; edits in one tab appear in the other; election survives closing the leader ([09](09-realtime-collaboration.md), [19 §19.8](19-open-questions.md)).
7. **Revoke / rotate** — revoke a share, confirm access cut + key rotation; **version history** diff + restore.

## 18.6 Conformance vectors (the critical gate)

Interop is non-negotiable in a zero-knowledge multi-client system. The build **fails on any drift from the server's canonical ledger**.

### Crypto known-answer & cross-client vectors

- Shared vector files (KATs + cross-client pairs) are **co-owned with server/desktop/Android** and checked into the web repo's `core-crypto` test resources, updated in lockstep ([android 18 §18.6](https://github.com/Nyxite/NyxiteAndroid)).
- Coverage:
  - **AES-256-GCM framing** — `magic "NYXC" | version 0x01 | key_id | nonce | ciphertext | tag`, AAD = `magic ‖ version ‖ key_id ‖ file_id ‖ object_kind`; decrypt server-produced frames and produce byte-identical frames for fixed inputs ([06 §6.3](06-cryptography.md)).
  - **HPKE wrap/unwrap interop** — DHKEM(X25519,HKDF-SHA256)/HKDF-SHA256/AES-256-GCM (`KEM 0x0020`/`KDF 0x0001`/`AEAD 0x0002`); browser wraps an FK → server/desktop/Android unwrap, and vice-versa ([06 §6.4](06-cryptography.md)).
  - **Ed25519** sign/verify, **X25519** agreement, **BLAKE3-256** content addressing, **Argon2id** derivation against fixed `kdf_params` (m=64 MiB, t=3, p=1 floor — [19 §19.3](19-open-questions.md)) — all against shared KATs.
- A spike confirms **hpke-js suite IDs match exactly** and **libsodium X25519/Ed25519** match the server's outputs ([19 §19.2](19-open-questions.md), [19 §19.4](19-open-questions.md)).

### CRDT wire-protocol conformance

- Yjs is the **reference** implementation but is still **pinned** against `ydotnet` and android `yrs/UniFFI` via shared Yrs wire vectors ([02 §2.7](02-tech-stack-and-libraries.md), [09 §9.12](09-realtime-collaboration.md)):
  - **byte-identical merged state** for a fixed update sequence;
  - **byte-identical encoded updates** for the same edit sequence;
  - **state-vector reconstruction** from snapshot + log matches the other clients.
- Mirrors the server's `CrdtConformanceTests`; a protocol change that breaks a vector must **break CI on all clients**.

### Failing-by-default tracking tests

For each **server-owned PINNED value** ([19 §19.6](19-open-questions.md)) there is a conformance test that is **red until the value is confirmed against the server ledger** and then locks it: frame magic/version/AAD/`object_kind` enum, HPKE suite IDs, recovery AES-GCM-under-Argon2id scheme (AAD = `userId ‖ version`), snapshot triggers (≥200 / 5 min / last-leave), token lifetimes (access ~5 min / socket ticket 60 s single-use / guest share-session 15 min).

## 18.7 CI pipeline

**On PR** (fast, blocking):
1. `pnpm install --frozen-lockfile`
2. `tsc --noEmit` (strict)
3. ESLint (incl. `eslint-plugin-boundaries` crypto/layer test) + Prettier check
4. Vitest **unit** (domain, data with fake server, stores)
5. **Crypto conformance** KATs/cross-client vectors (real WebCrypto + WASM)
6. **CRDT conformance** vectors
7. React Testing Library **component** tests
8. **Build + static export** + **bundle-size** check
9. **Dependency audit** (`pnpm audit`) + **supply-chain scan** (e.g. OSV/Socket) — no remote-code or phone-home deps ([02 §2.12](02-tech-stack-and-libraries.md))

**On main / tags** (extended):
10. **Playwright e2e** against the test server ([§18.5](#185-end-to-end-flows-playwright)) — incl. guest, offline/PWA, multi-tab
11. **Lighthouse / PWA** audit (installability, offline, performance, a11y budgets — [15](15-ui-and-navigation.md), [16](16-offline-and-pwa.md))
12. Publish immutable assets via NyxiteDeploy ([§18.2](#182-distribution--deployment))

**Fail fast on any conformance regression** — crypto or CRDT drift blocks merge unconditionally.

## 18.8 Early validation spikes (do these first)

1. **Crypto/WASM perf spike** — AES-256-GCM + HPKE(X25519) + Argon2id + BLAKE3 in the crypto worker at editing scale (large docs/blobs) across Chromium/Firefox/Safari ([19 §19.2](19-open-questions.md)).
2. **hpke-js suite-id check** — confirm exact match to the server's X25519/HKDF-SHA256/AES-256-GCM suite; confirm libsodium X25519/Ed25519 interop ([19 §19.4](19-open-questions.md)).
3. **Key-storage/lock spike** — non-extractable WebCrypto vault key wrapping the identity bundle; WebAuthn/passphrase unlock; idle lock; behavior across Chromium/Firefox/Safari ([19 §19.1](19-open-questions.md)).
4. **SignalR-in-browser spike** — `@microsoft/signalr` binary `EncryptedUpdate`, group join/leave, reconnect/backoff, guest share-token upgrade ([09](09-realtime-collaboration.md)).
5. **Multi-tab coordination spike** — Web Locks / BroadcastChannel single-relay/single-syncing-tab election ([19 §19.8](19-open-questions.md)).
6. **Storage persistence spike** — `navigator.storage.persist()`, quota, Safari ITP eviction per browser ([19 §19.9](19-open-questions.md), [16 §16.5](16-offline-and-pwa.md)).
