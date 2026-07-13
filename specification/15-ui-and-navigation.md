# 15 — UI & Navigation

React 19 + shadcn/ui (Radix primitives) + Tailwind CSS v4, **dark-first** with the Nyxite brand. Routing is the Next.js App Router rendered **fully client-side under static export** ([00 §0.5](00-overview.md), [02 §2.1](02-tech-stack-and-libraries.md)). The web client is responsive (phone → desktop) and is the **primary guest surface**; the ink editor is "view + basic editing", not the showcase ([10](10-editors.md)). This mirrors the Android client's navigation discipline ([android 15](https://github.com/Nyxite/NyxiteAndroid)).

The UI implements the shared [NyxiteDesign](https://github.com/Nyxite/NyxiteDesign) system's **two-layer** adoption ([OPEN-DECISIONS DS](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md), Live decision DS): **Layer A** = the generated design tokens ([02 §2.2](02-tech-stack-and-libraries.md)) + the standard shadcn/ui component set used everywhere; **Layer B** = the consumer app shell — the left **rail** (Document / Presentation / Spreadsheet), the **three toolbar densities** (classic / slim / minimal), and the editor **canvas** — shared with the other consumer clients.

## 15.1 Route map (App Router segments, all client-rendered)

Every segment is a client component; there is no SSR of content and no server runtime ([01 §1.1](01-architecture.md)). The app shell (header, nav, providers, crypto worker, query client) is precached by the service worker ([16](16-offline-and-pwa.md)).

| Group | Segment | Screen |
|-------|---------|--------|
| **Auth** | `app/login/page.tsx` | Native login — password+TOTP + passkey (WebAuthn) ([14](14-authentication.md)) |
| | `app/auth/callback/page.tsx` | Enterprise Keycloak OIDC redirect handler (code → tokens, in memory) |
| | `app/enroll/page.tsx` | Browser (device) enrollment ([07](07-key-and-device-management.md)) |
| | `app/enroll/approve/page.tsx` | Approve another browser/device from this one |
| | `app/recovery/setup/page.tsx` | Recovery-key creation (BIP39, hard warnings) |
| | `app/recovery/restore/page.tsx` | Recover identity from recovery key |
| **Browse** | `app/(app)/projects/page.tsx` | Project list |
| | `app/(app)/project/[projectId]/page.tsx` | Project → folder/file list |
| | `app/(app)/folder/[folderId]/page.tsx` | Folder → folder/file list |
| | `app/(app)/search/page.tsx` | Local-subset search ([11](11-search.md)) |
| **Editor** | `app/(app)/file/[fileId]/page.tsx` | Dispatches to **text** or **ink** editor by `contentType` ([10](10-editors.md)) |
| **History** | `app/(app)/file/[fileId]/versions/page.tsx` | Version timeline |
| | `app/(app)/file/[fileId]/versions/[from]/[to]/page.tsx` | Client-computed diff view ([12](12-version-history.md)) |
| **Share** | `app/(app)/share/manage/[targetType]/[targetId]/page.tsx` | Manage account + link shares for a target ([13](13-sharing.md)) |
| | `app/share/[[...token]]/page.tsx` | **Guest entry** (optional catch-all, see §15.2) |
| **Settings** | `app/(app)/settings/page.tsx` | Root settings |
| | `app/(app)/settings/security/page.tsx` | Unlock method, idle auto-lock, session scope ([17](17-security.md)) |
| | `app/(app)/settings/devices/page.tsx` | Enrolled browsers/devices, revoke |
| | `app/(app)/settings/storage/page.tsx` | Usage, convenience cache, kept set ([16 §16.7](16-offline-and-pwa.md)) |
| | `app/(app)/settings/accounts/page.tsx` | Add / switch / remove accounts, per-account instance host |
| | `app/(app)/settings/about/page.tsx` | Version, licenses, build hash |
| **Support** | `app/(app)/support/report/page.tsx` | **Report a bug** composer — present only when `support.enabled` (§15.10) |
| | `app/(app)/support/tickets/page.tsx` | **My tickets** — own reports' status + replies (§15.10) |

`(app)` is a route group requiring an unlocked account session ([01 §1.8](01-architecture.md)); a route guard redirects to `login`/`enroll`/unlock when no session exists. Auth and guest segments live **outside** `(app)` so they render without a session.

## 15.2 Static-export routing detail **[P]**

`output: 'export'` disables SSR, route handlers, and middleware ([02 §2.1](02-tech-stack-and-libraries.md)), so every dynamic segment is resolved in the browser:

- **Guest link** — `app/share/[[...token]]/page.tsx` is a **client-rendered optional catch-all**. The static build emits one shell (`/share/index.html` plus a catch-all fallback) that handles `/share`, `/share/{token}`, and deeper paths. The component reads the token from `useParams()`/`location.pathname` and the **file key from `location.hash`** (`#k=…`), strips the fragment via `history.replaceState`, and proceeds entirely client-side ([13](13-sharing.md), [17 §17.5](17-security.md)).
- **Other dynamic segments** (`[projectId]`, `[folderId]`, `[fileId]`, versions, share-manage) are exported with a catch-all fallback page; the hosting layer is configured to serve the SPA shell for unknown paths (`trailingSlash`, a `404.html`/rewrite to the shell) so a deep-linked or reloaded route boots the app and the router re-resolves the segment from `LocalStore`/API.
- **Fallback shell** — while the bundle, crypto worker, account session, and data load, each route renders a skeleton (shadcn `Skeleton`) keyed to its layout; no blank document is ever shown.

**Deep links:** share URLs are `https://{host}/share/{token}#k=…` (host resolved per account/instance, [00 §0.5](00-overview.md)); the enterprise Keycloak OIDC redirect is `https://{host}/auth/callback`. Both are pure static paths needing no server.

## 15.3 Responsive layout

| Width | Layout |
|-------|--------|
| Wide (≥ `lg`) | **Master-detail**: shadcn `Sidebar` (account switcher + project/folder tree) + a `Resizable` two-pane (browse list ‖ open file/editor). Panel sizes persist (non-secret pref). |
| Medium | Collapsible sidebar (icon rail), single content pane; list and detail swap with in-app navigation. |
| Narrow (phone) | **Single-pane** with back navigation; bottom-sheet `Dialog`/`Drawer` for actions; sidebar becomes an off-canvas sheet. |

Core shadcn components: `Sidebar`, `Resizable`, `Dialog`/`Drawer`/`Sheet`, `Command` (the search/command palette, `Cmd/Ctrl-K`), `DropdownMenu`/`ContextMenu` (item actions), `Tabs`, `Tooltip`, `Toast` (status). The command palette spans navigation, file actions, and search jump-to-result.

## 15.4 Key screens

| Screen | Contents |
|--------|----------|
| **Projects / Folder list** | Decrypted names, folders + files, content-type icons (markdown/text/ink via Phosphor), **per-item sync + cache badges** ([08](08-sync-engine.md)), create/move/rename/delete (`Dialog` + `react-hook-form`/`zod`), **keep-in-browser toggle at file/folder/project (cascading)** ([16 §16.3](16-offline-and-pwa.md)), exclude-from-sync, inline search box, pull/refresh → delta sync. |
| **Text editor** | View ⇄ edit toggle; CodeMirror 6 bound to the Yjs doc with remote carets + presence chips ([09](09-realtime-collaboration.md)); markdown rendered via react-markdown + **rehype-sanitize** ([10 §10.2](10-editors.md)); relay connection state; history/share/snapshot actions. |
| **Ink editor** | Pointer-Events `<canvas>` (optionally `desynchronized`), tool palette (pen/highlighter/eraser, color, width), pages, undo/redo, zoom/pan, palm rejection; trimmed feature set vs. native ([10 §10.4](10-editors.md)). |
| **Version history** | Timeline with author + timestamp, fetch+view a version, diff two versions (worker-computed), restore ([12](12-version-history.md)). |
| **Share manage** | Account shares (HPKE) + link shares (URL fragment), create/revoke, read/write + expiry, copy-link with fragment warning, recipient public-key fingerprint ([13](13-sharing.md)). |
| **Search** | Query box, results with snippets + a **"local subset only"** scope hint ([11](11-search.md)). |
| **Settings** | Keep-in-browser defaults, convenience-cache toggle/cap + clear, storage usage breakdown ([16 §16.7](16-offline-and-pwa.md)), security (unlock method, idle auto-lock, "remember this browser"), devices, recovery-key status, accounts + per-account instance host, about. |
| **Onboarding / enrollment** | Login → browser enroll/approve → recovery-key creation with hard warnings ([07](07-key-and-device-management.md)). |
| **Guest reader / editor** | **Trimmed chrome**: no sidebar/account switcher; just the file, presence, and (if `write`) editing. Identity-bound actions (share, history, keep-in-browser) are hidden ([13](13-sharing.md)). |

## 15.5 Account switcher (multi-account)

Because the app is **multi-account from v1.0.0** ([14 §14.7](14-authentication.md)), the sidebar header hosts a persistent switcher showing the active account (and its instance host). Switching tears down the prior `UserSession` (drops in-memory keys/plaintext) and re-roots the browse tree at the new account's IndexedDB database ([01 §1.8](01-architecture.md)). "Add account" runs the auth+enroll flow against a chosen instance host.

## 15.6 Status surfacing

- **Per-file badges**: sync state (`Synced`/`PendingPush`/`PendingPull`/`Downloading`/`Uploading`/`Conflicted`/`Rotating`/`Excluded`/`Error`) and cache state (cached offline / title-only) — same canonical set as Android/[04](04-local-data-model.md). Clicking a badge explains it and offers actions.
- **Global**: an **offline banner**, an **"N pending"** outbox indicator ([08](08-sync-engine.md)), and the **relay connection state** in editors.
- **Conflicts**: LWW ink/binary conflicts surface as a non-destructive "two versions" prompt linking to history ([12](12-version-history.md)).
- **PWA install**: a dismissible install prompt (driven by `beforeinstallprompt`, deferred and surfaced as a shadcn banner/`Toast`) plus an "Install app" item in settings ([16 §16.2](16-offline-and-pwa.md)).

## 15.7 Theming & brand

Dark-first via CSS variables (light/dark tokens), Nyxite "Nyx" night palette, Nyxite icon/brand assets ([03](03-project-structure.md)). The light/dark semantic themes, deep-purple accent, spacing, radius, shadow, and motion all come from the shared [NyxiteDesign](https://github.com/Nyxite/NyxiteDesign) tokens — the generated CSS vars and Tailwind theme ([02 §2.2](02-tech-stack-and-libraries.md)), not hand-picked here. Typography is **Manrope** (UI) and **Source Serif 4** (document content); it is legible for long-form reading and code and respects the browser/system font-scaling (rem-based, no fixed px on text). Tailwind v4 tokens drive both shadcn and bespoke components.

**Fonts are self-hosted, never CDN-fetched.** Manrope and Source Serif 4 — and every brand asset — are **bundled and served same-origin** ([03 §3.1](03-project-structure.md)); the app **never** fetches fonts from Google Fonts or any external CDN at runtime. A runtime font fetch would leak the user's IP and request timing to a third party (and pull in Google), which is incompatible with the zero-knowledge stance — the CSP forbids it ([17](17-security.md)).

## 15.8 Accessibility & input

- ARIA semantics come from Radix primitives; all interactive controls are labelled and reachable by **full keyboard** (visible focus rings, logical tab order, the command palette as a keyboard entry point).
- Sufficient contrast in both themes; honor `prefers-reduced-motion` (disable non-essential transitions) and `prefers-color-scheme`.
- Pointer + touch + stylus coexist in the ink editor ([10 §10.4](10-editors.md)); touch targets sized for phones.

## 15.9 Empty / error / loading / locked / offline / unsupported states

Every screen defines explicit **loading** (skeletons), **empty** (no projects/files/results), **error** (typed `NyxiteApiError`, [05](05-api-client.md)), **locked** (needs unlock/enrollment → unlock dialog), and **offline** (read/edit cached subset, banner) states — no silent blank screens.

A dedicated **unsupported-browser screen** is shown at boot when a hard requirement is missing — **WebCrypto** (`crypto.subtle`), **IndexedDB**, **Service Worker / WebAssembly**, or a **secure context** ([00 §0.6](00-overview.md), [17](17-security.md)). It explains the requirement and lists supported browsers rather than degrading into an insecure path.

## 15.10 Bug reporting & support

In-app bug reporting routing to the maintainer-run `NyxiteSupport` helpdesk. This runs on the project's **one deliberate, consensual non-E2EE support plane**, disjoint from the content plane — see [17 §17.10](17-security.md), the master feature [support.md](https://github.com/Nyxite/Nyxite), the [NyxiteSupport `specification/02`](https://github.com/Nyxite/NyxiteSupport), and [OPEN-DECISIONS SUP-1–SUP-13](https://github.com/Nyxite/Nyxite).

- **Capability-gated surface (SUP-9).** The **"Report a bug"** entry (in the sidebar / command palette / `about`) and the **"My tickets"** view render **only when the server advertises the `support.enabled` capability flag** (v1 = the maintainer's official instance(s) only). Where the flag is absent the surfaces are **simply absent** — no disabled control, no hint.
- **Report composer.** Free-text **title + description** (`Dialog`/route + `react-hook-form`/`zod`), plus an optional screenshot and the diagnostic envelope below.
- **Screenshot capture + destructive redaction (SUP-2).** Optional capture of the current view by **rendering it to a `<canvas>`** (or a user-granted `getDisplayMedia` display capture), opened in a redaction editor offering **black-box** and **blur** tools. Redaction is **destructive and client-side**: the redacted regions are **flattened into the pixels (re-encoded PNG) before upload**, EXIF/metadata stripped; the **original image and any redaction mask are never sent** — there is no peel-back layer. Redaction is manual (the user decides what to hide).
- **Consent + destination notice before send (SUP-1).** Before a report can transmit, the user must confirm a clear notice that — **unlike their files — this report is *not* end-to-end encrypted and goes to the Nyxite maintainer**, shown alongside a GDPR disclosure. A report carries **no content key and no content-plane ciphertext**.
- **User-reviewable, editable diagnostic envelope.** A non-content technical bundle shown for **review + edit** before send: app version/build, platform/browser, locale, the **current screen/route id** (a UI location, never a file/project name or any content), **scrubbed** client-side error logs / recent stack traces, and coarse connection/relay state. Nothing is attached that is not represented in this envelope.
- **"My tickets" view (SUP-3).** An account user sees their own reports' status, the operator's **public replies**, and gets **in-app notifications** (`Toast`) of updates — a genuine two-way thread, not fire-and-forget. Guests (share/guest sessions) instead supply an email + optional name and receive replies via email + a tokenized no-login link (SUP-6).
- **Submission via the server as an authenticating relay (SUP-7).** The client **never contacts the helpdesk directly**: it submits to its own `NyxiteServer`, which authenticates the submitter and relays to `NyxiteSupport` tagged with the instance fingerprint + an opaque user reference. Submission is best-effort and off the critical path; a transport failure surfaces a retryable error and never blocks the app.
