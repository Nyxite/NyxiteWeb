# 03 — Project Structure

The web client is a **single Next.js 15 package** (no Gradle-style multi-module split), so the compile-time module boundaries the Android client gets from Gradle ([android 03](https://github.com/Nyxite/NyxiteAndroid)) are reproduced here with **TypeScript path aliases + an ESLint boundary rule** ([01 §1.2](01-architecture.md)). The layering is identical: presentation → domain → data, with crypto and network isolated so plaintext can never reach a network client.

## 3.1 Folder layout

```
NyxiteWeb/
├── package.json                       # pnpm; scripts: dev, build (next build → out/), lint, test, e2e
├── pnpm-lock.yaml
├── next.config.ts                     # output:'export', strict, headers for dev only (CSP is host-served, [17])
├── tsconfig.json                      # strict; path aliases (§3.3)
├── tailwind.config.ts · postcss.config.mjs
├── eslint.config.mjs                  # eslint-plugin-boundaries layer rules (§3.5)
├── vitest.config.ts · playwright.config.ts
├── components.json                    # shadcn/ui config (copied-in components)
│
├── app/                               # App Router — ALL segments client-rendered (static export)
│   ├── layout.tsx                     # root shell: providers (Query, theme, account session), <html>
│   ├── page.tsx                       # redirect → /app or /login
│   ├── globals.css                    # Tailwind layer + brand CSS variables
│   ├── login/page.tsx                 # native login (password+TOTP + passkey) ([14])
│   ├── auth/callback/page.tsx         # enterprise Keycloak OIDC redirect handler (PKCE) ([14])
│   ├── app/                           # authenticated area (account session required)
│   │   ├── layout.tsx                 # nav chrome, account switcher, sync badges
│   │   ├── page.tsx                   # project list / browse root
│   │   ├── p/[projectId]/...          # browse tree, folder, file views (client params)
│   │   └── settings/...               # accounts, storage, security, keep-in-browser
│   └── share/[[...token]]/page.tsx    # optional catch-all; guest bootstrap, fragment key ([13])
│
├── src/
│   ├── domain/                        # pure TS — NO window/fetch/React/DOM imports
│   │   ├── entities/                  # File, Project, Folder, Share, FileKey, Version… (types/enums)
│   │   ├── repositories/              # FileRepository, StructureRepository, KeyRepository,
│   │   │                              #   ShareRepository, SyncRepository, CollabRepository,
│   │   │                              #   SearchRepository, VersionRepository (interfaces)
│   │   ├── usecases/                  # verb-first use cases ([01 §1.4]): OpenFileForEdit, RunDeltaSync…
│   │   └── policy/                    # sync-state machine, keep-in-browser resolution, key-gen rules
│   │
│   ├── data/                          # repository impls + the engines (ONLY layer wiring crypto+network)
│   │   ├── repositories/              # *RepositoryImpl — orchestrate engines below
│   │   ├── local/                     # LocalStore: Dexie DB class + schema + migrations ([04])
│   │   │   ├── db.ts                  # Dexie subclass, version()/stores(), per-accountId DB name
│   │   │   ├── tables.ts             # Dexie row interfaces (FileRow, FileKeyRow, …)
│   │   │   └── accountRegistry.ts    # cross-account registry DB (open before unlock)
│   │   ├── api/                       # ApiClient: typed fetch + TanStack Query fns, DTOs, zod ([05])
│   │   ├── relay/                     # RelayClient: @microsoft/signalr RelayHub wrapper ([09])
│   │   ├── crypto/                    # CryptoEngine: main-thread façade + worker RPC proxy ([06])
│   │   ├── crdt/                      # CrdtEngine: Yjs apply/encode/state-vector/snapshot ([09])
│   │   ├── keyvault/                  # KeyVault: non-extractable CryptoKey handles in IDB ([07])
│   │   ├── blobcache/                 # BlobCache: Cache Storage + IDB metadata ([16])
│   │   ├── auth/                      # AuthManager: native fetch/WebAuthn (+ oidc-client-ts for enterprise), token + share-token store ([14])
│   │   └── mappers/                   # DTO ↔ entity ↔ Dexie row mappers (DTOs never escape data/)
│   │
│   ├── features/                      # feature slices: components + hooks + Zustand store, each self-contained
│   │   ├── browse/                    # project/folder/file navigation
│   │   ├── editor/                    # text (CodeMirror+Yjs) and ink (Pointer Events) editors ([10])
│   │   ├── history/                   # version list, client diff, restore ([12])
│   │   ├── share/                     # create/manage shares, open link, guest mode ([13])
│   │   ├── settings/                  # config, keep-in-browser, storage usage, accounts, security
│   │   └── auth/                      # login, TOTP handoff, browser enrollment, recovery ([14],[07])
│   │
│   ├── components/ui/                 # shadcn/ui primitives copied in-repo (owned, not a CDN dep)
│   │
│   ├── lib/                           # cross-cutting, leaf-only utilities
│   │   ├── config.ts                 # runtime instance config (config.json loader), per-account host
│   │   ├── result.ts · time.ts · base64url.ts · uuidv7.ts
│   │   └── logger.ts                 # scrubbing Logger facade ([02 §2.11], [17])
│   │
│   └── workers/                       # bundled as separate worker entries
│       ├── crypto.worker.ts          # WASM (BLAKE3, Argon2id) + bulk AES-GCM/HPKE ([01 §1.6])
│       └── diff.worker.ts            # optional: large version diffs ([12])
│
├── service-worker/                    # SW SOURCE (Workbox); built to public/sw.js — app-shell only ([16])
│   └── sw.ts                          # precache + runtime routing; NEVER decrypts, holds no keys ([17])
│
└── public/
    ├── config.json                    # runtime instance config (API base, relay, share base; OIDC authority for enterprise Keycloak)
    ├── manifest.webmanifest           # PWA identity (§3.4)
    ├── icons/                         # from Nyxite/icons set; install/maskable icons
    └── *.wasm                         # blake3, argon2id, libsodium WASM artifacts
```

`app/` holds only thin route segments (each `'use client'`); all real logic lives under `src/`. The presentation layer (`app/`, `features/`) imports from `domain/` only; `data/` is reached through dependency injection at the composition root in `app/layout.tsx` (a small provider that constructs the account-scoped repositories — [01 §1.8](01-architecture.md)).

## 3.2 Module → responsibility

| Path | Responsibility | Layer / may import |
|------|----------------|--------------------|
| `app/**` | Route segments, client-rendered shells, provider composition root | Presentation → `domain`, `features`, `components/ui`, `lib` |
| `src/features/*` | Screen components, hooks, per-feature Zustand store | Presentation → `domain`, `components/ui`, `lib` |
| `src/components/ui` | shadcn/Radix primitives (owned in-repo) | Presentation leaf → `lib` |
| `src/domain/entities` | Pure entities, enums, value types | Domain → nothing (leaf) |
| `src/domain/repositories` | Repository **interfaces** | Domain → `entities` |
| `src/domain/usecases` | Single-responsibility use cases (`invoke`-style) | Domain → `entities`, `repositories`, `policy` |
| `src/domain/policy` | Sync-state machine, keep-in-browser resolution, key-gen staleness | Domain → `entities` |
| `src/data/repositories` | Repository **impls**: orchestrate engines, seal/open at the boundary | Data → `domain` + all engines |
| `src/data/local` (`LocalStore`) | Dexie DB, schema, migrations, account registry | Data → `lib` |
| `src/data/api` (`ApiClient`) | Typed REST over `/api/v1`, DTOs, zod, problem+json | Data → `lib` (**not** crypto/domain-content) |
| `src/data/relay` (`RelayClient`) | SignalR `RelayHub` connect/join/submit/awareness | Data → `lib` (**not** crypto) |
| `src/data/crypto` (`CryptoEngine`) | seal/open frames, HPKE, sign/verify, address, KDF; worker RPC | Data → `workers`, `lib` |
| `src/data/crdt` (`CrdtEngine`) | Yjs updates, state vectors, snapshots | Data → `lib` |
| `src/data/keyvault` (`KeyVault`) | Identity-key handles, FK handle map, unlock gating | Data → `lib` |
| `src/data/blobcache` (`BlobCache`) | Cached ciphertext/decrypted blobs, eviction | Data → `lib` |
| `src/data/auth` (`AuthManager`) | Server access/refresh tokens, silent refresh, relay ticket, guest share-token | Data → `lib` |
| `src/data/mappers` | DTO ↔ entity ↔ Dexie row conversion | Data → `domain`, `api`, `local` |
| `src/lib/*` | Config, Result, time, base64url, uuidv7, scrubbing logger | Leaf → nothing |
| `src/workers/*` | Off-main-thread crypto + diff | Worker context (no DOM) |
| `service-worker/sw.ts` | App-shell precache, offline navigation | Separate, less-trusted context ([17]) |

## 3.3 Path aliases

`tsconfig.json` `paths` map the layers so imports read like the Android module graph and the boundary rule can target them:

```jsonc
"paths": {
  "@/app/*":        ["app/*"],
  "@/domain/*":     ["src/domain/*"],
  "@/data/*":       ["src/data/*"],
  "@/features/*":   ["src/features/*"],
  "@/ui/*":         ["src/components/ui/*"],
  "@/lib/*":        ["src/lib/*"],
  "@/workers/*":    ["src/workers/*"]
}
```

- **Crypto worker entry:** `@/workers/crypto.worker.ts`, instantiated by `CryptoEngine` via `new Worker(new URL('@/workers/crypto.worker.ts', import.meta.url), { type: 'module' })` so the bundler emits a hashed worker chunk. Same pattern for the optional `diff.worker.ts`.
- **Dexie schemas** live in `@/data/local/db.ts` (the `Dexie` subclass with `version().stores()`) and `@/data/local/tables.ts` (row interfaces) — see [04](04-local-data-model.md).
- **Service worker source** lives outside `src/` in `service-worker/sw.ts` and is built separately to `public/sw.js`; it is intentionally not on the app's import graph (different trust tier).

## 3.4 Naming conventions

- Engines/clients are an **interface + impl**: `CryptoEngine` (interface) / `CryptoEngineImpl` or `WebCryptoEngine`; `RelayClient` / `SignalRRelayClient`; backing-tech prefixes (`Dexie*`, `Yjs*`, `Hpke*`) signal the library, mirroring the Android `Tink*`/`Yrs*` convention.
- Use cases are verb-first classes/functions (`OpenFileForEdit`, `RunDeltaSync`) with a callable `invoke`.
- DTOs (network) carry the `Dto` suffix and **never escape `data/`** — mapped to entities in `data/mappers`.
- Dexie row interfaces carry a `Row` suffix and never leave the data layer.
- Components: `XView`/`XScreen` for the screen, `useX` for hooks, `xStore` for the feature's Zustand store, `XDialog`/`XSheet` for overlays.
- Files: `kebab-case.ts`; React components `PascalCase.tsx`.

## 3.5 Boundary enforcement (`eslint-plugin-boundaries`)

The single-package equivalent of the Android `konsist`/Gradle boundary ([01 §1.2](01-architecture.md)). Element types are bound to the aliases above and these rules **fail the build**:

| Element (`type`) | May import |
|------------------|------------|
| `app` (composition root) | everything |
| `feature` (`src/features/*`) | `domain`, `ui`, `lib` — **never** `data`, `crypto`, `relay`, `api` |
| `ui` (`src/components/ui`) | `lib` |
| `domain` (`src/domain/*`) | `domain`, `lib` only — no DOM/IO/React/library imports |
| `data` (`src/data/*`) | `domain`, sibling engines, `lib` |
| `api`/`relay` (`src/data/api`,`src/data/relay`) | `lib` — **forbidden**: `crypto`, `crdt`, any `domain/entities` content type |
| `lib` (`src/lib/*`) | nothing (leaf) |

The critical rule is the last one: a network module that imports a crypto module or a plaintext-carrying entity type is rejected, making it structurally impossible to serialize plaintext onto the wire ([01 §1.2](01-architecture.md)). A complementary lint forbids `localStorage`/`sessionStorage`/URL writes of secret-typed values outside `lib/logger` allowlists ([17](17-security.md)).

## 3.6 PWA app identity

- The bundle is **instance-agnostic** ([00 §0.5](00-overview.md)) but the manifest gives the installed app a stable identity at the **serving host** (e.g. `https://nyxite.app`):

| Manifest field | Value |
|----------------|-------|
| `id` | `/` (resolved against origin → per-host install identity) **[P]** |
| `name` / `short_name` | `Nyxite` / `Nyxite` |
| `start_url` | `/app` |
| `scope` | `/` |
| `display` | `standalone` |
| `theme_color` / `background_color` | Nyxite brand (dark-first, [15 §15.4](15-ui-and-navigation.md)) |
| `icons` | `public/icons/**` (incl. maskable) |

- **No native application id.** Unlike the Android client's `app.nyxite.android` application ID ([android 03 §3.2](https://github.com/Nyxite/NyxiteAndroid)), the web app has no OS package identity — its identity is **origin-scoped** (the manifest `id` resolved against the host serving the static bundle). Two instances on different hosts install as distinct PWAs; multi-account ([01 §1.8](01-architecture.md)) is handled inside a single installed app via per-account IndexedDB.

## 3.7 Repository files

Per org convention ([master `docs/LICENSING-AND-REPOS.md`](https://github.com/Nyxite/Nyxite)): `README.md`, `FEATURES.md`, `LICENSE.md` (PolyForm Noncommercial 1.0.0), this `specification/` folder, and the Next.js project. Conformance test vectors (crypto KATs, CRDT wire) live under `src/data/crypto/__vectors__` and `src/data/crdt/__vectors__`, shared with the other clients ([18](18-build-ci-testing.md)).
