# 10 — Editors

Three content forms, each with a distinct **view mode** and **edit mode**: markdown, plain text, and handwritten ink. On the web the text editors are first-class (CodeMirror 6 bound to the Yjs doc), while ink is the **most constrained** surface — Pointer Events instead of a native stylus pipeline, so it ships **view first, then basic editing** ([00 §0.3](00-overview.md)). All decryption happens in the data layer; an editor only ever receives already-plaintext state ([01 §1.2](01-architecture.md)).

## 10.1 Shared editor scaffold

- A `feature-editor-*` screen is driven by an **`useEditorController` hook** (a Zustand-backed controller kept alive for the open file, not re-created per render — [01 §1.3](01-architecture.md)) exposing `EditorState` and dispatching actions:

```ts
interface EditorState {
  mode: "view" | "edit";
  content: DecryptedContent;            // markdown/plaintext string view-model, or ink page model
  sync: SyncState;                       // Synced | PendingPush | Uploading | … ([04](04-local-data-model.md))
  collab: CollabState;                   // connected | reconnecting | offline (text only)
  peers: AwarenessPeer[];                // remote carets/selection + presence ([09](09-realtime-collaboration.md))
}
type EditorAction =
  | "EnterEditMode" | "ApplyEdit" | "ToggleView"
  | "Snapshot" | "KeepInBrowser" | "Share" | "Back";
```

- **Opening a file** — `OpenFileForEdit`:
  1. Resolves the **FK** (unwrap via HPKE / fragment / device, [07](07-key-and-device-management.md)).
  2. **Bootstraps content**: for text, fetch the latest encrypted snapshot + the CRDT update log since it, decrypt, and replay into the Yjs doc; for ink, fetch the **latest encrypted blob** for the page set.
  3. For text, **joins the relay room** for live updates and awareness ([09](09-realtime-collaboration.md)).
- **Saving** is **continuous** for CRDT text — every edit is a Yjs update, encrypted and relayed immediately, no explicit save. For ink it is an **explicit/auto `PUT`** of the encrypted page ([08 §8.5](08-sync-engine.md)).
- **View ↔ edit toggle is instant** and preserves scroll position and caret/selection. **View mode is the default** for shared/received files (and for any file the user only opened to read).
- The controller owns the expensive, hard-to-rebuild state (the Yjs doc, the in-progress ink stroke buffer) and tears it down on `Back`/account switch so plaintext is not retained beyond the session ([01 §1.8](01-architecture.md)).

## 10.2 Markdown editor

- **Edit mode**: **CodeMirror 6** with the **`y-codemirror.next`** binding over the **Yjs-backed document**. Local keystrokes mutate the shared `Y.Text` → an encrypted Yjs update → relay; inbound updates apply to the doc and CodeMirror re-renders in place (no reload, caret preserved). A lightweight **formatting toolbar** (bold / italic / heading / list / link / code / checkbox) inserts the corresponding markdown syntax around the selection.
- **View mode**: **react-markdown** + **remark-gfm** + **rehype-sanitize** — headings, lists, tables, fenced code, **task-lists** (GFM), links, and images **only from available decrypted blobs**. **No remote network fetch from content** ever: external `http(s)`/`data:` image and link fetches are stripped by the sanitizer schema, and the CSP forbids content-origin connections (privacy + XSS, [17](17-security.md)). Only embedded/attached encrypted blobs the client already holds are rendered.
- **Collaboration overlays**: remote carets and selection ranges come from **awareness** ([09](09-realtime-collaboration.md)) rendered as CodeMirror decorations; **presence chips** (avatar + color) appear in the editor header. Each peer's color keys the caret and chip.
- **Large documents**: debounce the markdown parse in view mode; CodeMirror virtualizes the edit viewport natively. Defer re-parse while a burst of inbound updates is being applied.

## 10.3 Plain-text editor

Same **CRDT backbone** and collaboration as markdown (CodeMirror 6 + `y-codemirror.next` over the Yjs doc), **without rendering** — there is no view-mode parse, just the text. Monospace font optional. This is the basis for **`sourcecode`** (Phase 5), where **view-mode syntax highlighting** is added later via CodeMirror language modes; the edit/CRDT path is unchanged.

## 10.4 Ink editor (browser-constrained)

Goal: **pressure/tilt-aware** handwriting that round-trips faithfully against desktop/Android and stores in the shared encrypted vector format — accepting that browser stylus fidelity is below native ([00 §0.3](00-overview.md)). **View-first; basic editing.**

### Capture
- Use the **Pointer Events API** on a `<canvas>`. On `pointermove` read `pressure`, `tiltX`/`tiltY`; call **`getCoalescedEvents()`** to recover the sub-frame samples the browser batched into one frame, and **`getPredictedEvents()`** where available to hide latency on the wet stroke. Use **`pointerType`** to separate `"pen"` from `"touch"`/`"mouse"`: pen draws; touch is **palm-rejected** and routed to **pan/zoom** instead.

```ts
canvas.addEventListener("pointermove", (e: PointerEvent) => {
  if (e.pointerType !== "pen") return;            // palm rejection / finger = pan-zoom
  for (const s of e.getCoalescedEvents())          // sub-frame fidelity
    appendSample({ x: s.offsetX, y: s.offsetY,
                   pressure: s.pressure, tiltX: s.tiltX, tiltY: s.tiltY,
                   tRelMs: now() - strokeStartMs });
});
```

- Tools: **pen / highlighter / eraser**, color and base width, and a **pressure→width** mapping. The eraser removes whole strokes (stroke-level erase v1.0.0).

### Page model
- A file is a sequence of **pages**; a page holds **strokes**; a stroke is a polyline of **timestamped samples** `(x, y, pressure, tilt, orientation, tRelMs)` plus **brush attributes** (tool, color, base width, blend). `orientation` is recorded where the browser exposes it (often absent — stored as `null`).
- Editing operations: add stroke, erase stroke(s), insert/delete page. v1.0.0 ink sync is **LWW per page/file** (not CRDT): concurrent edits resolve last-write-wins, with the **loser retained** as a sibling version in history ([12](12-version-history.md)). A per-file **version-vector** (`{ deviceId -> counter }`, bumped on each local committed ink edit) rides in encrypted `metadata_enc` and classifies edits as equal/ancestor/concurrent ([08 §8.5](08-sync-engine.md)).

### Rendering
- Render **committed strokes** from the page model plus the **in-progress stroke** on a **`desynchronized`** (low-latency) Canvas 2D context where supported, then commit. Smooth strokes with **Catmull-Rom** segmentation for natural curves; honor pressure (and tilt where present) in width/opacity. Finger pan/zoom transforms the canvas viewport without touching the stroke geometry.

## 10.5 Ink storage format (shared, versioned, encryptable)

- The serialized page/file uses the **shared Nyxite ink vector format**: **deterministic CBOR** via **`cbor-x` in canonical mode** ([02 §2.8](02-tech-stack-and-libraries.md)) of pages → strokes → samples + brush metadata. It must be **byte-stable** so the **BLAKE3 content address is reproducible**, and **co-designed with desktop/Android** so a file drawn on one client opens on the others. The exact field schema is **`[P]` pending the cross-client co-design**; this client must conform exactly and is verified by the shared conformance vectors ([18](18-build-ci-testing.md)).
- The serialized page/file is **sealed per page/save with the FK** ([06 §6.3](06-cryptography.md)) and stored as an encrypted blob; the LWW version-vector rides in encrypted `metadata_enc` ([04](04-local-data-model.md)).
- The format carries a **`version`** for forward evolution and **reserves a recognized-text layer** so ink can later feed the search index ([11](11-search.md)).
- **Out of scope v1.0.0**: handwriting recognition / searchable ink text, and Samsung `.sdoc` import (separate migration item).

## 10.6 Performance & input quality

- Use **`getCoalescedEvents()`** so high-rate digitizers are not down-sampled to the display refresh; predict with `getPredictedEvents()` to mask latency.
- Keep heavy work **off the main thread where possible**: CBOR serialization, sealing, and BLAKE3 addressing run in the **crypto worker** ([01 §1.6](01-architecture.md)); the wet-stroke render path stays on the main thread but does no crypto.
- **Batch sample persistence** — never seal/encrypt per sample. **Seal per committed stroke-set / page save**, debounced, so a fast scribble is one encrypted write, not thousands.

## 10.7 Accessibility & cross-cutting

- Text editors: **scalable fonts**, ARIA roles/labels on the editor and toolbar, full **selection/clipboard** support, and **find-in-document** over the local decrypted text (no network).
- Ink: provide non-stylus **alternatives** (toolbar reachable by keyboard/mouse/touch; pen actions have button equivalents); ensure nothing requires a stylus to operate.
- **Limitation:** a browser **cannot enforce a secure / screenshot-proof window** — there is no `FLAG_SECURE` equivalent. The web client cannot block screenshots or screen capture; this is documented as an accepted platform limitation and folded into the threat model ([17](17-security.md)).
