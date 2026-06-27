# 00 — Overview

## 0.1 Purpose

The Nyxite Web client is a **browser-based, end-to-end-encrypted notes and documents app**. It lets a user, from any modern browser with no install:

- Browse the project → folder → file hierarchy (names decrypted in the browser).
- Read and edit **markdown**, **plain text**, and **handwritten ink** files, with distinct **view** and **edit** modes.
- Sync content across devices with per-file policies, **encrypting before upload and decrypting after download** — the server only moves ciphertext.
- Collaborate live with other users and anonymous guests via a **client-side CRDT merge** (Yjs) over an encrypted relay.
- Search its **local subset** of decrypted content.
- Browse **version history** with client-computed diffs, and restore.
- Create and consume **share links** (key in the URL fragment) and **account shares** (file key wrapped to a recipient's public key via HPKE).

The web client is the **primary surface for anonymous guest access**: a guest opens a share link in a browser with no account, and the decryption key comes from the link's URL fragment, never the server. It is also a full account client (login, editing, sharing, history).

## 0.2 What makes this client different from a normal web app

Because Nyxite is zero-knowledge, the browser carries responsibilities a typical web client offloads to a backend:

- It **generates and holds the user's keys**; it performs **all** AES-256-GCM encryption/decryption and HPKE wrap/unwrap locally (WebCrypto + audited WASM).
- It **computes content addresses** (BLAKE3 of plaintext) itself; the server stores ciphertext under the client-supplied address and cannot verify it.
- It **runs the CRDT engine** (Yjs) and merges locally; the server never merges.
- It **builds and queries its own full-text search index** over what it has decrypted; there is no server search.
- It **computes diffs** between decrypted snapshots; there is no server diff.
- It **drives key rotation** on share revocation for forward secrecy.

The server gives it: authenticated transport, durable ordered storage of ciphertext, an encrypted relay, an ACL gate, a key directory (public keys), and structure metadata. Everything cryptographic is the browser's job. See [06-cryptography.md](06-cryptography.md) and [07-key-and-device-management.md](07-key-and-device-management.md).

## 0.3 The browser is the most constrained peer (accepted, by design)

The web client is deliberately the weakest of the three on reach, because a browser:

- **Cannot hold the whole corpus.** Search and offline are bounded by Storage-API quota and eviction; it indexes/caches a **local subset** (kept-in-browser + recently opened), not everything. Desktop is the full-corpus search surface ([11](11-search.md), [16](16-offline-and-pwa.md)).
- **Has a more hostile execution environment.** XSS, extensions, and a generally untrusted client mean key material and plaintext at rest in IndexedDB get a dedicated threat model ([17](17-security.md)).
- **Cannot use the platform keystore.** There is no Keystore/StrongBox equivalent; keys live as **non-extractable WebCrypto `CryptoKey` handles in IndexedDB**, session- or persistence-scoped ([07](07-key-and-device-management.md)).
- **Has no native stylus pipeline.** Ink uses Pointer Events (`pressure`, `tiltX/Y`), which capture less fidelity than S-Pen — web ink is **view + basic editing**, not the showcase ([10](10-editors.md)).

These are the direct, accepted consequences of privacy-first; the spec degrades gracefully rather than weakening encryption.

## 0.4 Scope of v1.0.0

v1.0.0 is the **complete E2EE web client** spanning Phases 0–6 of the master roadmap, built in the same phase order as the server (see [20-roadmap.md](20-roadmap.md)). The key/identity/recovery subsystem is foundational (Phase 0) and is never retrofitted later.

Explicitly **in scope**: markdown + plaintext + ink editing/viewing, the server sync policies (`server-default`/`excluded`) plus client-local keep-in-browser, encrypted relay collaboration with guests, account + link sharing, rotation-based revocation, version history with client diffs/restore, in-browser search over the local subset, Keycloak login with TOTP, identity-key handling and the user-held recovery-key flow in the browser, installable PWA with offline support, and **multiple accounts / instance switching** ([14 §14.7](14-authentication.md)).

Explicitly **out of scope for v1.0.0** (deferred, matching server Phase 5–6 and the separate migration item): office-document and source-code content types, image attachments, chunked upload for very large binaries, key-transparency/safety-number verification, metadata-graph hiding, handwriting recognition, and Samsung Notes `.sdoc` import. Seams are left where they touch the architecture.

## 0.5 Deployment & rendering model (ratified)

- **Static-export SPA.** The app is built with Next.js (App Router) and exported to **static assets** (`output: 'export'`) served from a CDN/static host. There is **no Next.js server runtime, no SSR of content, no server components touching data** — the strongest zero-knowledge posture and the simplest thing to self-host alongside the API ([01](01-architecture.md), [18](18-build-ci-testing.md)).
- **Everything is client-side.** Routing, data fetching, crypto, CRDT merge, search and rendering run in the browser. Dynamic routes that depend on runtime values (e.g. `/share/{token}`) are served by a single statically-exported shell rendered fully client-side ([15 §15.1](15-ui-and-navigation.md)).
- **Instance configuration.** The static bundle is instance-agnostic; the API base (`/api/v1`), OIDC authority, relay hub URL, and public-share base are resolved at runtime from a small `config.json` / environment (per-account override for multi-instance) ([14](14-authentication.md), [19](19-open-questions.md)).

## 0.6 Target browsers & platform baseline

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Browsers | **Evergreen Chromium (Chrome/Edge), Firefox, Safari** — last 2 major versions | Need WebCrypto, IndexedDB, Service Workers, WebAssembly, Pointer Events, WebSockets. **[P]** |
| Secure context | **HTTPS only** (or `localhost` in dev) | WebCrypto, Service Workers, and clipboard require a secure context ([17](17-security.md)). |
| WASM | Required | BLAKE3 + Argon2id run as WASM ([06](06-cryptography.md)). |
| Form factors | Desktop, tablet, phone; responsive | shadcn/Tailwind responsive layout; list/detail on wide screens ([15](15-ui-and-navigation.md)). |
| Stylus | Pointer Events (`pressure`, `tiltX/tiltY`) where the device/browser exposes them | Browser ink fidelity is below native; view + basic editing only ([10](10-editors.md)). |
| Offline | First-class via PWA | Installable, service-worker app shell, IndexedDB-persisted subset ([16](16-offline-and-pwa.md)). |

Browsers without the required APIs (e.g. WebCrypto absent, third-party-storage partitioning that blocks IndexedDB) get a clear unsupported-browser screen rather than a degraded, insecure path.

## 0.7 Actors (as seen by the web client)

| Actor | On Web |
|-------|--------|
| **User** | Signs in via Keycloak (OIDC + PKCE + TOTP) in the browser, establishes/holds the identity keypair, owns/edits/shares files. |
| **Guest** | This browser acting on a link share: **no Keycloak account**, file key taken from the URL fragment, relay access via a short-lived share-session token. The app both *opens* incoming links and *creates* link shares. The web client is the primary guest surface. |
| **Peer** | Another user/guest editing the same document; seen through presence/awareness over the relay. |
| **Server** | Blind relay/store. The client treats every byte it sends as ciphertext the server cannot read. |
| **Keycloak** | External IdP for account auth and TOTP. |

## 0.8 Glossary (client-facing)

- **FK (file key)** — per-file AES-256-GCM 256-bit key, generated in-browser, stored on the server only *wrapped*.
- **Identity keypair** — per-user X25519 (HPKE/key-agreement) + Ed25519 (signing); private parts never leave the browser/account session.
- **Device** — in the web client, a *browser profile* counts as a device for enrollment; its enrollment keypair approves the session's access to the identity private key ([07](07-key-and-device-management.md)).
- **Recovery key** — high-entropy user-held secret (BIP39 phrase) that, via Argon2id, wraps an escrow of the identity private key (AES-256-GCM); the only recovery path.
- **Wrapped key** — an FK encrypted to a member's X25519 public key via HPKE (account share).
- **Fragment key** — an FK carried in a share link's URL fragment (`#k=…`), never sent to the server (link/guest share).
- **Content address** — BLAKE3-256 hash of the *plaintext*, used as the blob's storage key; computed in-browser.
- **Encrypted frame** — the on-the-wire/at-rest container: `magic(4)|version(1)|key_id(16)|nonce(12)|ciphertext|tag(16)` with AAD binding `file_id` + `object_kind` ([06](06-cryptography.md)).
- **Key generation** — integer that bumps on FK rotation; clients must use the current generation.
- **Sync policy** — per-file server policy: `server-default` | `excluded`. (Offline pinning is the separate **client-local** `keepInBrowser` field, never sent to the server — [16 §16.2](16-offline-and-pwa.md).)
- **CRDT / Yjs** — the Yjs (Yrs-family) CRDT used to merge text documents in the browser; the **reference** implementation of the wire protocol the other clients also speak.
- **Local subset** — the files this browser has decrypted/cached (kept-in-browser + recently opened); the only thing it can search offline.

## 0.9 Phase map (Web)

| Phase | Web deliverable |
|-------|-----------------|
| 0 | App shell (static-export SPA + PWA install), Keycloak login + TOTP, browser enrollment, identity keys in IndexedDB (non-extractable), recovery-key UX, structure browsing with encrypted names, local encrypted store. |
| 1 | Markdown + plaintext editing on the encrypted Yjs document (single user, offline-capable), blob sync, server sync policies (server-default/excluded) + client-local keep-in-browser, on-demand download, view/edit modes, in-browser search over the local subset. |
| 2 | Live relay collaboration (client-side merge), account + link sharing, **guest mode (primary surface)**, rotation-based revocation, version history with client diffs + restore. |
| 3 | Pointer-events ink capture + the shared vector stroke format, LWW/version-vector ink sync (encrypted blobs); ink viewing first, basic editing. |
| 4 | Rich per-user/per-file config, settings, audit surfacing where applicable, polish, PWA hardening. |
| 5 | (Optional) office/source/image content types as encrypted blobs; chunked upload. |
| 6 | (Optional) key-transparency/safety-number verification UI. |

See [20-roadmap.md](20-roadmap.md) for detail and acceptance criteria.
