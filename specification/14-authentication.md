# 14 — Authentication

Account authentication is **Nyxite-native and server-owned by default**: the user signs in with **password + required TOTP** or with a **passkey (WebAuthn)** directly against the Nyxite server API over TLS, and the **server issues its own access + refresh tokens**. An **enterprise** deployment may instead plug in **Keycloak OIDC** (Authorization Code + PKCE), which resolves to the **same internal server token**; the client is IdP-agnostic and simply holds the server's bearer token. Login yields an **API token but no content key** — decryption is governed entirely by the in-browser identity keys ([07](07-key-and-device-management.md)). The two layers are deliberately separate ([SPECIFICATION §10](https://github.com/Nyxite/Nyxite/blob/main/docs/SPECIFICATION.md), [server 08](https://github.com/Nyxite/NyxiteServer)): authenticating the *account* never hands the server anything that can decrypt content, and the **login password never feeds content-key derivation**. `AuthManager` owns tokens and silent refresh; it has no reference to the `CryptoEngine` or key material ([01 §1.5](01-architecture.md)). This model follows [OPEN-DECISIONS #9](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md).

## 14.1 Identity provider

- **Native auth is the default IdP** ([SPECIFICATION §10](https://github.com/Nyxite/Nyxite/blob/main/docs/SPECIFICATION.md)), owned by the Nyxite server. Two **co-equal** primary methods from day one: (a) **password with a required TOTP** second factor, and (b) **passkeys (WebAuthn)**, phishing-resistant and sufficient on their own. The server verifies the password against an **Argon2id** verifier and verifies TOTP itself; registration, password reset, and TOTP enrollment/challenge are server-native flows. The server holds **no content-derivable secrets** — a password verifier or passkey credential unlocks an API token, never a content key.
- **Keycloak/OIDC is the enterprise pluggable option**, not the default. Where an enterprise enables it, Keycloak handles credentials and TOTP in its hosted pages and the server accepts that IdP behind the **same internal token**. Native auth and Keycloak both resolve to one server-issued token; the rest of the API never sees which IdP was used.
- **License-gated (L-3).** SSO/OIDC is a **licensed enterprise feature**; a **community-mode** server does not advertise it, so the client offers native login only, and a lapsed license stops new OIDC logins. The client is unchanged — it simply renders whatever login methods the server advertises (server [16 §16.5](https://github.com/Nyxite/NyxiteServer), [08](https://github.com/Nyxite/NyxiteServer)).
- **Account auth ≠ content access.** Login produces an access token for `/api/v1`, **never a content key**. Content keys derive from the user's **identity private key** held in this browser/account session ([07](07-key-and-device-management.md)).
- **Account recovery ≠ content recovery.** Forgot-password / email reset restores **login only**. Content still requires the **recovery phrase or an enrolled device** ([07](07-key-and-device-management.md)), so an account or email compromise yields no content.

## 14.2 Browser auth flow

**Default — native auth (direct to the Nyxite server API over TLS):**

- **Password + TOTP:** the app renders a native login form and `POST`s the credentials (password + the current TOTP code) to the Nyxite server's auth endpoint via `fetch`. The server verifies the Argon2id password verifier and the TOTP code and returns **access + refresh tokens**. The static-export shell sends the password only transiently over TLS and retains no copy.
- **Passkey (WebAuthn):** the app invokes the browser **WebAuthn API** (`navigator.credentials.get`) against a server-issued challenge; the server verifies the assertion and returns the same **access + refresh tokens**. A passkey is sufficient on its own (no separate TOTP step).
- Either native path yields the server's tokens; the app reads the user's identity (`sub`, name/email, roles `user`/`admin`) and the 2FA signal from the server's token/claims.

**Enterprise option — Keycloak OIDC (Authorization Code + PKCE via oidc-client-ts):**

- Where an instance is configured for Keycloak, use **oidc-client-ts** against the realm. Config (per account, runtime-resolved — [§14.7](#147-multi-account--instance-switching)): `authority` (issuer URL), `client_id`, `redirect_uri` (the `/auth/callback` static route), `scope` (`openid profile email` + any Nyxite scope), `response_type=code`.
- Redirect the **top-level browser** to Keycloak (not an iframe) for the authorization request; **TOTP is performed at Keycloak** as part of its flow. Keycloak redirects back to the **statically-exported callback route** (`/auth/callback`); oidc-client-ts exchanges the code (with the PKCE verifier) for tokens, which the server resolves to its **internal access + refresh tokens**.

After a successful login the app checks device enrollment + identity-key availability ([07](07-key-and-device-management.md)): new account → generate identity keypair, enroll this browser, set recovery key; existing account on a new browser → device approval or recovery-phrase unwrap; already enrolled → unlock the identity key into the session. Until the identity key is present the app can show structure (encrypted names unreadable) but cannot decrypt content.

## 14.3 Token model

| Token | Lifetime | Source / use |
|-------|----------|--------------|
| Server **access** | **~5 min** | Bearer for `/api/v1`; refreshed routinely ([§14.4](#144-token-storage--refresh)). |
| Server **refresh** | server-configured | Drives silent renewal of the access token. |
| Relay **socket ticket** | **single-use, 60 s** | Minted from the bearer to authorize a relay upgrade ([09](09-realtime-collaboration.md)). |
| Guest **share-session token** | **15 min, renewable** while the share is valid | Authorizes relay/ciphertext **reach** for link guests — *not* a content key ([13 §13.3](13-sharing.md)). |

The server issues its own access + refresh tokens regardless of which IdP authenticated the user (native or enterprise Keycloak); the client holds an opaque bearer and need not know the IdP.

- **Silent renewal:** `AuthManager` refreshes the access token **before expiry** (native refresh-token grant against the server, or — for the enterprise Keycloak path — oidc-client-ts's refresh-token grant / silent `prompt=none` request). `ApiClient` attaches the bearer; on `401 token_expired` it performs a **single** silent refresh and retries, surfacing re-login only if that fails ([05 §5.4](05-api-client.md)).
- **2FA gating:** the 2FA signal may be required for sensitive **account/admin** actions; otherwise the API returns **`403 2fa_required`** and the app routes the user back through a step-up (a native TOTP/passkey re-prompt, or a Keycloak step-up under the enterprise option). There is **no content break-glass** to gate — content is unreadable to the server regardless.

## 14.4 Token storage [P]

The browser is a hostile environment for secrets ([17](17-security.md)). Therefore:

- **Prefer in-memory access tokens** managed by `AuthManager` (and, under the enterprise Keycloak path, oidc-client-ts), with refresh handled in-session — **not `localStorage`/`sessionStorage`**, to limit XSS token theft. Tokens live in the account session and are gone on tab close.
- **If refresh tokens must persist** (to survive reloads without a full re-login), treat them with the **same caution as key material** and document the trade-off explicitly: the XSS exposure window grows, mitigated by a strict CSP, short access-token TTL, and instant server-side revocation ([17](17-security.md)). This persistence is opt-in and called out in [19](19-open-questions.md).
- **Never** place tokens, keys, or URL fragments in `localStorage`, the URL, logs, or telemetry ([02 §2.11](02-tech-stack-and-libraries.md)).
- **On logout**, clear all tokens *and* **drop the in-memory key session** (zeroize the unlocked identity-key handles), then revoke the server session (and, under the enterprise Keycloak path, perform Keycloak end-session) ([§14.8](#148-sessions--logout)).

## 14.5 Share-token sessions (guests)

Link guests authenticate **without any account login** (neither native auth nor Keycloak). `GET /share/{token}` resolves the token and mints the **15-min renewable share-session token** that authorizes relay/ciphertext reach; per relay connection a **single-use 60 s socket ticket** is issued ([13 §13.3](13-sharing.md), [09](09-realtime-collaboration.md)). The **decryption key is the URL fragment**, never the server ([server 08 §8.4](https://github.com/Nyxite/NyxiteServer)). Guest sessions run **without the per-account IndexedDB DB, keys, or device subsystem**: decrypted views are ephemeral, in-memory only, and discarded when the token lapses or the tab closes. Guest sessions are independent of accounts and may be opened with no account signed in.

## 14.6 Authorization model

Two independent gates ([server 08 §8.5](https://github.com/Nyxite/NyxiteServer)):

1. **Server ACL** — can the principal *reach* ciphertext / *join* the relay? (owner / grantee / link guest / admin-structure-only).
2. **Crypto** — can they *decrypt*? Requires the right wrapped or fragment file key; the server cannot influence this.

Resource-action policies the client expects the server to enforce:

| Policy | Granted to (server ACL) |
|--------|-------------------------|
| `resource:read-ciphertext` | Owner, grantee (read/write), guest via link; **admin: no** |
| `resource:write` | Owner, grantee `write`, guest via write link |
| `resource:share` | Owner |
| `admin:*` | Admin role — structure/usage/audit only |

There is deliberately **no `resource:read-content`**: reading content is a purely client-side capability gated by key possession, so the client must never assume ACL reach implies readability.

## 14.7 Multi-account / instance switching

The app is **multi-account from v1.0.0** ([01 §1.8](01-architecture.md)); each account is a fully isolated tenant in this browser:

- **Session isolation.** An account-scoped `UserSession` (keyed by `accountId`) owns that account's server tokens, unlocked identity-key handles, repositories, its **own IndexedDB database** (named per `accountId`), its `BlobCache` partition, and its search index. No content, names, index, or keys cross accounts ([04](04-local-data-model.md), [17](17-security.md)).
- **Per-account instance host.** Each account stores its **own instance bases** — API base (`/api/v1`), **relay hub URL**, **public-share base**, and (for the enterprise Keycloak option) the **OIDC authority** — so a personal self-hosted instance and a shared instance coexist side by side. The `authority`/`client_id` in [§14.2](#142-browser-auth-flow) come from the active account's config when the enterprise OIDC option is in use.
- **Switching.** An **account switcher** in the UI ([15](15-ui-and-navigation.md)) adds/switches/removes accounts. Switching **tears down the prior in-memory session** (zeroizing its keys, dropping its tokens) and roots the UI at the target account. Removing an account deletes its DB, cache, index, tokens, and key handles ("forget account").
- **Guests** ([§14.5](#145-share-token-sessions-guests)) sit outside this model and need no account.

## 14.8 Instance configuration

The static bundle is **instance-agnostic** ([00 §0.5](00-overview.md)). The instance bases are resolved **at runtime**, not baked in: a small **`config.json`** fetched on boot provides defaults, and each account may override them with its own host ([§14.7](#147-multi-account--instance-switching)). This keeps one CDN-served build usable against any Nyxite instance ([19](19-open-questions.md)).

## 14.9 Sessions & logout

- Auth is **stateless bearer** against the API; session lifetime is governed by the server-issued tokens. Logout clears tokens, drops the in-memory key session, and revokes the server session (under the enterprise Keycloak option it also calls **Keycloak end-session** — RP-initiated logout via oidc-client-ts) — it does **not** delete enrolled keys/local data unless the user chooses "forget account" ([§14.7](#147-multi-account--instance-switching)).
- **Error mapping** ([05 §5.4](05-api-client.md)): `401 token_expired` → one silent refresh + retry; persistent `401`/refresh failure → re-login (preserving the route, including an open `/share/{token}`); `403 2fa_required` → 2FA step-up (native TOTP/passkey re-prompt, or Keycloak step-up under the enterprise option); relay upgrade `401` → mint a fresh 60 s socket ticket.
