# Nyxite Web — Features

Next.js + shadcn/ui client. Primary surface for anonymous guest access.

**Privacy first / full E2EE.** All encryption, decryption, and CRDT merge happen **in the browser** (WebCrypto). The server only ever sees ciphertext. The browser holds the session's keys, decrypts content for display, and re-encrypts edits before they leave. Of the three clients the browser is the **most constrained** — it cannot hold the whole corpus, so its search and offline reach are the weakest.

## Editing

- Markdown, handwritten ink, and plain-text editing and viewing
- View and edit modes

## Organization

- Project and folder navigation (file/folder names are decrypted client-side; the server stores them encrypted)

## Collaboration

- Live collaborative editing — **client-side CRDT merge** (Yjs, the production default); the server is a blind **encrypted relay**, not an authoritative merger
- Anonymous guest access via share links (opens in browser, no account); the **decryption key comes from the link's URL fragment**, never from the server

## Search

- **Client-side** full-text search over content decrypted in the session (the local/cached subset only — a browser cannot index the full corpus); the desktop client is the full-corpus search surface

## Version history

- Version history with **client-side diffs** (two encrypted snapshots are fetched and diffed locally) and restore

## Sharing

- Share link creation and management; for link shares the file key is placed in the URL fragment client-side; for account shares the file key is wrapped to the recipient's public key (HPKE) in the browser

## Encryption & keys

- Browser-side AES-256-GCM content encryption; HPKE wrapping/unwrapping of file keys
- Device/identity key handling in the browser, including recovery via the user's recovery phrase unwrapping the client-encrypted recovery blob (AES-256-GCM under an Argon2id-derived key)

## Authentication

- **Native login** — password + required TOTP, or **passkeys (WebAuthn)** — authenticates the account and returns the server's own token; decryption is governed by the in-browser key material. Enterprise Keycloak/OIDC SSO is a pluggable option. (See [SPECIFICATION §10](../docs/SPECIFICATION.md).)

## Open questions

See [../docs/OPEN-DECISIONS.md](../docs/OPEN-DECISIONS.md). Web-specific:

- In-browser key storage and recovery — where the identity private key lives (IndexedDB / non-extractable WebCrypto keys), session vs persistent, and the recovery-phrase UX in a browser
- WebCrypto coverage and performance for AES-256-GCM + HPKE (X25519) at editing scale
- Guest session model: how anonymous link sessions are issued, scoped, and expired, with the fragment key never reaching the server
- In-browser ink editing (pointer events / canvas) and parity with native clients
- Read-only vs editable guest links and the per-link permission UI
- How much client-side search/index is practical in a browser, and graceful degradation when the corpus exceeds local limits
