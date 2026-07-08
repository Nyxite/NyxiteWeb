# 02 — Tech Stack & Libraries

Concrete choices with rationale. Versions are **[P]** floors at time of writing; pin exact versions in the lockfile and keep them current. Prefer the smallest well-maintained library that meets the need; **every dependency in a zero-knowledge browser client is attack surface** (it runs with full access to plaintext and keys) — see the dependency policy ([§2.11](#211-dependency-policy)) and [17-security.md](17-security.md).

## 2.1 Language, framework, build, baseline

| Concern | Choice | Notes |
|---------|--------|-------|
| Language | **TypeScript 5.x** (`strict`) | No untyped content paths; domain layer is pure TS. |
| Framework | **Next.js 15 (App Router)** with **`output: 'export'`** | Static-export SPA; **no server runtime** ([00 §0.5](00-overview.md), [01](01-architecture.md)). React Server Components/SSR of data are **not used**. |
| UI runtime | **React 19** | Concurrent rendering; matches Next 15. |
| Package manager / runtime | **pnpm**; **Node 20+** (build only) | Node is build-time only; nothing runs server-side in prod. |
| Bundler | Next's bundler (Turbopack/webpack) | Static export output is plain HTML/JS/CSS/WASM. |

> **Static-export caveat (must handle):** `output: 'export'` disables SSR, route handlers, middleware, and image optimization. Runtime-dependent routes (e.g. `/share/{token}`) use a **client-rendered optional catch-all** segment; data is fetched in the browser ([15 §15.1](15-ui-and-navigation.md)). This is intentional — it keeps the deployment a dumb static host and removes any server surface that could see content.

## 2.2 UI & styling

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Component system | **shadcn/ui** (Radix UI primitives, copied-in components) | Accessible, unstyled-by-default primitives we own in-repo; no opaque component CDN. |
| CSS | **Tailwind CSS v4** | Utility styling shadcn is built on; tree-shaken. |
| Icons | **lucide-react** | Matches shadcn. |
| Theming | CSS variables (light/dark, Nyxite brand); **dark-first** | Notes app, night use, "Nyx" branding ([15 §15.4](15-ui-and-navigation.md)). |
| Markdown render (view mode) | **react-markdown** + **remark-gfm** + **rehype-sanitize** | GFM tables/task-lists; **sanitized**, **no remote fetch** from content (privacy/XSS) ([10 §10.2](10-editors.md), [17](17-security.md)). |
| Text editor (edit mode) | **CodeMirror 6** + **y-codemirror.next** | Collaborative text editing bound directly to the Yjs doc, with remote cursors; works for markdown and plaintext (and `sourcecode` later). |

> **Design tokens are shared, not hand-defined here.** Colors (deep-purple accent), typography (Manrope UI / Source Serif 4 content), spacing, radius, shadow, and motion are **not** authored in this repo — they come from the shared [NyxiteDesign](https://github.com/Nyxite/NyxiteDesign) system, whose `nyxite-tokens.json` is the platform-agnostic single source of truth. The Tailwind v4 theme extension and the CSS custom properties (`:root` + `[data-theme="dark"]`, consumed as `var(--accent)` etc.) are **generated** from that file by the token build pipeline so the client never drifts ([03 §3.1](03-project-structure.md), [18 §18.1](18-build-ci-testing.md)). shadcn/ui supplies the standard components — this is **Layer A** (tokens + standard components) of the design system's two-layer adoption. See [OPEN-DECISIONS DS](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (Live decision DS).

## 2.3 State, data fetching, routing

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Server-state cache | **TanStack Query v5** | Caching, retries, invalidation for the ciphertext/structural REST surface ([05](05-api-client.md)); query fns decrypt via repositories. |
| Client/UI state | **Zustand** | Small, simple stores for editor mode, selection, presence, sync badges ([01 §1.3](01-architecture.md)). |
| Routing | **Next.js App Router** (client-rendered) | File-based routes; deep links for share URLs ([15](15-ui-and-navigation.md)). |
| Forms/validation | **react-hook-form** + **zod** | Typed forms; zod also validates decoded DTOs ([05](05-api-client.md)). |

## 2.4 Local persistence (in-browser)

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Structured store | **IndexedDB via Dexie** | Single source of truth: structure, sync state, cached metadata/plaintext, search index ([04](04-local-data-model.md)). |
| Blob/ciphertext cache | **Cache Storage API** (+ IndexedDB for metadata) | Streamable ciphertext + decrypted ink/binary; works with the service worker ([16](16-offline-and-pwa.md)). |
| Key material | **non-extractable WebCrypto `CryptoKey`** stored in IndexedDB (structured-clone of the key handle) | The browser's nearest analogue to a keystore: the private bits can't be read back out as bytes ([07](07-key-and-device-management.md), [17](17-security.md)). |
| Non-secret prefs | IndexedDB / a small `localStorage` of **non-secret** UI prefs only | Never store keys, tokens, or fragments here ([17](17-security.md)). |
| Persistence durability | **`navigator.storage.persist()`** | Request persistent storage so the OS doesn't silently evict the local subset/keys ([16 §16.5](16-offline-and-pwa.md)). |

## 2.5 Networking & realtime

| Concern | Choice | Rationale |
|---------|--------|-----------|
| HTTP | **`fetch`** wrapped in a typed `ApiClient` (interceptors for auth, retry, idempotency) | No heavy HTTP dep; `/api/v1` over ciphertext ([05](05-api-client.md)). |
| Realtime | **`@microsoft/signalr`** (JS client) | The server relay is SignalR `RelayHub` ([09](09-realtime-collaboration.md)); official browser client, binary protocol for `EncryptedUpdate`. |
| Auth | **fetch + WebAuthn API** (default); **oidc-client-ts** (enterprise) | **Default native auth**: `fetch` posts password + TOTP to the Nyxite server API, and the browser **WebAuthn API** handles passkeys. **oidc-client-ts** is only for the **enterprise Keycloak option** (Authorization Code + **PKCE**, silent refresh) ([14](14-authentication.md)). |

## 2.6 Cryptography (must match the server's algorithm table exactly)

All primitives must match the server's ledger ([server 07 §7.3](https://github.com/Nyxite/NyxiteServer); reproduced in [06 §6.2](06-cryptography.md)). Wrap **everything** behind the `CryptoEngine` interface so a library can be swapped without touching repositories.

| Purpose | Library | Notes |
|---------|---------|-------|
| AES-256-GCM (content / CRDT / snapshots / names / recovery blob) | **WebCrypto `SubtleCrypto`** (`AES-GCM`) | Hardware-accelerated; keys held as **non-extractable** `CryptoKey` where possible. 96-bit nonce, 128-bit tag. |
| Hybrid HPKE wrap (file-key to a member; device enrollment to a device pubkey) | **hpke-js** (`@hpke/core`) **+ a WASM PQC KEM** | v1 KEM is the **hybrid `X25519MLKEM768`** — classical DhkemX25519HkdfSha256 (RFC 9180 `KEM 0x0020`) concatenated with **ML-KEM-768** — with HkdfSha256 (`KDF 0x0001`) / Aes256Gcm (`AEAD 0x0002`); NIST level 3. **WebCrypto ships no ML-KEM**, so the PQC half needs a WASM PQC lib (follow-up, [06 §6.13](06-cryptography.md)); conformance-test against server/desktop/Android ([06 §6.4](06-cryptography.md)). |
| X25519 + Ed25519 (classical halves of the hybrid suites) | **libsodium-wrappers-sumo** (WASM) | Consistent X25519/Ed25519 across all browsers (WebCrypto Ed25519/X25519 support is uneven); also a fallback HPKE building block. |
| ML-KEM-768 + ML-DSA-65 (PQC halves — hybrid) **[P]** | **audited WASM PQC library** (choice TBD) | Neither WebCrypto nor libsodium exposes ML-KEM/ML-DSA; a dedicated WASM PQC lib supplies the post-quantum halves of the hybrid HPKE KEM and hybrid signatures, byte-compatible with server/desktop/Android. Self-hosted WASM, behind `CryptoEngine`, gated on the shared vectors ([06 §6.2](06-cryptography.md), [06 §6.13](06-cryptography.md)). |
| BLAKE3-256 (content addressing) — *unchanged, quantum-safe* | **hash-wasm** (`blake3`) | Streamed hashing of large blobs; test vectors verified. |
| Argon2id (recovery-key derivation) — *unchanged, quantum-safe; no PQC/pepper* | **hash-wasm** (`argon2id`) | Params from `recovery_blobs.kdf_params` (m=64 MiB, t=3, p=1 floor; tune — [19 §19.3](19-open-questions.md)); run in the crypto worker. |
| CSPRNG | **`crypto.getRandomValues`** | FK and nonce generation. |

> Crypto-agility: the encrypted frame carries `version` and `key_id`/generation, so primitives can rotate without a format break ([06 §6.3](06-cryptography.md)). The WASM crypto runs in a **Web Worker** ([01 §1.6](01-architecture.md)). The **asymmetric** primitives ship **post-quantum hybrid at v1.0.0** — X25519+ML-KEM-768 key wrap and Ed25519+ML-DSA-65 signatures, NIST level 3; symmetric primitives (AES-256-GCM, Argon2id, BLAKE3) are unchanged ([06 §6.2](06-cryptography.md)).

## 2.7 CRDT

| Concern | Choice | Risk |
|---------|--------|------|
| Text CRDT | **Yjs** (+ **y-protocols** for awareness, **y-codemirror.next** for the editor binding) | **Lowest-risk binding of the three clients** — Yjs is the reference, most battle-tested Yrs-family implementation. The wire protocol is pinned via the shared conformance vectors against ydotnet (desktop) and yrs/UniFFI (Android) ([09 §9.12](09-realtime-collaboration.md), [18](18-build-ci-testing.md)). Updates are **encrypted before they leave** ([06](06-cryptography.md)). |

## 2.8 Ink / stylus (browser-constrained)

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Capture | **Pointer Events API** on `<canvas>` | `pointermove` with `pressure`, `tiltX`/`tiltY`, `getCoalescedEvents()` for sub-frame samples; `pointerType` for palm rejection. |
| Rendering | Canvas 2D (optionally low-latency `desynchronized` context) | Wet-ink rendering; commit smoothed strokes ([10 §10.4](10-editors.md)). |
| Storage format | **Deterministic CBOR** (`cbor-x` in canonical mode) | The shared, byte-stable Nyxite ink vector format, co-designed with desktop/Android so files round-trip and the BLAKE3 address is reproducible ([10 §10.5](10-editors.md)). |

## 2.9 PWA / offline

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Service worker | **Workbox** (precache app shell, runtime routing) | App-shell offline; **no content decryption in the SW** ([16](16-offline-and-pwa.md), [17](17-security.md)). |
| Manifest | Web App Manifest (installable, icons, share_target opt) | Install to home screen / desktop ([16 §16.1](16-offline-and-pwa.md)). |

## 2.10 Diff, search, utilities

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Text diff (history) | **`diff`** (jsdiff) line/word; Yjs structural where richer | Client-side version diffs ([12 §12.4](12-version-history.md)). |
| Full-text search | **MiniSearch** (or FlexSearch) over decrypted local subset, persisted to IndexedDB | In-browser FTS; index never uploaded ([11](11-search.md)). |
| Dates/ids | `uuidv7` generator; `Intl`/`date-fns` for display | UUIDv7 client-allocated ids ([05](05-api-client.md)). |

## 2.11 Observability & quality

| Concern | Choice |
|---------|--------|
| Logging | A thin `Logger` facade with **content/key scrubbing** — never log plaintext, keys, tokens, or URL fragments ([17](17-security.md)). |
| Crash/analytics | **None by default.** No third-party content-touching or analytics SDKs; any opt-in error reporting must be self-hosted and PII/secret-scrubbed. **[P]** |
| Testing | **Vitest** + **React Testing Library** (unit/component), **Playwright** (e2e), plus **crypto + CRDT conformance vectors** ([18](18-build-ci-testing.md)). |
| Static analysis | **ESLint** (+ **eslint-plugin-boundaries** for the crypto/layer boundary, [01 §1.2](01-architecture.md)) + **Prettier** + `tsc --noEmit`. |

## 2.12 Dependency policy

- Every dependency is reviewed for maintenance, license compatibility (PolyForm-noncommercial app; dependencies must be redistributable), and whether it could exfiltrate content. A dep that loads remote code at runtime is rejected.
- **No analytics/ad/attribution/CDN-injected SDKs.** No dependency that phones home by default. The **CSP forbids third-party script/connect origins** beyond the configured instance ([17](17-security.md)).
- Pin versions and integrity hashes (lockfile + Subresource Integrity for any external asset, though the app self-hosts all code); enable dependency auditing and supply-chain scanning in CI ([18](18-build-ci-testing.md)).
