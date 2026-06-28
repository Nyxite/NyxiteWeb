# 16 — Offline & PWA

The web client is a **full installable PWA** ([00 §0.5](00-overview.md)). With the service worker installed and the local subset cached, the user can browse, read, edit, draw, and search the cached subset with no connectivity; changes queue in the outbox and flush on reconnect ([08](08-sync-engine.md)). As on Android, local storage is **a set of per-file local copies the user controls** ([android 16](https://github.com/Nyxite/android)) — but here it is bounded by browser Storage-API quota and eviction, and everything is gated by the at-rest model in [17](17-security.md). **The service worker never decrypts content or holds keys** ([17 §17.7](17-security.md)).

## 16.1 Web App Manifest

| Field | Value |
|-------|-------|
| `name` / `short_name` | "Nyxite" / "Nyxite" |
| `id` | stable app id (e.g. `/?app=nyxite`) so installs are de-duplicated |
| `start_url` | `/` (scope `/`) |
| `display` | `standalone` |
| `theme_color` / `background_color` | Nyxite dark brand tokens ([15 §15.7](15-ui-and-navigation.md)) |
| `icons` | maskable + any, full PWA icon set from the brand assets |
| `share_target` **[P]** | optional — accept shared text/files into a "new note" flow; opt-in, never auto-uploads plaintext |

Installability requires HTTPS (secure context), a registered service worker, and the manifest. An in-app install prompt is offered via `beforeinstallprompt` ([15 §15.6](15-ui-and-navigation.md)).

## 16.2 Service worker (Workbox)

- **Precache** the app shell (HTML shell, JS/CSS chunks, WASM for BLAKE3/Argon2id, fonts, icons) via Workbox precaching with revision hashes, so the app boots offline and is integrity-checked ([17 §17.7](17-security.md)).
- **Runtime caching** is deliberately narrow:

| Request | Strategy | Rationale |
|---------|----------|-----------|
| Static assets / chunks not in precache | `StaleWhileRevalidate` | App-shell assets only |
| **GET ciphertext blob by content address** | **`CacheFirst`** | Content-addressed (BLAKE3) blobs are **immutable**, so address = cache key; safe to cache forever, evict by capacity ([06](06-cryptography.md)) |
| Authed API JSON (structure, manifests, shares) | **Not cached by the SW** | Authed + per-account; cached in IndexedDB via TanStack Query/repositories instead ([05](05-api-client.md), [01](01-architecture.md)) |
| Auth/relay endpoints (native auth, relay, and enterprise OIDC) | Never intercepted | Auth/realtime must hit the network |

The SW caches **only** static assets and immutable ciphertext blobs (by address). It does not see access tokens, the fragment key, or any plaintext, and has no path to the crypto worker or key vault ([17](17-security.md)). Decryption of cached ciphertext happens in the page/worker, never in the SW.

## 16.3 SW update flow

A new deploy publishes a new precache manifest. The waiting worker uses `skipWaiting` + `clientsClaim` **only after user consent**: the app detects the waiting worker and shows a non-blocking **"Update available — reload"** prompt; on accept it messages the SW to activate and reloads. This avoids swapping chunks under a live editing/crypto session.

## 16.4 Local subset model

IndexedDB (Dexie) is the **source of truth** the UI renders from ([01 §1.1](01-architecture.md), [04](04-local-data-model.md)). What lives locally is the user-controlled subset:

- **`keepInBrowser`** — a per-item pin with values `keep` / `dontKeep` / `inherit`, at **file / folder / project** granularity with **cascade** (this is the web analogue of Android's keep-on-device). It is a **client-local choice, never sent to the server** — every kept/not-kept file is still `server-default` server-side; the separate server **`excluded`** policy ("device-only, never upload") is independent ([08 §8.2](08-sync-engine.md), [00 §0.8](00-overview.md)).
  - Effective state = the item's own explicit setting, else nearest ancestor, else account default (**not kept**). New items inherit the parent's effective setting.
  - A **kept** file is proactively downloaded, decrypted, **indexed for search** ([11](11-search.md)), and available offline.
- **Not-kept files** are fetched and decrypted **on open**, then discarded on close.
- **Convenience cache** (default **on**, optional small size cap) retains *recently-opened* not-kept files so reopening is instant; it never holds kept files (guaranteed) or `excluded` content, and evicting an item only means re-download next time.
- **On eviction** of a not-kept body, drop its FTS **body** but keep its **title** so it stays discoverable; reopening re-indexes it ([11](11-search.md)). So FTS body set = kept files + files currently in the convenience cache.
- **Prefetch** of kept content runs when online; respect **Save-Data** and metered-connection hints (`navigator.connection`) and pause on a metered/Save-Data connection unless the user opts in.

## 16.5 Storage limits & eviction

- Call **`navigator.storage.persist()`** at enrollment/first kept-pin to request **persistent** storage so the browser does not silently evict the local subset or wrapped keys; surface whether it was granted.
- Use **`navigator.storage.estimate()`** for the usage/quota UI ([§16.7](#167-storage-usage-settings-screen)).
- Handle **`QuotaExceededError`**: stop caching new bodies, surface a clear **"storage full"** state, and let the user free space (clear convenience cache) or reduce the kept set; never silently drop kept content.
- **Cross-browser caveats**: quotas differ widely (Chromium per-origin %-of-disk vs. tighter Firefox/Safari limits); **Safari/ITP** can evict IndexedDB/Cache Storage for sites that are **not installed** and unused for ~7 days — document this, push install + `persist()`, and treat the server as the durable copy of everything except `excluded` content.

## 16.6 Offline UX

- Read and edit the cached subset offline; queue mutations in the **outbox** ([08](08-sync-engine.md)).
- Use the **Background Sync API** (where supported) to flush the outbox on reconnect; otherwise flush on the next foreground/online event.
- Clear, consistent banners: offline / "N pending" / reconnecting / relay state ([15 §15.6](15-ui-and-navigation.md)).

## 16.7 Storage usage settings screen

Surfaces ([15 §15.4](15-ui-and-navigation.md)):

- A **breakdown** (kept content vs. convenience cache vs. ciphertext blob cache vs. FTS index) from `storage.estimate()`.
- Toggle / cap / **clear the convenience cache** (evicts not-kept bodies and their FTS bodies; titles/metadata remain).
- **Manage the kept set** (keep-in-browser toggles at file/folder/project, mirrored from the browse list).
- Persistent-storage status and a re-request action; QuotaExceeded guidance.

## 16.8 Export **[P]**

- **Single-file export**: the user may explicitly trigger a **browser download** of one **decrypted** file (e.g. the markdown source or a rendered note) for portability — a deliberate, per-item action.
- **No automatic bulk plaintext export to disk**: the client never writes a general plaintext working copy or dumps the corpus to the filesystem (privacy; keeps content inside the encrypted boundary). **Bulk export remains a desktop feature**, consistent with Android ([android 16](https://github.com/Nyxite/android)).
