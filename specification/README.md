# Nyxite Web — Specification (v1.0.0)

This folder is the detailed, implementation-level specification for the **Nyxite Web client** — the Next.js + React + shadcn/ui application that lets a user read, write, hand-draw, organize, sync, collaborate on, share, and search end-to-end-encrypted notes and documents **from a browser**, and is the **primary surface for anonymous guest access** via share links.

It expands the architectural planning documents in the central [`Nyxite`](https://github.com/Nyxite/Nyxite) repository and the [`server` specification](https://github.com/Nyxite/NyxiteServer) into a concrete client build specification covering the full v1.0.0 product across all roadmap phases. It is the web counterpart to the [`android` specification](https://github.com/Nyxite/NyxiteAndroid).

It is written to be **self-contained enough to build the app from**: it names the libraries, the module graph, the in-browser data stores, the API/relay calls, the crypto primitives and how they map to **WebCrypto / WASM** libraries, the editor and sync internals, the UI surface, and the build/test/CI setup.

## Guiding principle: privacy first (full E2EE), in the browser

**Nyxite is end-to-end encrypted everywhere.** All encryption, decryption, content addressing, CRDT merge, search indexing, and diffing happen **in the browser**. The server only ever sees ciphertext, opaque IDs, the structure graph, ACL grants, wrapped-key blobs, sizes and timestamps — it never holds a content key and cannot read notes, ink, names, or collaboration traffic. The web client is therefore not a thin viewer: it is a **full cryptographic peer** that holds the session's keys, performs all crypto locally (WebCrypto + audited WASM), and treats the server as a blind relay/store.

Of the three clients the browser is the **most constrained**: it cannot hold the whole corpus, so its search and offline reach are the **weakest**. The web client indexes and caches its **local subset** (kept-in-browser + recently opened), not the whole corpus. The desktop remains the full-corpus surface. The browser also carries unique threats (XSS, an untrusted-by-default storage tier, the fragment-key-in-URL exposure) that drive [17-security.md](17-security.md).

## Ratified product decisions (this spec)

| Decision | Choice | Where |
|----------|--------|-------|
| Rendering / deployment | **Static-export SPA** — Next.js App Router exported to static assets (`output: 'export'`), CDN-served; **no Next.js server runtime ever touches plaintext or keys** | [01](01-architecture.md), [02](02-tech-stack-and-libraries.md), [18](18-build-ci-testing.md) |
| Offline model | **Full installable PWA** — service worker app shell + **IndexedDB-persisted decrypted local subset** and key material; works offline as a constrained peer | [16](16-offline-and-pwa.md), [04](04-local-data-model.md) |
| Ink in the browser | **View + basic Pointer-Events editing** (pressure where the device exposes it); shared deterministic-CBOR format with desktop/Android | [10](10-editors.md) |
| Text CRDT engine | **Yjs** — the reference Yrs-family implementation; lowest-risk binding of the three clients | [02](02-tech-stack-and-libraries.md), [09](09-realtime-collaboration.md) |
| Multi-account | **Yes, from v1.0.0** — per-account isolated in-browser store, keys, and index; per-account instance host | [07](07-key-and-device-management.md), [14](14-authentication.md), [04](04-local-data-model.md) |
| Key storage & lock | **Non-extractable WebCrypto vault key** wraps the identity bundle; unlock via **WebAuthn passkey, else passphrase**; idle auto-lock; **persistent by default** | [07 §7.3–7.4](07-key-and-device-management.md), [17](17-security.md), [19 §19.1](19-open-questions.md) |
| Curve25519 library | **libsodium-wrappers-sumo (WASM)** for X25519/Ed25519 (consistent across browsers) | [02 §2.6](02-tech-stack-and-libraries.md), [06](06-cryptography.md), [19 §19.3](19-open-questions.md) |
| Search scope | **MiniSearch over the local subset**; titles for all known files; body index → **title-only fallback** over quota | [11](11-search.md), [19 §19.5](19-open-questions.md) |
| Multi-tab | **Single leader tab** (Web Locks + BroadcastChannel); one relay socket | [09 §9.10](09-realtime-collaboration.md), [19 §19.8](19-open-questions.md) |
| Guest data | **In-memory only** — fragment key + content never persisted | [13](13-sharing.md), [17](17-security.md), [19 §19.4](19-open-questions.md) |

All web-specific open questions are now **ratified** — see [19-open-questions.md](19-open-questions.md) for the full set, each with its validation spike and fallback.

## Source of truth & convention

- The central `Nyxite` repo is authoritative for product decisions; the `server/specification` set is authoritative for the wire protocol, schema, crypto model, and API. This spec **consumes** those and must not contradict them. Where it picks a concrete web-side mechanism the server spec left open, it is marked **[P]** (Proposed), exactly as the server and Android specs use the tag.
- **[OD-n]** references a numbered item in the master `docs/OPEN-DECISIONS.md`.
- Open questions specific to web are tracked in [19-open-questions.md](19-open-questions.md) and link back to the master open-decisions list.
- Every cryptographic and wire value is **pinned to the server's canonical ledger** and locked via the shared conformance vectors ([18](18-build-ci-testing.md)); the build fails on any drift.

## Documents

| # | Document | Covers |
|---|----------|--------|
| 00 | [overview.md](00-overview.md) | Purpose, scope, target browsers, actors, glossary, phase map |
| 01 | [architecture.md](01-architecture.md) | SPA layering, unidirectional data flow, the crypto boundary, workers/threading |
| 02 | [tech-stack-and-libraries.md](02-tech-stack-and-libraries.md) | Concrete library choices with versions and rationale |
| 03 | [project-structure.md](03-project-structure.md) | Next.js App Router layout, module/package graph, naming |
| 04 | [local-data-model.md](04-local-data-model.md) | IndexedDB schema, in-memory session state, sync state, search index |
| 05 | [api-client.md](05-api-client.md) | REST client, DTO mapping, error/retry, idempotency, TLS |
| 06 | [cryptography.md](06-cryptography.md) | AEAD, HPKE, Ed25519/X25519, BLAKE3, Argon2id, framing, WebCrypto/WASM mapping |
| 07 | [key-and-device-management.md](07-key-and-device-management.md) | Identity keypair, device enrollment, recovery key, in-browser key storage |
| 08 | [sync-engine.md](08-sync-engine.md) | Manifest/delta sync, policies, CRDT/LWW split, background sync |
| 09 | [realtime-collaboration.md](09-realtime-collaboration.md) | SignalR relay, Yjs merge, awareness, guests, snapshots |
| 10 | [editors.md](10-editors.md) | Markdown, plaintext, and pointer-events ink editors; view/edit modes |
| 11 | [search.md](11-search.md) | In-browser FTS over the local subset; indexing lifecycle |
| 12 | [version-history.md](12-version-history.md) | Snapshot fetch, client-side diff, restore |
| 13 | [sharing.md](13-sharing.md) | Account shares (HPKE), link shares (URL fragment), guest mode, revocation |
| 14 | [authentication.md](14-authentication.md) | Native auth (password+TOTP, passkeys) default; enterprise Keycloak OIDC + PKCE; token storage, refresh |
| 15 | [ui-and-navigation.md](15-ui-and-navigation.md) | Routes, screens, shadcn/Tailwind, responsive, accessibility |
| 16 | [offline-and-pwa.md](16-offline-and-pwa.md) | PWA install, service worker, IndexedDB-cached subset, keep-in-browser, storage limits |
| 17 | [security.md](17-security.md) | Browser threat model, at-rest in IndexedDB, CSP/XSS, fragment handling |
| 18 | [build-ci-testing.md](18-build-ci-testing.md) | Build/static export, CI, test strategy, crypto/CRDT conformance |
| 19 | [open-questions.md](19-open-questions.md) | Web-specific open items with **recommended resolutions** + validation spikes |
| 20 | [roadmap.md](20-roadmap.md) | Web phase mapping aligned to the server roadmap |

## Status

Specification for a greenfield build. No web code exists yet; the `web` repo currently holds only `FEATURES.md` and `LICENSE.md`. This document set defines what to build.

## License

PolyForm Noncommercial License 1.0.0 — see the repo `LICENSE.md`.
