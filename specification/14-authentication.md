# 14 — Authentication

Account authentication is **Keycloak OIDC with TOTP**, performed in the browser via the Authorization Code + PKCE flow. It yields an **API token but no content key** — decryption is governed entirely by the in-browser identity keys ([07](07-key-and-device-management.md)). The two layers are deliberately separate ([server 08](https://github.com/Nyxite/server)): authenticating the *account* never hands the server anything that can decrypt content. `AuthManager` (built on **oidc-client-ts**) owns tokens and silent refresh; it has no reference to the `CryptoEngine` or key material ([01 §1.5](01-architecture.md)).

## 14.1 Identity provider

- **Keycloak is the sole IdP** (OIDC): credentials, registration, password reset, and **TOTP enrollment/challenge** all happen at Keycloak, in its hosted pages. The API server is an **OIDC resource server** that validates bearer tokens against Keycloak JWKS; it stores no passwords, only a thin `users` projection keyed by `sub`.
- **Account auth ≠ content access.** Login produces an access token for `/api/v1`, **never a content key**. Content keys derive from the user's **identity private key** held in this browser/account session ([07](07-key-and-device-management.md)).

## 14.2 Browser auth flow

- **Authorization Code + PKCE** via **oidc-client-ts** against the Keycloak realm. Config (per account, runtime-resolved — [§14.7](#147-multi-account--instance-switching)): `authority` (issuer URL), `client_id`, `redirect_uri` (a static callback route), `scope` (`openid profile email` + any Nyxite scope), `response_type=code`.
- Redirect the **top-level browser** to Keycloak (not an iframe) for the authorization request; **TOTP is performed there** as part of Keycloak's flow. The static-export shell never renders credentials and never sees the password.
- Keycloak redirects back to the **statically-exported callback route** (e.g. `/auth/callback`); oidc-client-ts exchanges the code (with the PKCE verifier) for **access + refresh tokens**. The client validates `iss`, `aud`, `exp`, and signature against **JWKS**, and reads `sub`, name/email, roles (`user`/`admin`), and the `amr`/`acr` 2FA signal.

After a successful login the app checks device enrollment + identity-key availability ([07](07-key-and-device-management.md)): new account → generate identity keypair, enroll this browser, set recovery key; existing account on a new browser → device approval or recovery-phrase unwrap; already enrolled → unlock the identity key into the session. Until the identity key is present the app can show structure (encrypted names unreadable) but cannot decrypt content.

## 14.3 Token model

| Token | Lifetime | Source / use |
|-------|----------|--------------|
| OIDC **access** | **~5 min** (Keycloak-configured) | Bearer for `/api/v1`; refreshed routinely ([§14.4](#144-token-storage--refresh)). |
| OIDC **refresh** | Keycloak-configured | Drives silent renewal of the access token. |
| Relay **socket ticket** | **single-use, 60 s** | Minted from the bearer to authorize a relay upgrade ([09](09-realtime-collaboration.md)). |
| Guest **share-session token** | **15 min, renewable** while the share is valid | Authorizes relay/ciphertext **reach** for link guests — *not* a content key ([13 §13.3](13-sharing.md)). |

- **Silent renewal:** oidc-client-ts refreshes the access token **before expiry** (refresh-token grant, or a silent-iframe `prompt=none` request where refresh tokens aren't available). `ApiClient` attaches the bearer; on `401 token_expired` it performs a **single** silent refresh and retries, surfacing re-login only if that fails ([05 §5.4](05-api-client.md)).
- **2FA gating:** the `amr`/`acr` signal may be required for sensitive **account/admin** actions; otherwise the API returns **`403 2fa_required`** and the app routes the user back through a Keycloak step-up. There is **no content break-glass** to gate — content is unreadable to the server regardless.

## 14.4 Token storage [P]

The browser is a hostile environment for secrets ([17](17-security.md)). Therefore:

- **Prefer in-memory access tokens** managed by oidc-client-ts, with refresh handled in-session — **not `localStorage`/`sessionStorage`**, to limit XSS token theft. Tokens live in the account session and are gone on tab close.
- **If refresh tokens must persist** (to survive reloads without a full re-login), treat them with the **same caution as key material** and document the trade-off explicitly: the XSS exposure window grows, mitigated by a strict CSP, short access-token TTL, and instant server-side revocation ([17](17-security.md)). This persistence is opt-in and called out in [19](19-open-questions.md).
- **Never** place tokens, keys, or URL fragments in `localStorage`, the URL, logs, or telemetry ([02 §2.11](02-tech-stack-and-libraries.md)).
- **On logout**, clear all tokens *and* **drop the in-memory key session** (zeroize the unlocked identity-key handles), then perform Keycloak end-session ([§14.8](#148-sessions--logout)).

## 14.5 Share-token sessions (guests)

Link guests authenticate **without Keycloak**. `GET /share/{token}` resolves the token and mints the **15-min renewable share-session token** that authorizes relay/ciphertext reach; per relay connection a **single-use 60 s socket ticket** is issued ([13 §13.3](13-sharing.md), [09](09-realtime-collaboration.md)). The **decryption key is the URL fragment**, never the server ([server 08 §8.4](https://github.com/Nyxite/server)). Guest sessions run **without the per-account IndexedDB DB, keys, or device subsystem**: decrypted views are ephemeral, in-memory only, and discarded when the token lapses or the tab closes. Guest sessions are independent of accounts and may be opened with no account signed in.

## 14.6 Authorization model

Two independent gates ([server 08 §8.5](https://github.com/Nyxite/server)):

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

- **Session isolation.** An account-scoped `UserSession` (keyed by `accountId`) owns that account's OIDC tokens, unlocked identity-key handles, repositories, its **own IndexedDB database** (named per `accountId`), its `BlobCache` partition, and its search index. No content, names, index, or keys cross accounts ([04](04-local-data-model.md), [17](17-security.md)).
- **Per-account instance host.** Each account stores its **own instance bases** — API base (`/api/v1`), **OIDC authority**, **relay hub URL**, and **public-share base** — so a personal self-hosted instance and a shared instance coexist side by side. The `authority`/`client_id` in [§14.2](#142-browser-auth-flow) come from the active account's config.
- **Switching.** An **account switcher** in the UI ([15](15-ui-and-navigation.md)) adds/switches/removes accounts. Switching **tears down the prior in-memory session** (zeroizing its keys, dropping its tokens) and roots the UI at the target account. Removing an account deletes its DB, cache, index, tokens, and key handles ("forget account").
- **Guests** ([§14.5](#145-share-token-sessions-guests)) sit outside this model and need no account.

## 14.8 Instance configuration

The static bundle is **instance-agnostic** ([00 §0.5](00-overview.md)). The instance bases are resolved **at runtime**, not baked in: a small **`config.json`** fetched on boot provides defaults, and each account may override them with its own host ([§14.7](#147-multi-account--instance-switching)). This keeps one CDN-served build usable against any Nyxite instance ([19](19-open-questions.md)).

## 14.9 Sessions & logout

- Auth is **stateless bearer** against the API; session lifetime is governed by Keycloak. Logout clears tokens, drops the in-memory key session, and calls **Keycloak end-session** (RP-initiated logout via oidc-client-ts) — it does **not** delete enrolled keys/local data unless the user chooses "forget account" ([§14.7](#147-multi-account--instance-switching)).
- **Error mapping** ([05 §5.4](05-api-client.md)): `401 token_expired` → one silent refresh + retry; persistent `401`/refresh failure → re-login (preserving the route, including an open `/share/{token}`); `403 2fa_required` → Keycloak step-up; relay upgrade `401` → mint a fresh 60 s socket ticket.
