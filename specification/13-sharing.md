# 13 — Sharing & Guest Mode

Sharing has the **same two enforcement layers** everywhere ([server 09](https://github.com/Nyxite/server)), and the browser is a full participant in both:

1. **Server ACL** — decides who may *fetch ciphertext* or *join the relay*. Enforced server-side at every checkpoint; **revocation here is instant**. The browser only asks; it cannot grant itself reach.
2. **Cryptographic access** — decides who can *decrypt*, via possession of the file key (FK), delivered either **HPKE-wrapped** to a member's public key or carried in a link's **URL fragment**. The server can neither grant nor read this; all wrap/unwrap happens in the `CryptoEngine` ([06](06-cryptography.md)).

Two share **kinds** — account (`user_grant`) and `link`; two **permissions** — `read` | `write`; three **target scopes** — `file` | `folder` (subtree) | `project`. The API addresses every target uniformly as `targetType` + `targetId` ([05](05-api-client.md)).

The web client is the **primary surface for anonymous guest access** ([00 §0.7](00-overview.md)): it both *creates* link shares and *opens* incoming links with no account. `ShareRepository` orchestrates the REST/ACL side; `CryptoEngine` + `KeyRepository` do the wrapping; `RelayClient` carries the encrypted edits ([01 §1.5](01-architecture.md)).

## 13.1 Account share (user grant)

The sharer grants a named Nyxite user; the FK is wrapped to that user's published public key so only they can unwrap it.

1. **Look up the grantee.** `GET /keys/directory?userId=` or `?email=`; cache as a directory entry ([04](04-local-data-model.md)). **Verify the entry's Ed25519 self-signature** before trusting the X25519 public key ([06 §6.7](06-cryptography.md)); a failed signature aborts the share.
2. **Wrap the FK.** `CryptoEngine.hpkeWrap(fk, granteeX25519Pub)` using exactly DhkemX25519HkdfSha256 / HkdfSha256 / Aes256Gcm ([06 §6.4](06-cryptography.md)).
3. **Create the grant.** `POST /shares { targetType, targetId, kind: "user_grant", granteeId, permission }` plus `POST /files/{id}/keys` carrying the **wrapped blob** (or the combined create-share payload). The server stores only the opaque blob — it never sees the FK.
4. **Grantee unwraps.** The grantee's browser `GET /files/{id}/keys` → `CryptoEngine.hpkeUnwrap(blob, identityX25519Priv)` → decrypts content. No server key exchange occurred.

### Subtree shares (folder / project)

For a `folder` or `project` target the client enumerates the subtree and wraps **each current file-key** to the grantee, in batches:

```ts
// idempotent, partial-success per item
await api.post('/files/keys:batch', {
  grants: chunk.map(f => ({ fileId: f.id, keyId: f.keyId, memberId: granteeId,
                            wrappedKey: engine.hpkeWrap(f.fk, granteePub) }))
});
```

- **Completeness rule:** the subtree is **"fully granted"** only once *every current file-key in the subtree* has a wrapped row for the grantee. Until then the UI shows "granting…" and the client **drains a resumable batch queue** (persisted in `LocalStore`), tolerant of partial success and reconnects ([08](08-sync-engine.md)).
- New files created in the subtree after the grant are wrapped to existing members on creation, preserving the invariant.

## 13.2 Link share (fragment key)

1. Mint a high-entropy link **token** (≥128-bit, CSPRNG); the FK (≥256-bit) is the **fragment key**.
2. Assemble the URL against the account's configured public-share base ([14 §14.7](14-authentication.md)):
   `https://{shareHost}/share/{token}#k=<base64url(FK)>`. Everything after `#` is the **fragment** and is **never sent to the server** by the browser.
3. `POST /shares { targetType, targetId, kind: "link", permission: read|write, expiresAt?, linkTokenHash }` — the server stores only `link_token_hash`, never the token plaintext and never the FK.
4. Offer the URL via the Web Share API / copy-to-clipboard with a **warning**: anyone holding the full link can access the file (per permission) until it expires or is revoked, and *the key lives in the link* ([§13.7](#137-fragment-key-caveats--web-mitigations)).

The full URL (with fragment) **cannot be reconstructed** by the server or app after creation; only re-show it if the user explicitly chose to keep it (stored among non-secret prefs is forbidden — keep only in the in-memory session for that page, or not at all).

## 13.3 Guest mode (primary surface)

A visitor opens a link in any browser with **no Keycloak account and no server key exchange**. The flow:

1. **Shell loads.** The static-export shell serves the `/share/{token}` route as a fully client-rendered optional catch-all ([15 §15.1](15-ui-and-navigation.md)); it boots a **trimmed guest client** (editor only, no full nav).
2. **Capture then strip the fragment.** On first paint, read `location.hash` into **memory only**, then immediately `history.replaceState(null, '', location.pathname)` to **remove the fragment from the visible URL**, address bar, and subsequent history entries — reducing shoulder-surfing, history-sync, and referrer leakage ([17](17-security.md)). The in-memory FK survives the rewrite; it is never persisted ([01 §1.7](01-architecture.md)).
3. **Resolve the token.** `GET /share/{token}` resolves the token → returns share metadata + **ciphertext** (or authorizes the relay) and mints a short-lived **share-session token** (15 min, renewable) for relay/ciphertext reach — *not* a content key ([14 §14.5](14-authentication.md)).
4. **Derive the FK.** `engine.fkFromFragment(hash)` (base64url-decode `#k=`); guests never receive a wrapped key.
5. **Open the document.**
   - **Read** content by downloading ciphertext and decrypting with the fragment FK; markdown renders sanitized, no remote fetch ([10](10-editors.md)).
   - For **text/live** docs, join the relay at `/share/{token}/ws` ([09](09-realtime-collaboration.md)) using the share-session token (a single-use 60 s socket ticket is minted per connection). **Read-only** guests receive `OnUpdate`/`OnAwareness` but get `OnError` if they call `SubmitUpdate`; **write** links may edit.
6. **Guest edits store `author_id = null`** server-side; presence shows an anonymous guest.

Guests run **without any account IndexedDB DB**: decrypted views are held in an **ephemeral in-memory cache only**, never written to an account store, and discarded when the tab closes or the share-session token lapses ([14 §14.5](14-authentication.md), [16](16-offline-and-pwa.md)).

If the visitor *is* signed in, opening a link can optionally **"save to my account"** by creating an account share to themselves (wrap the FK to their own public key, [§13.1](#131-account-share-user-grant)); otherwise it stays a constrained guest session.

## 13.4 Permission resolution

Effective permission for (principal, resource) = **max** over:

- **ownership**,
- **direct grant** on the resource,
- **inherited** folder/project grant, and
- (for guests) the **resolving link's** permission.

The two gates are independent: the **server ACL** uses this maximum to gate ciphertext reach and relay writes, while the **crypto layer** *additionally* requires the right key — having ACL reach without a key yields ciphertext you cannot read, and vice-versa. Admins get **structure/usage only, never content** — no key exists for them and there is no break-glass ([14 §14.5](14-authentication.md), [server 09 §9.5](https://github.com/Nyxite/server)).

## 13.5 Managing shares

- A per-target management panel lists active shares: `GET /shares?targetType={file|folder|project}&targetId={id}` — each row shows kind (grantee or "link"), permission, and expiry.
- Change permission/expiry: `PATCH /shares/{id}`; revoke: `DELETE /shares/{id}` ([§13.6](#136-revocation-two-layer)).
- For account shares, surface the grantee's **key fingerprint** (BLAKE3 over the directory entry's public keys) so cautious users can compare out-of-band. Full key-transparency / safety-number verification is **deferred to Phase 6** ([19](19-open-questions.md)); until then trust is **TLS + Ed25519 self-signature** on the directory entry.

## 13.6 Revocation (two-layer)

1. **Instant ACL cutoff** — `DELETE /shares/{id}` sets `revoked_at`. The removed member/link can no longer fetch ciphertext or join the relay, enforced at **every checkpoint** (fetch, join, submit, periodic re-check). Effective immediately; live guest sessions die within one 15-min renewal cycle.
2. **Cryptographic rotation** — to protect **future** content from a party who already holds the FK, a **remaining member's browser** rotates the key ([07 §7.6](07-key-and-device-management.md)): generate a new FK (new `key_id`, `generation + 1`), re-encrypt the head + write a new snapshot under it, **re-wrap to all remaining members**, then:

```ts
await api.post(`/files/${id}/keys/rotate`,
  { newKeyId, generation, wrappedKeys, newHeadRef });
```

- The server commits **only if** `generation == current + 1`, else **`409`** (another browser's rotation won — refetch and re-drive). In-flight CRDT updates tagged with the **old** `key_id` are accepted **until commit**, then rejected **`412 key_generation_stale`**; the client re-encrypts and re-submits under the new key ([05 §5.4](05-api-client.md)). The file shows the `Rotating` sync state throughout ([08](08-sync-engine.md)).
- **Honest limitation:** content the removed party **already downloaded cannot be un-seen**. Instant relay cutoff + forward-secrecy rotation is the strongest practical E2EE revocation; the UI says so plainly.

## 13.7 Fragment-key caveats & web mitigations

The link **is** the secret — anyone with the full URL can decrypt. Browser-specific mitigations, layered:

| Risk | Mitigation |
|------|------------|
| Brute-force / enumeration | Token ≥128-bit, FK ≥256-bit, CSPRNG; server **rate-limits** `/share/{token}` access ([05](05-api-client.md)). |
| Unbounded exposure | `expires_at` (encourage short expiries) + instant revocation ([§13.6](#136-revocation-two-layer)). |
| Address bar / shoulder / history-sync | `history.replaceState` strips the fragment on first paint ([§13.3](#133-guest-mode-primary-surface)). |
| Referrer leakage | `Referrer-Policy: no-referrer` on the app; sanitized markdown does no remote fetch ([17](17-security.md)). |
| Logs / telemetry | The fragment is **never logged**; the `Logger` scrubs `#k=`; no analytics SDKs ([02 §2.11](02-tech-stack-and-libraries.md)). |
| Naive resharing | Copy-link UX **warns** the key is in the link; prefer account shares for sensitive material. |

**Residual, documented:** the browser's own **history, session restore, or profile sync may retain the original URL** *before* `replaceState` runs (e.g. if the page is bookmarked or sync captured the navigation). This is inherent to fragment delivery and called out in [17](17-security.md); account shares avoid it entirely.

## 13.8 Audit

Share creation, permission changes, revocations, and **key rotations** are audited **server-side** with target/actor — **never content or keys** ([server 09 §9.7](https://github.com/Nyxite/server)). Link **access** by guests is auditable by **token hash** (+ IP/UA), never the fragment key. The client surfaces nothing the server could not already log.
