# NyxiteWeb — Implementation Readiness Report

**Question:** Can NyxiteWeb be **fully implemented across all phases** using only (a) the NyxiteWeb repo (`specification/` + `FEATURES.md`) and (b) the shared Nyxite info repo (`docs/`, `features/`, `implementation/`)?

**Verdict: NO — not fully.** The vast majority of the client (essentially all of Phases 0–2, the core E2EE web client) is specified to an exceptional, build-ready standard. But a small set of **hard blockers** remain, all of the same character: they are **cross-repo / cross-client artifacts that are authoritative outside these two repos**, or **co-design artifacts the plan itself defers to a later phase and has not yet produced**. Without them you cannot achieve the non-negotiable byte-for-byte interop gate, and you cannot complete Phase 3 (ink), Phase 4.3 (key transparency), or Phase 4.4 (groups) correctly.

**Hard blockers: 6.** (Plus 2 optional Phase-6 gaps and several minor ambiguities.)

---

## Scope covered by this assessment

Read in full: all 24 files of `NyxiteWeb/specification/` (00–20 + README), `NyxiteWeb/FEATURES.md`; `Nyxite/docs/SPECIFICATION.md`, `OPEN-DECISIONS.md`; `Nyxite/features/web.md`, `groups.md`; `Nyxite/implementation/README.md`, `test-methods/README.md`, and phase files `0.1`, `3.1`, `4.3`, `4.4`, `5.1`, `6.2`, `6.3` directly (the phases carrying the risk), with the remaining phase files cross-checked against the web spec chapters that cover their `-WEB-` deliverables.

Every `-WEB-` step across the 19 build phases maps cleanly onto a web `specification/` chapter:

| Phase | Web step(s) | Backing spec | Buildable from the two repos? |
|---|---|---|---|
| 0.1 Account & identity | P0.1-WEB-1/2/3 | 01,02,03,04,06,07,14,16,17 | Yes, except shared crypto vectors (HB-1) |
| 0.2 Encrypted structure | (WEB) | 04,05,15 | Yes, except exact server DTOs (HB-2) |
| 1.1 Notes that sync | (WEB) | 08,09,10 | Yes, except sync payload shapes (HB-3) |
| 1.2 Storage control & search | (WEB) | 08,11,16 | Yes |
| 2.1 Live collaboration | (WEB) | 09 | Yes, except CRDT/relay wire confirmation |
| 2.2 Sharing | (WEB) | 13,14 | Yes |
| 2.3 Revocation & rotation | (WEB) | 07,13 | Yes |
| 2.4 Version history | (WEB) | 12 | Yes |
| 3.1 Handwriting | P3.1-WEB-1/2 | 10 §10.4–10.5 | **No — ink CBOR schema undefined (HB-4)** |
| 4.1 Device/key & multi-account | (WEB) | 07,14,15 | Yes |
| 4.2 Admin & audit | (WEB, minimal) | 05,17 | Yes (admin dashboard is a separate repo) |
| 4.3 Key transparency | P4.3-WEB-1 | (no web chapter) | **No — proof format undefined (HB-5)** |
| 4.4 Groups | P4.4-WEB-1 | 06 §6.14, 07 §7.11, 13 §13.9 | **No — depends on 4.3 + CORE fixtures (HB-6)** |
| 4.5 Polish & distribution | (WEB) | 15,16,18 | Yes (minor: design system, see M-3) |
| 5.1 Office docs | P5.1-WEB-1 | 05 §5.6 (seam), 16 | Partial — chunked-upload contract + office viewer thin |
| 5.2 Source code | P5.2-WEB-1 | 10 §10.3 | Yes |
| 5.3 Images | P5.3-WEB-1 | (seam only) | Partial — image viewer/thumbnail approach thin |
| 6.2 Metadata-graph hiding | P6.2-WEB-1 | 03 (seam) | No — scheme undefined by design (optional) |
| 6.3 Leak-free search index | P6.3-WEB-1 | 11 §11.7 (seam) | No — scheme undefined by design (optional) |

The web `specification/19-open-questions.md` is fully ratified (every web-side fork is decided), which is why the remaining gaps are almost entirely **externally-owned** (server ledger, cross-client co-design) rather than web decisions still open.

---

## HARD BLOCKERS

### HB-1 — Shared crypto conformance vectors + the server's canonical algorithm ledger
- **What is missing:** The actual **KAT / cross-client vector files** (AES-256-GCM framing, HPKE wrap/unwrap pairs, Ed25519, X25519, BLAKE3, Argon2id) and the server's **canonical algorithm ledger** (referenced as `server 07 §7.3`). The web spec pins *every primitive parameter* superbly (06 §6.2, frame magic `NYXC`/version `0x01`/AAD/object_kind enum, HPKE suite `0x0020/0x0001/0x0002`, Argon2id `m=64MiB,t=3,p=1`), but the vector **files do not exist in either repo** — `implementation/phase-0.1.md` (P0.1-CORE-2) explicitly *authors them during Phase 0.1*, and `03 §3.7`/`18 §18.6` say they are "co-owned with server/desktop/android."
- **Also unpinned without vectors:** a few framing minutiae the spec says are "**locked by vectors**" — e.g. HPKE base-mode `info`/`aad` ("empty unless the protocol specifies otherwise — locked by vectors", 06 §6.5).
- **Blocks:** the **non-negotiable conformance gate** that gates *every* phase (18.6, 20 "Every phase is gated by the cross-client conformance vectors"). Foundational to Phase 0.1 (P0.1-WEB-2) and all crypto. **This is the highest-priority blocker of the six** — the byte-for-byte interop test that every subsequent phase depends on cannot run at all without it, and the vector-locked framing minutiae (e.g. HPKE `info`/`aad`) cannot be resolved without it.
- **Where I looked:** web 06, 18 §18.6, 03 §3.7; `implementation/phase-0.1` P0.1-CORE-2, README "conformance harness."
- **Need:** the checked-in shared vector files (or the server ledger + one counterpart implementation to generate/verify them). You can *implement* the crypto from the specs; you cannot *prove interop* or resolve the vector-locked framing details from these two repos alone.
- **Related dependency:** the audited WASM PQC library for the hybrid halves is gated on these same shared vectors — see M-6.

### HB-2 — Exact server REST contract / OpenAPI (`/openapi/v1.json`)
- **What is missing:** The authoritative server wire contract. Web `05` gives a thorough TS translation of the DTOs and a full endpoint list, but explicitly derives it from `server 04 §4.2` / "the server C# records," and `05 §5.7` states the zod schemas are validated **against `/openapi/v1.json` in CI** — an artifact not present in either repo. Exact field names/nullability, and the exact `application/problem+json` `type` URIs behind the `NyxiteApiError` discriminants (05 §5.4), live in the server repo.
- **Blocks:** `ApiClient` + zod schemas + error mapping (Phase 0.2 onward) — every phase that calls the API. Buildable defensively, but not to a guaranteed-correct, CI-green contract.
- **Where I looked:** web 05 (all), 18 §18.7 (CI: "zod schemas checked against /openapi/v1.json").
- **Need:** the server's published OpenAPI (`/openapi/v1.json`) and problem-type registry. (Soft-ish: the web spec is detailed enough to start; exact reconciliation needs the server repo.)

### HB-3 — `/sync/changes` + `/sync/manifest` payload shapes and per-kind `ref` formats
- **What is missing:** The exact request/response bodies and the per-`kind` `ref` formats (`structure`/`blob`/`crdt`/`delete`/`keyrotate`).
- **This is called out as a gap by the spec itself:** web `19 §19.6` lists this as "*the only item still genuinely loose server-side*" (Track & confirm); `OPEN-DECISIONS.md` → "Sync payload shapes / `ref` formats … server-owned; clients code defensively and lock them via shared conformance vectors when the server pins them."
- **Blocks:** the Phase 1.1/1.2 sync engine (web 08 §8.3) to a *final* correctness. Mitigated by "code defensively" + a failing-by-default conformance test, but not fully implementable until the server pins it.
- **Where I looked:** web 08 §8.3, 19 §19.6; OPEN-DECISIONS "Tracked for implementation."
- **Need:** the server's pinned `/sync` payload schemas.

### HB-4 — Shared ink CBOR field schema (Phase 3.1)
- **What is missing:** The **exact deterministic-CBOR field schema** for the ink vector format (pages → strokes → samples + brush metadata). Web `10 §10.5` explicitly states: "The exact field schema is **`[P]` pending the cross-client co-design**." `implementation/phase-3.1` gates the phase on `P3.1-CORE-1` ("Shared ink format co-design done … before any client serialization is committed"), co-designed by **desktop + android** with **web conforming**. `OPEN-DECISIONS.md` lists it under "Tracked for implementation … the only remaining work is desktop + Android co-design of the exact field schema."
- **Blocks:** `P3.1-WEB-2` — you cannot produce a **byte-stable, reproducible BLAKE3 content address** (the whole point) or pass the round-trip conformance test `P3.1-TC-2` without the exact schema. `P3.1-WEB-1` (capture/render) *is* buildable; serialization/interop is not.
- **Where I looked:** web 10 §10.5, 02 §2.8, 19 §19.7; phase-3.1 (P3.1-CORE-1/WEB-2, TC-2); OPEN-DECISIONS.
- **Need:** the finalized shared ink-format field schema + cross-client round-trip vectors.

### HB-5 — Key-transparency inclusion-proof format + safety-number derivation (Phase 4.3)
- **What is missing:** The **transparency-log inclusion/consistency-proof format** and the **safety-number derivation algorithm**. There is **no `specification/` chapter for key transparency in the NyxiteWeb repo at all** — the web roadmap (20) treats "key-transparency / safety-number verification UI" as *optional Phase 6*, while the build plan pulled it into required **Phase 4.3** (`implementation/phase-4.3`, re-sequencing note). The format is defined in `P4.3-CORE-1` (cross-cutting, "define safety-number derivation … and the transparency-log inclusion-proof format; ship cross-client vectors") — i.e., it does not yet exist. The web spec only leaves **seams**: 06 §6.10 and 13 §13.5 defer verification, and 07 §7.11.2 references "Phase 4.3 inclusion proof" without defining it.
- **Blocks:** `P4.3-WEB-1` (verify inclusion proofs, safety-number compare UI) and, transitively, **all of groups** (HB-6).
- **Where I looked:** web 06 §6.10, 07 §7.11.2, 13 §13.5, 20 (Phase 6); phase-4.3 (P4.3-CORE-1/WEB-1, TC-1/2); README re-sequencing note.
- **Need:** the transparency-log design (log structure, inclusion/consistency proof format, verification algorithm), the safety-number derivation spec, the server verification-data endpoints, and cross-client vectors. **None are in either repo.**

### HB-6 — Group-key wire fixtures + enrollment/rotation protocol (Phase 4.4)
- **What is missing:** The **pinned** group-key grant blob shape and DEK-to-group wrapped-key shape, and the group endpoint contracts. Web `06 §6.14` gives a *proposed* grant layout (`group_id|scope_id|member_id|generation|alg_id|hpke_ct`) and `CryptoEngine` additions, but marks the section `[P]` and states the shapes are "pinned in **P4.4-CORE-1**" and conformance-locked — the fixtures (`implementation/phase-4.4`, P4.4-CORE-1/2) are not yet produced. Group `POST /groups`, `/members`, `/keys/rotate`, and the reader-group attachment field are server-owned (P4.4-SRV-1..4) and not in the provided contract (ties to HB-2).
- **Additional dependency:** Phase 4.4 has a **hard entry dependency on Phase 4.3** (decision G-3: enrollment wraps only to transparency-verified keys) — so HB-5 blocks HB-6 outright.
- **Blocks:** `P4.4-WEB-1` (group keygen, transparency-verified enrollment, unwrap/wrap, rotate/re-seal, reader-group auto-wrap, management screen).
- **Where I looked:** web 06 §6.14, 07 §7.11, 13 §13.9, 19 §19.12; features/groups.md; phase-4.4 (CORE-1/2, WEB-1, TC-1..7); OPEN-DECISIONS G-1..G-5.
- **Need:** the pinned group-key wire fixtures + group endpoint contracts + the Phase-4.3 transparency artifact (HB-5).

---

## OPTIONAL-PHASE GAPS (Phase 6 — not required for v1.0.0, undefined by design)

- **OG-1 — Metadata-graph-hiding scheme (Phase 6.2, P6.2-WEB-1).** The encrypted-containment scheme is undefined; `phase-6.2` (P6.2-CORE-1) designs it "if pursued." Web spec has only a seam (03). Not a v1.0.0 blocker; a real gap for "all phases."
- **OG-2 — Leak-free searchable-index scheme (Phase 6.3, P6.3-WEB-1).** Deliberately a go/no-go (`P6.3-CORE-1`); web 11 §11.7 flags it "deferred." Not a v1.0.0 blocker.

---

## MINOR AMBIGUITIES (buildable; would benefit from clarification)

- **M-1 — Chunked/resumable upload contract (Phase 5.1/5.3).** Web 05 §5.6 is referenced but 05 does not actually detail the chunk part-addressing/resume/idempotency contract; it is pinned in `P5.1-CORE-1` (shared base). 5.4 only says `413` → "chunking is Phase 5." Office/image are optional-in-v1.0.0, so minor.
- **M-2 — Office/image viewer approach (Phase 5.1/5.3).** Web editors chapter (10) covers markdown/plaintext/ink/sourcecode; it does **not** specify how office docs or images are rendered/previewed in-browser (10.5 out-of-scope note aside). The `-WEB-` steps focus on blob+chunked upload (covered by seam), but the actual preview/edit UX is unspecified. Optional phase, minor.
- **M-3 — Visual/product design system (dependency on NyxiteDesign).** `OPEN-DECISIONS.md` backlog "Design": "the visual & product design direction is undefined … nothing pins the look-and-feel yet"; the master backlog defers the visual identity to **NyxiteDesign**. Web 15 gives structure, responsive breakpoints, dark-first, shadcn components — enough to build *functionally* with shadcn defaults, but no brand or design tokens pin the look-and-feel. Affects Phase 4.5 polish; not a functional blocker.
- **M-4 — CRDT cross-client wire vectors.** Yjs is the **reference** (web is authoritative for the protocol), so the web client is buildable and self-consistent; but the cross-client vectors against ydotnet/yrs (18 §18.6) still require those clients to fully close the loop. Low risk for building the web client itself.
- **M-5 — Enterprise Keycloak realm specifics.** `client_id`/`authority` are runtime (per-account `config.json`), so this is fine; only the realm/claims mapping details would come from a deployment, resolved at runtime via `config.json` rather than the spec. Minor.
- **M-6 — Audited WASM PQC library selection (hybrid ML-KEM-768 / ML-DSA-65).** WebCrypto and libsodium expose **neither** post-quantum primitive, so the hybrid halves require an audited third-party WASM library. Per `OPEN-DECISIONS.md` this is the one explicitly-open **follow-up dependency** — kept behind the `CryptoEngine` boundary and gated on the shared conformance vectors (HB-1). Needed for Phase 0 crypto. This is a library-selection decision, not a spec gap: once chosen it is buildable, and the abstraction means the choice does not leak into callers. Tracked, not blocking design.

---

## What IS fully specified and build-ready (for balance)

The web specification is unusually complete and internally consistent. All of the following are build-ready from the two repos: the static-export SPA + PWA architecture and layering (01–03), the ESLint crypto boundary, the full Dexie/IndexedDB data model and per-file sync state machine (04), the `CryptoEngine` interface and worker model with every primitive parameter pinned (06–07), the sync engine's two-tier reconcile/outbox/idempotency model (08), the SignalR relay hub contract and Yjs collaboration/awareness/snapshot triggers (09), the CodeMirror+Yjs and markdown-view editors (10), MiniSearch local-subset search (11), client-side version history/diff/restore (12), account+link+guest sharing and rotation-based revocation (13), native auth + WebAuthn + enterprise-OIDC seam + multi-account isolation (14), routing/UI/states (15), offline/PWA/quota (16), the browser threat model + CSP (17), and CI/testing strategy (18). Every web-side open question (19) is ratified. Phases 0–2 — the complete core E2EE web client — are implementable to a high fidelity, with the caveat that the *interop proof* (HB-1) and the *exact server contract* (HB-2/HB-3) are externally owned.

---

## Bottom line

You could build most of NyxiteWeb — essentially the entire Phase 0–2 core client and much of Phase 4 — from these two repos, and to a high standard. As a static-export Next.js 15 SPA + PWA that is itself a full in-browser crypto peer (the primary anonymous guest/share surface, talking to a blind-relay server), NyxiteWeb is buildable **today for Phases 0–2 except for the interop-proof artifacts**. You **cannot fully implement all phases** because six externally-owned or not-yet-produced artifacts are required: the shared crypto conformance vectors + server ledger (HB-1), the exact server REST/OpenAPI contract (HB-2) and its `/sync` payload shapes (HB-3), the shared ink CBOR schema (HB-4), the key-transparency proof format + safety-number derivation (HB-5), and the pinned group-key wire fixtures (HB-6, which also depends on HB-5).

**All six hard blockers are known, intentional cross-client / cross-repo co-design seams — not oversights.** By the plan's own design they are outputs of cross-repo/cross-client co-design (the server owns the wire contracts and ledger; desktop + Android co-author the ink schema and transparency format; web conforms). Every web-side open question (19) is already ratified, which is exactly why the residue is externally owned rather than web-undecided. But — and this is the operative point — **not one of the six is present in the NyxiteWeb repo or in the shared Nyxite info repo today.** Until they are produced and checked in, the byte-for-byte interop gate cannot run and Phases 3, 4.3, and 4.4 cannot be completed correctly.
