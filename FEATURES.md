# Nyxite Web — Features

Next.js + shadcn/ui client. Primary surface for anonymous guest access.

**Privacy first / full E2EE.** All encryption, decryption, and CRDT merge happen **in the browser** (WebCrypto). The server only ever sees ciphertext. The browser holds the session's keys, decrypts content for display, and re-encrypts edits before they leave. Of the three clients the browser is the **most constrained** — it cannot hold the whole corpus, so its search and offline reach are the weakest.

## Editing

- Markdown, handwritten ink, and plain-text editing and viewing
- View and edit modes

## Organization

- Project and folder navigation (file/folder names are decrypted client-side; the server stores them encrypted)
- **Trash** — deleted items move to a visible Trash with **one-click restore** for the retention window (default 30 d), then leave the UI; a delete never purges immediately (staged Trash → grace → purge, DL-1–DL-5). A share-recipient's delete removes only their own copy; the owner's delete removes it for everyone. See [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (DL-1–DL-5).

## Collaboration

- Live collaborative editing — **client-side CRDT merge** (Yjs, the production default); the server is a blind **encrypted relay**, not an authoritative merger
- Anonymous guest access via share links (opens in browser, no account); the **decryption key comes from the link's URL fragment**, never from the server

## Search

- **Client-side** full-text search over content decrypted in the session (the local/cached subset only — a browser cannot index the full corpus); the desktop client is the full-corpus search surface

## Version history

- Version history with **client-side diffs** (two encrypted snapshots are fetched and diffed locally) and restore

## Sharing

- Share link creation and management; for link shares the file key is placed in the URL fragment client-side; for account shares the file key is wrapped to the recipient's public key (HPKE) in the browser

## Group sharing (enterprise/family)

- Group management ([features/groups.md](https://github.com/Nyxite/Nyxite/blob/main/features/groups.md)) in the browser worker: generate a group keypair, enroll/remove members, unwrap the group key → DEKs; enrollment **verifies the member's public key against the key-transparency log** before wrapping
- Wrap a file/subtree DEK to a group public key; honor the per-project/folder **reader-group attachment** — auto-wrap new files to the attached group's public key (the enterprise "manager reads all" path)
- Scope-scoped group-key rotation on member removal, with the `412` re-seal flow; honest UI that already-decrypted content can't be recalled
- Recovering the identity key restores group access automatically

## Encryption & keys

- Browser-side AES-256-GCM content encryption; **hybrid HPKE (X25519 + ML-KEM-768)** wrapping/unwrapping of file keys — post-quantum hybrid at v1.0.0
- Device/identity key handling in the browser (hybrid X25519+ML-KEM-768 / Ed25519+ML-DSA-65 keys), including recovery via the user's recovery phrase unwrapping the client-encrypted recovery blob (AES-256-GCM under an Argon2id-derived key — symmetric, unchanged)

## Authentication

- **Native login** — password + required TOTP, or **passkeys (WebAuthn)** — authenticates the account and returns the server's own token; decryption is governed by the in-browser key material. Enterprise Keycloak/OIDC SSO is a pluggable option. (See [SPECIFICATION §10](https://github.com/Nyxite/Nyxite/blob/main/docs/SPECIFICATION.md).)

## Bug reporting & support

- **"Report a bug"** — an in-app report composer, shown when the instance has reporting enabled (a server `support.enabled` capability flag, **on by default** — every instance can file, routing to the maintainer's central desk or, if the operator opted into one, their own desk — SUP-9 / SUP-10–SUP-13).
- **Screenshot capture + destructive redaction** — optionally attach a screenshot (a canvas render of the current view, or a display capture you grant); a redaction editor with **black-box + blur** tools **flattens redacted regions into the pixels before upload**, so the original image and mask never leave the browser (SUP-2).
- **Consent + destination notice** — before sending, a clear notice that, unlike your files, the report is **not end-to-end encrypted** and goes to the **Nyxite maintainer**, plus a GDPR disclosure (SUP-1); a **user-reviewable diagnostic envelope** (app version/build, platform, locale, current screen id — never content, scrubbed logs, connection state) is editable before send.
- **"My tickets"** — track your own reports' status and support replies with in-app notifications; submission goes through the server as an **authenticating relay** (the browser never contacts the helpdesk directly — SUP-3/SUP-7).
- Runs on the **consensual, non-E2EE support plane** — disjoint from content, carrying no content key or content-plane ciphertext. Detailed in [Nyxite Support](https://github.com/Nyxite/Nyxite/blob/main/features/support.md) / the `NyxiteSupport` repo `specification/`; decisions in [OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md) (SUP-1–SUP-13).

## Open questions

See [../docs/OPEN-DECISIONS.md](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md). Web-specific:

- In-browser key storage and recovery — where the identity private key lives (IndexedDB / non-extractable WebCrypto keys), session vs persistent, and the recovery-phrase UX in a browser
- WebCrypto coverage and performance for AES-256-GCM + **hybrid HPKE (X25519 + ML-KEM-768)** at editing scale
- **Selecting an audited WASM PQC library** for the hybrid ML-KEM-768 / ML-DSA-65 halves — WebCrypto exposes no ML-KEM/ML-DSA, so a WASM PQC lib is a required follow-up dependency (kept behind `CryptoEngine`, gated on the shared conformance vectors)
- Guest session model: how anonymous link sessions are issued, scoped, and expired, with the fragment key never reaching the server
- In-browser ink editing (pointer events / canvas) and parity with native clients
- Read-only vs editable guest links and the per-link permission UI
- How much client-side search/index is practical in a browser, and graceful degradation when the corpus exceeds local limits
