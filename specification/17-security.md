# 17 — Security (Browser-Side)

The server-side model is [server 13](https://github.com/Nyxite/NyxiteServer) (zero-knowledge: DB/blob theft, malicious operator, curious admin all yield no readable content). This document covers the **browser-side** threats the web client must defend. The browser is a **more hostile environment than a native app**: it runs untrusted page context, hosts extensions, may be a shared/public machine, has no platform keystore, and carries the unique fragment-key-in-URL exposure. This mirrors the Android device-side model ([android 17](https://github.com/Nyxite/NyxiteAndroid)) but recentres it on the web threats.

## 17.1 Posture recap

- **Zero-knowledge / full E2EE**: plaintext and keys exist only in the browser; the server sees ciphertext, opaque IDs, the structure graph, ACL grants, wrapped-key blobs, sizes, and timestamps ([server 13 §13.1](https://github.com/Nyxite/NyxiteServer)).
- **The cardinal crypto boundary** ([01 §1.2](01-architecture.md)): exactly one place (`CryptoEngine`, in a Web Worker) turns plaintext ↔ ciphertext; network clients handle only opaque bytes and structural JSON, enforced by an ESLint boundary rule.

## 17.2 Threat model

| Threat | Defended by | Residual / boundary |
|--------|-------------|---------------------|
| Server / operator compromise | **E2EE** — no content key ever on the server | Metadata (structure, sizes, timestamps) visible |
| Network attacker | TLS + only ciphertext on the wire; BLAKE3 verify on download, hybrid Ed25519+ML-DSA-65 verify on directory/updates | Traffic analysis on sizes/timing |
| Harvest-now-decrypt-later (future quantum) | Asymmetric seams are **hybrid X25519+ML-KEM-768 / Ed25519+ML-DSA-65** from v1.0.0 ([06 §6.2](06-cryptography.md)), so stored wrapped keys resist a future quantum adversary | Classical-only ciphertext never shipped |
| **XSS (top web threat)** | Strict CSP, Trusted Types, sanitized rendering, no inline/eval, self-hosted code (§17.3) | A script-execution bug in-page can read an unlocked session — minimized, not eliminated |
| Malicious browser extension | Same-origin isolation; least surface; cannot read non-extractable key bytes | A content-script-capable extension can read page DOM/plaintext — out of the app's control |
| Shared / public machine | Session-only mode (no persistence), idle auto-lock, explicit logout/clear | A keylogger/compromised OS defeats this — advise against use on untrusted machines |
| Local storage theft (disk image) | Identity bundle wrapped under a **non-extractable** vault key; optional unlock + auto-lock (§17.5) | A persisted unlocked session is readable; mitigated by session-only/auto-lock |
| Fragment-key leakage | Capture-then-strip, `Referrer-Policy: no-referrer`, never logged (§17.6) | Browser history / synced history retains the URL the user pasted |

## 17.3 XSS defenses (primary)

XSS is the **top** web threat: any script running in the page runs with full access to the unlocked session. Layered defenses:

- **Strict Content-Security-Policy** (delivered via meta + host headers): `default-src 'self'`; **`script-src 'self'`** (no `'unsafe-inline'`, no `'unsafe-eval'`); `connect-src` limited to the **configured instance** API/relay origins (plus the OIDC authority under the enterprise Keycloak option) only; `frame-ancestors 'none'`; `object-src 'none'`; `base-uri 'self'`; `style-src 'self'` (+ nonces if needed); `img-src 'self' data:` (no remote fetch from notes).
- **Trusted Types [P]** (`require-trusted-types-for 'script'`) to choke DOM-XSS sinks; the only sink allowed is the markdown sanitizer's output policy.
- **Sanitize all rendered markdown** via **rehype-sanitize** with a strict schema ([02 §2.2](02-tech-stack-and-libraries.md), [10 §10.2](10-editors.md)); **no remote content fetch** from notes (no remote images/scripts/iframes).
- **Ban `dangerouslySetInnerHTML`** outside the single audited sanitizer wrapper (lint rule).
- **Fully self-hosted code, no third-party script/CDN** ([02 §2.12](02-tech-stack-and-libraries.md)); SRI on any external asset (there should be none). This keeps the CSP allow-list tight and removes remote-code attack surface.

## 17.4 At-rest in the browser

Cached plaintext (the local subset, [16](16-offline-and-pwa.md)) and wrapped key material live in **IndexedDB**; ciphertext blobs in Cache Storage. The browser has no platform keystore, so the analogue is built from WebCrypto:

- The **identity bundle** (private X25519+ML-KEM-768 / Ed25519+ML-DSA-65 hybrid keys) is wrapped under a **non-extractable WebCrypto vault `CryptoKey`** stored in IndexedDB ([07](07-key-and-device-management.md)) — its private bits cannot be read back out as bytes by page script.
- **Optional unlock** gates the vault key: **WebAuthn passkey preferred**, or a passphrase (Argon2id-derived, [06](06-cryptography.md)); plus **idle auto-lock** that drops the in-memory unwrapped keys and decrypted state.
- **Session-only vs. persistent** ("remember this browser") is a user choice: session-only keeps nothing across tabs-closed; persistent requests durable storage but requires unlock on return.
- **File-key handles** are kept as non-extractable `CryptoKey`s where the algorithm allows; **per-account isolation** (separate IndexedDB DB, blob partition, index per `accountId`, [01 §1.8](01-architecture.md)) means one account's compromise can't read another's.
- Request **persistent storage** ([16 §16.5](16-offline-and-pwa.md)) but treat the device as **semi-trusted**.
- **Honest limit**: a fully compromised browser/OS, or a logged-in **unlocked** session, can read content — true of any client. The controls raise the bar (non-extractable keys, optional unlock, auto-lock, session-only) but cannot defeat a privileged in-context attacker.

## 17.5 Fragment-key handling

The share-link file key arrives in `location.hash` ([13](13-sharing.md), [server 09 §9.4](https://github.com/Nyxite/NyxiteServer)):

- **Capture to memory on first paint**, then **`history.replaceState`** to strip `#k=…` from the visible URL/history entry.
- Serve the app with **`Referrer-Policy: no-referrer`** so the fragment never leaks via `Referer`.
- **Never log it**, never put it in storage, never include it in any network request (it is for client-side decrypt only).
- **Copy-link UX warning**: the share-create flow warns that the link *is* the secret, and notes the **browser-history / history-sync retention risk** for anyone who pastes it ([13](13-sharing.md)).

## 17.6 Token handling

- **Access token in memory only** (a module-scoped variable inside the session), never `localStorage`/`sessionStorage` ([14](14-authentication.md)).
- **Cautious refresh** of the server's access token (native refresh-token grant, or oidc-client-ts silent refresh under the enterprise Keycloak option); the relay socket ticket and guest share-session token are short-lived ([server 13 §13.4](https://github.com/Nyxite/NyxiteServer)).
- **Never** place tokens, keys, or fragments in `localStorage`/`sessionStorage`/URL/telemetry. **Logout** clears the in-memory session and tears down the `UserSession`.

## 17.7 Service worker

- The SW is **scoped** (`/`), precached with revision hashes (integrity-checked), and caches **only** static assets and immutable content-addressed ciphertext ([16 §16.2](16-offline-and-pwa.md)).
- It **does not handle plaintext or keys**, has no reference to the crypto worker or key vault, and runs in a separate, less-trusted context.
- **Cache-poisoning considerations**: ciphertext is cached **by BLAKE3 content address**, so a substituted blob fails the post-decrypt integrity check ([06](06-cryptography.md)); precache entries are revisioned; the SW never caches authed API responses.

## 17.8 Logging & telemetry

A scrubbing `Logger` facade ([02 §2.11](02-tech-stack-and-libraries.md)): never log plaintext, keys, tokens, fragments, recovery phrases, or full share URLs (ciphertext sizes/IDs are acceptable). **No third-party analytics** or content-touching SDKs; any opt-in error reporting is self-hosted and secret-scrubbed.

## 17.9 Other controls & honest limits

- **Clickjacking**: `frame-ancestors 'none'` (and `X-Frame-Options: DENY`).
- **Clipboard caution**: warn on copying secrets/links; do not auto-copy keys.
- **Secure-context requirement**: WebCrypto, service workers, and clipboard require HTTPS/`localhost`; the unsupported-browser screen blocks insecure contexts ([15 §15.9](15-ui-and-navigation.md), [00 §0.6](00-overview.md)).
- **Cannot prevent screenshots / screen capture** in a browser — there is no `FLAG_SECURE` equivalent; this is a documented limitation.
- **Zeroization is best-effort** in JS: garbage collection and immutable strings mean key bytes can't be reliably wiped; prefer non-extractable `CryptoKey` handles (never materialized as JS bytes) and drop references on lock to minimize exposure.
- **Rate-limit cooperation**: honor server `429 rate_limited` + `Retry-After` ([05](05-api-client.md), [server 13 §13.5](https://github.com/Nyxite/NyxiteServer)); don't hammer auth/key-directory/share endpoints.
- **Dependency / supply chain**: every dep runs with full access to plaintext and keys — pinned lockfile + integrity hashes, no remote-code-loading deps, dependency audit and supply-chain scan in CI ([02 §2.12](02-tech-stack-and-libraries.md), [18](18-build-ci-testing.md)).

## 17.10 Support plane — the one consensual non-E2EE exception

In-app bug reporting ([15 §15.10](15-ui-and-navigation.md)) runs on the project's **single, deliberate exception to zero-knowledge**: a **consensual, non-E2EE support plane** that a user enters only by an explicit action, provably **disjoint from the content plane** (SUP-1). It is safe because it does not weaken content E2EE:

- A report carries **no content key and no content-plane ciphertext** — only the free text, the redacted screenshot, and a user-reviewed diagnostic envelope; the content-plane zero-knowledge guarantee is **untouched**. The load-bearing protections here are the explicit non-E2EE + "goes to the Nyxite maintainer" notice + GDPR shown before send, and the user's own redaction — **not** encryption (reports are stored server-readable by the maintainer, SUP-1/SUP-8).
- **Screenshot redaction is destructive and client-side (SUP-2):** black-box + blur are **flattened into the pixels (re-encoded PNG, EXIF stripped) before upload**; the original image and any redaction mask are **never sent** — no peel-back layer. (The support screenshot is the one deliberate exception to the browser's "cannot prevent screen capture" limit above — here the user is capturing their *own* view on purpose.)
- The client **never contacts the helpdesk directly**; it relays through its own `NyxiteServer` as an authenticating relay (SUP-7). Detail: master feature [support.md](https://github.com/Nyxite/Nyxite), [NyxiteSupport `specification/02`](https://github.com/Nyxite/NyxiteSupport), [OPEN-DECISIONS SUP-1–SUP-13](https://github.com/Nyxite/Nyxite).
