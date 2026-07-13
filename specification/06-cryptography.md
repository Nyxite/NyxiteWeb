# 06 — Cryptography

This is the most consequential module (`core-crypto`). It must be **bit-compatible with the server's framing and the other clients' implementations** — Yjs/web is the *reference* for the CRDT wire, but the **crypto must match server, desktop (ydotnet), and Android (Tink)** byte-for-byte: a frame sealed in one client must open in all the others. Everything here mirrors [server 07](https://github.com/Nyxite/NyxiteServer) and the sibling [android 06](https://github.com/Nyxite/NyxiteAndroid).

## 6.1 Posture

The browser is a **full cryptographic peer**, not a thin viewer ([00 §0.2](00-overview.md)). It generates file keys, encrypts/decrypts all content, wraps/unwraps keys, computes content addresses, signs, and verifies — all locally, in a **Web Worker** ([§6.11](#611-the-crypto-worker)). The server is **blind**: it stores ciphertext, wrapped keys, public keys, opaque IDs, sizes, timestamps — nothing that lets it read content. No Next.js server runtime ever touches plaintext or keys ([00 §0.5](00-overview.md), [01 §1.2](01-architecture.md)).

All crypto is reached through one component, `CryptoEngine`, which fronts the worker; repositories never call a primitive directly. This is the only place plaintext becomes ciphertext and back ([01 §1.2](01-architecture.md)).

## 6.2 Algorithm table (must match server exactly)

| Purpose | Algorithm | Web mapping |
|---------|-----------|-------------|
| Content / CRDT / snapshot / name / metadata encryption | **AES-256-GCM** (96-bit nonce, 128-bit tag) — *unchanged, quantum-safe* | **WebCrypto** `SubtleCrypto` `AES-GCM` (keys as non-extractable `CryptoKey`) |
| Public-key wrap (file-key to a member; device/browser enrollment to a pubkey) | **Hybrid HPKE**: **X25519 + ML-KEM-768** KEM (classical DHKEM(X25519, HKDF-SHA256) — RFC 9180 `KEM=0x0020` — **concatenated** with ML-KEM-768) / HKDF-SHA256 (`KDF=0x0001`) / AES-256-GCM (`AEAD=0x0002`); **NIST level 3**; suite `alg_id` **`X25519MLKEM768`** | **hpke-js** (`@hpke/core`) for the classical DHKEM half **+ a WASM PQC KEM** for the ML-KEM-768 half — WebCrypto ships neither ML-KEM nor ML-DSA (see [§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)) — **verify suite IDs** ([§6.4](#64-hpke-wrapping)) |
| Symmetric wrap (recovery blob) | **AES-256-GCM** under an Argon2id-derived key — *unchanged, quantum-safe* | **WebCrypto** `AES-GCM` ([§6.9](#69-recovery-key-derivation), [07 §7.5](07-key-and-device-management.md)) |
| Identity key agreement | **X25519 + ML-KEM-768** (hybrid; classical + PQC concatenated, NIST level 3) | **libsodium-wrappers-sumo** X25519 + a **WASM PQC** ML-KEM-768 lib ([§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)) |
| Signing (updates, key-directory entries, enrollment) | **Ed25519 + ML-DSA-65** (hybrid; both signatures produced and verified, NIST level 3) | **libsodium-wrappers-sumo** Ed25519 + a **WASM PQC** ML-DSA-65 lib ([§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)) |
| Recovery-key derivation | **Argon2id → wrapping key** (m = 64 MiB, t = 3, p = 1; tunable, stored in `recovery_blobs.kdf_params`) | **hash-wasm** (`argon2id`) in the worker |
| Plaintext hashing (content address) | **BLAKE3-256** | **hash-wasm** (`blake3`), streamed |
| CSPRNG (FK + nonce) | OS CSPRNG | **`crypto.getRandomValues`** |

These values are **pinned to the server's canonical ledger** ([server 07 §7.3](https://github.com/Nyxite/NyxiteServer)) and must be byte-identical across clients; the client tracks the server's `07` and **fails CI on any drift** via the shared conformance vectors ([18 §18.5](18-build-ci-testing.md)).

**Post-quantum hybrid from v1.0.0.** Every **asymmetric** seam ships **hybrid classical + PQC** at v1.0.0 (ratified 2026-07-07 — [OPEN-DECISIONS](https://github.com/Nyxite/Nyxite/blob/main/docs/OPEN-DECISIONS.md)): HPKE key wrap / key agreement is **X25519 + ML-KEM-768**, and every signature is **Ed25519 + ML-DSA-65**, both at **NIST level 3**. *Hybrid* means the classical and PQC halves are **concatenated** (concatenated shared secrets / dual signatures), so a break of **either** half alone is non-fatal — this closes the **harvest-now-decrypt-later** exposure on the server's indefinitely-stored wrapped keys. The **symmetric** primitives are **unchanged** — AES-256-GCM, Argon2id, and BLAKE3 are already quantum-safe (only Grover-halved) and **PQC does not touch password hashing**. Because wraps cover only the **small** keys, the hybrid suite rides the existing agility tags (`version`, and the group `alg_id` [§6.14](#614-group-key-wrapping-enterprisefamily-groups-p)); a future primitive change re-wraps keys **without re-encrypting content** ([§6.3](#63-encrypted-object-framing)). **Follow-up dependency [P]:** the ML-KEM-768 / ML-DSA-65 halves require a **WASM PQC library** — WebCrypto exposes neither and libsodium does not provide them — kept behind `CryptoEngine`; the specific library is an open choice ([§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)).

**System rule — HPKE vs AES-256-GCM**: use **(hybrid) HPKE wherever the target is a public key** (file-key wrap to members, device/browser enrollment to a pubkey); use **AES-256-GCM wherever the key is symmetric** (all content + the recovery blob — no PQC dimension, symmetric primitives stay as-is).

## 6.3 Encrypted object framing

Every ciphertext object (blob, snapshot, CRDT update, encrypted name/title, encrypted settings/metadata, awareness) uses the shared frame:

```
magic(4) | version(1) | key_id(16) | nonce(12) | ciphertext(...) | gcm_tag(16)
```

- `magic` — fixed 4-byte Nyxite identifier: ASCII **`"NYXC"`** (`0x4E 0x59 0x58 0x43`).
- `version` — 1-byte frame/crypto-agility version, currently **`0x01`**. The client **refuses to misread a newer version** it does not understand and surfaces an **"update required"** state rather than guessing — the web client is the easiest to silently leave stale, so this guard is load-bearing.
- `key_id` — 16-byte UUID of the **specific file key** used. The server cannot resolve it to a usable key; the client maps it to a local `FileKeyHandle`. `key_id` is 1:1 with the integer `keyGeneration` rotation counter ([04 §4.2](04-local-data-model.md), [07 §7.6](07-key-and-device-management.md)) and is the stable frame/CRDT/version reference.
- `nonce` — 96-bit, **unique per (key, message)**, from `crypto.getRandomValues`; never reused under the same key (a hard GCM invariant). For high-volume CRDT updates under one FK, random 96-bit nonces keep collision probability negligible — documented and tested.
- **AAD** = `magic ‖ version ‖ key_id ‖ file_id(16) ‖ object_kind(1)`, binding the frame to its file and object kind. Decryption supplies the identical AAD; a mismatch fails authentication, preventing cross-context reuse of ciphertext. The `object_kind` enum (1 byte): `blob = 1`, `snapshot = 2`, `crdt = 3`, `name = 4`, `metadata = 5`, `awareness = 6`, `settings = 7`.

**Crypto-agility** is carried entirely by `version` + `key_id`/generation, so primitives and keys rotate without a format break. There is exactly **one** encode/decode path ([§6.12](#612-implementation-rules)).

## 6.4 `CryptoEngine` interface

The engine is an async facade over the crypto worker; every method posts to the worker and awaits a transferred `ArrayBuffer` ([§6.11](#611-the-crypto-worker)).

```ts
type ObjectKind = 1|2|3|4|5|6|7; // blob|snapshot|crdt|name|metadata|awareness|settings

interface CryptoEngine {
  // AES-256-GCM framing (symmetric, FK)
  seal(plaintext: Uint8Array, fk: FileKeyHandle, kind: ObjectKind, fileId: Uuid): Promise<Uint8Array>; // → frame
  open(frame: Uint8Array,  fk: FileKeyHandle, kind: ObjectKind, fileId: Uuid): Promise<Uint8Array>;     // → plaintext

  // HPKE (target is a PUBLIC KEY — hybrid X25519 ‖ ML-KEM-768)
  wrapToPublicKey(payload: Uint8Array, recipientHpkePub: Uint8Array): Promise<Uint8Array>; // → enc‖ct‖tag
  unwrapWithIdentity(wrapped: Uint8Array, identity: IdentityPrivateKeys): Promise<Uint8Array>;

  // Signatures (hybrid Ed25519 + ML-DSA-65)
  sign(message: Uint8Array, sigPriv: Uint8Array): Promise<Uint8Array>;   // hybrid dual detached sig (Ed25519 ‖ ML-DSA-65)
  verify(message: Uint8Array, sig: Uint8Array, sigPub: Uint8Array): Promise<boolean>; // both halves must verify

  // Content addressing
  contentAddress(plaintext: Uint8Array): Promise<Uint8Array>; // BLAKE3-256, 32 bytes

  // Recovery (symmetric, Argon2id → AES-256-GCM)
  deriveRecoveryKey(phrase: string, kdf: KdfParams): Promise<CryptoKey>;       // non-extractable AES-GCM
  sealRecoveryBlob(bundle: IdentityBundle, key: CryptoKey, userId: Uuid, version: number): Promise<RecoveryBlob>;
  openRecoveryBlob(blob: RecoveryBlob, key: CryptoKey, userId: Uuid): Promise<IdentityBundle>;

  // File keys
  generateFileKey(): Promise<FileKeyHandle>; // CSPRNG → non-extractable AES-GCM CryptoKey
}
```

`Uuid` values cross the AAD as their 16 raw bytes (not the hyphenated string), identical to server/desktop/Android.

## 6.5 HPKE wrapping (account shares, recovery transport, enrollment)

- To **share a file to a user** (account share), wrap the FK to the owner, or **enroll a new browser**, the client fetches the recipient's **hybrid HPKE public key** (X25519 + ML-KEM-768) from the directory ([13](13-sharing.md), [07 §7.4](07-key-and-device-management.md)) and HPKE-seals the payload to it. The result is the `wrapped_key` / `wrappedIdentityKey` blob stored server-side; only the recipient can open it with its identity private key.
- **HPKE wrap output framing**: `enc(hybrid encapsulated key) ‖ ciphertext ‖ tag(16)` — matches the server (the `enc` now carries the concatenated X25519 + ML-KEM-768 encapsulation, so its length grows by the ML-KEM-768 ciphertext).
- At v1.0.0 the HPKE KEM is the **hybrid `X25519MLKEM768`** — the classical `DhkemX25519HkdfSha256` below **concatenated** with **ML-KEM-768** — with `HkdfSha256` / `Aes256Gcm` unchanged (NIST level 3). The **classical half must equal** the server's suite:

```ts
import { CipherSuite, Aes256Gcm, HkdfSha256 } from "@hpke/core";
import { DhkemX25519HkdfSha256 } from "@hpke/dhkem-x25519";
const classical = new DhkemX25519HkdfSha256(); // RFC 9180 KEM 0x0020 — the CLASSICAL half only
const suite = new CipherSuite({
  kem:  hybridKem(classical, mlKem768), // X25519MLKEM768 — classical ‖ ML-KEM-768 (PQC half from a WASM PQC lib, §6.13)
  kdf:  new HkdfSha256(),               // KDF 0x0001
  aead: new Aes256Gcm(),                // AEAD 0x0002
});
// base mode, empty info/aad unless the protocol specifies otherwise — locked by vectors
```

- **Hybrid KEM [P].** The ML-KEM-768 half is supplied by a **WASM PQC library** (neither WebCrypto nor hpke-js's default build ships ML-KEM — [§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)); it is combined with the classical `DhkemX25519HkdfSha256` into the `X25519MLKEM768` hybrid KEM. The concrete hybrid-KEM construction (concatenation order, KEM id) is **pinned to the server ledger** and locked by the shared vectors; the library choice is a follow-up, behind `CryptoEngine`.
- **Conformance** ([18 §18.5](18-build-ci-testing.md)): a fixed-vector test **wraps with web and unwraps with server/desktop/Android**, and **wraps with each of those and unwraps with web**. A drift in suite IDs, the hybrid `enc` length, or framing fails the build. If hpke-js defaults ever change, the explicit `CipherSuite` config above pins them.
- **Recovery uses AES-256-GCM, not HPKE** ([§6.9](#69-recovery-key-derivation)) — the recovery key is symmetric, per the system rule (§6.2).

## 6.6 File keys

- An FK is a **256-bit AES-GCM key generated by `crypto.getRandomValues`** when a file is created, imported as a **non-extractable WebCrypto `CryptoKey`** — the `FileKeyHandle` abstraction ([§6.7](#67-filekeyhandle)).
- It encrypts that file's bytes, CRDT updates, snapshots, and encrypted name. It is **never sent unwrapped**; the server holds it only as `wrapped_key` rows (one HPKE-wrapped copy per member) and/or it travels in a link fragment ([13](13-sharing.md)).
- **No convergent encryption** — the FK is random, not derived from content (resists confirmation-of-file attacks). **Intra-file dedup** still works: identical plaintext under the same FK with the same nonce policy yields the same content address, so unchanged versions dedup; **cross-file dedup is impossible** under E2EE and is not a goal.

## 6.7 `FileKeyHandle`

The web tension: WebCrypto AES-GCM **does** support non-extractable keys, and an FK is only ever used for AES-GCM, so an FK can and **must** be a non-extractable `CryptoKey` — the bytes never sit in JS-reachable memory after import.

```ts
interface FileKeyHandle {
  readonly keyId: Uuid;            // = the 16-byte key_id in frames
  readonly generation: number;     // rotation counter
  readonly key: CryptoKey;         // non-extractable AES-GCM, lives in the worker
}
```

- Generation: `crypto.getRandomValues(32)` → `crypto.subtle.importKey('raw', bytes, 'AES-GCM', /*extractable*/ false, ['encrypt','decrypt'])` **inside the worker**; the raw buffer is overwritten immediately after import ([§6.12](#612-implementation-rules)).
- Handles are **short-lived**: held in the worker for the active file(s), dropped on file close, lock, or logout ([07 §7.3](07-key-and-device-management.md)). The *wrapped* form is cached in IndexedDB ([04 §4.2](04-local-data-model.md)); the unwrapped handle is never persisted.
- Because the key is non-extractable, even XSS that reaches the worker cannot read FK bytes — only request `encrypt`/`decrypt` while the handle exists (a real but bounded risk, see [17](17-security.md)).

## 6.8 Content addressing & verify-on-download

- `address = BLAKE3-256(plaintext)`, computed **on the client** via hash-wasm, streamed for large blobs (ink/binary, snapshots) to bound worker memory. The framed ciphertext is stored under that address; the server treats it as opaque and enforces write-once.
- **Verify-on-download**: after `open`, the client recomputes BLAKE3 over the recovered plaintext and **compares it to the claimed address**, constant-time, before trusting the blob. A mismatch is rejected as tampering. This defends against a **malicious or tampering relay** that returns wrong/withheld bytes — the server cannot read content, but it could still try to corrupt or swap ciphertext; the address check (plus GCM auth) catches it.

## 6.9 Recovery-key derivation

- The recovery key is a **BIP39 24-word phrase** shown once ([07 §7.5](07-key-and-device-management.md)). The wrapping key is derived via **Argon2id** with **m = 64 MiB, t = 3, p = 1** (the floor; actual values are read non-secret from `recovery_blobs.kdf_params` so any browser can recover). hash-wasm runs Argon2id **in the worker** off the main thread.
- The derived key (imported as a non-extractable AES-GCM `CryptoKey`) seals the **identity bundle** (the hybrid identity private keys: X25519 ‖ ML-KEM-768 ‖ Ed25519 ‖ ML-DSA-65) with **AES-256-GCM** — not HPKE (symmetric key, §6.2). The recovery Argon2id → AES-256-GCM path is **unchanged and un-peppered** — the pepper is an account-auth server secret only ([14 §14.1](14-authentication.md)). Blob shape:

```ts
interface RecoveryBlob {
  version: number;
  kdf: { alg: "argon2id"; m: number; t: number; p: number; salt: Uint8Array /*16*/ };
  nonce: Uint8Array; /*12*/ ciphertext: Uint8Array; tag: Uint8Array; /*16*/
}
```

with **AAD = `userId ‖ version`**. Stored/fetched via `PUT`/`GET /recovery`; the server stores it opaquely and **cannot open it** ([07 §7.5](07-key-and-device-management.md)). There is **no server escrow** — losing all browsers and the phrase is permanent loss.

## 6.10 Signing & verifying

- The client **signs** its key-directory entry (its published X25519+ML-KEM-768 / Ed25519+ML-DSA-65 public keys) and, where the protocol calls for it, CRDT updates / share grants with the **hybrid Ed25519 + ML-DSA-65** signature (libsodium Ed25519 + a WASM PQC ML-DSA-65 lib, [§6.2](#62-algorithm-table-must-match-server-exactly) / [§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)), so peers can detect a relay that swaps keys or injects updates.
- It **verifies** those hybrid signatures on directory entries it fetches **before wrapping a share to them** ([13](13-sharing.md)), and on relayed updates where signed — **both** the Ed25519 and the ML-DSA-65 half must verify. Full key-transparency / safety-numbers is deferred to Phase 6; v1.0.0 trust is **TLS + signature checks** ([00 §0.4](00-overview.md)).

## 6.11 The crypto worker

CPU-bound crypto must not block the React main thread ([01 §1.6](01-architecture.md)):

- A dedicated **Web Worker** hosts the WASM modules — **hash-wasm** (BLAKE3, Argon2id), **libsodium-wrappers-sumo** (classical X25519, Ed25519), a **WASM PQC library** (ML-KEM-768, ML-DSA-65 — [§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)), **hpke-js** — plus bulk **WebCrypto AES-GCM** seal/open. WebCrypto's `SubtleCrypto` is available in workers.
- The worker is the **sole holder** of unwrapped key material at runtime: `FileKeyHandle` `CryptoKey`s and the unwrapped identity bundle live there, not on the main thread. The main-thread `CryptoEngine` is a thin RPC proxy.
- **Transfer `ArrayBuffer`s** (zero-copy) for plaintext/ciphertext to avoid duplicating sensitive buffers across the postMessage boundary; the transferred buffer is detached on the sender side.
- Argon2id (seconds, 64 MiB) and large-blob BLAKE3/AES-GCM run here so the UI stays responsive; diff for history may share this worker or a sibling ([12](12-version-history.md)).

## 6.12 Implementation rules

- **One code path** for framing (`seal`/`open`) used by every content type and every transport (REST blob, relay update, snapshot, name, settings). No ad-hoc encryption elsewhere; the ESLint boundary rule blocks any network module from importing crypto ([01 §1.2](01-architecture.md)).
- **No homemade crypto.** Only the vetted libraries in [02 §2.6](02-tech-stack-and-libraries.md), behind the `CryptoEngine` interface for agility.
- **Constant-time comparisons** for tags, tokens, and content-address checks (libsodium `sodium.memcmp`); never use `===`/byte loops for secrets.
- **Zeroization is best-effort in JS** — there is **no reliable way to wipe memory** the GC controls, and `String`/many typed arrays may be copied by the engine. Mitigations, applied consistently:
  - Prefer **non-extractable `CryptoKey` handles** (FK, vault key, recovery key) so raw bytes never live in JS arrays at all.
  - For the unavoidable raw bytes (identity private bundle for libsodium; FK bytes pre-import; the BIP39 phrase), keep them **short-lived in `Uint8Array`** and **overwrite with `.fill(0)`** immediately after use; never place them in `String`s, `localStorage`, `sessionStorage`, the URL, logs, or telemetry.
  - **Drop on lock/logout/switch**: discard `FileKeyHandle`s and the in-memory identity bundle, tear down the worker if needed ([07 §7.2](07-key-and-device-management.md), [07 §7.8](07-key-and-device-management.md)).
  - Document the limitation in [17](17-security.md); it is an accepted browser constraint, not a defect.
- **Known-answer tests** for AES-GCM framing, HPKE wrap/unwrap, Ed25519, X25519, BLAKE3, and Argon2id, **plus cross-client conformance vectors** shared with server/desktop/Android ([18 §18.5](18-build-ci-testing.md)). **This is the single most important test surface in the app** — a divergence here silently corrupts user data across clients.

## 6.13 Browser-specific notes (WebCrypto availability & library choice) **[P]**

- **Secure context required.** `crypto.subtle` is only present in secure contexts (HTTPS, or `localhost` in dev) and in worker contexts; a browser without it gets the unsupported-browser screen ([00 §0.6](00-overview.md)), never a degraded path. The crypto worker confirms `self.crypto?.subtle` and `WebAssembly` at startup.
- **Performance.** WebCrypto AES-GCM is typically hardware-accelerated and fast for bulk content; Argon2id (64 MiB) is the only multi-second op and is why it lives in the worker. BLAKE3 via WASM streams large blobs.
- **Why libsodium for X25519/Ed25519, not WebCrypto.** Native WebCrypto support for `X25519` and `Ed25519` is **uneven across the target browser/version matrix** (Safari and older Firefox/Chromium lag and behave inconsistently). To guarantee one **bit-identical** implementation everywhere — and to match the suites the other clients use — the web client standardizes on **libsodium-wrappers-sumo** (WASM) for both, and uses hpke-js (which itself uses X25519) for the classical half of the public-key wrap. WebCrypto is used **only** for AES-GCM, where support is universal and acceleration matters.
- **Post-quantum halves need a WASM PQC library [P].** The hybrid suites ([§6.2](#62-algorithm-table-must-match-server-exactly)) add **ML-KEM-768** (the PQC half of the HPKE KEM) and **ML-DSA-65** (the PQC half of every signature) alongside the classical X25519/Ed25519. **Neither WebCrypto nor libsodium exposes ML-KEM or ML-DSA**, so the PQC halves come from a dedicated **audited WASM PQC library** loaded in the crypto worker; the classical halves stay on libsodium and the AEAD stays on WebCrypto. **The specific library is an open follow-up/dependency** — it is kept behind `CryptoEngine` and gated on passing the shared conformance vectors, exactly like the classical libs. Selection criteria: audited, constant-time, self-hosted WASM (no CDN, per [17](17-security.md)/[18](18-build-ci-testing.md)), and **byte-compatible** with the server/desktop/Android ML-KEM-768 / ML-DSA-65 outputs and the `X25519MLKEM768` hybrid KEM. This adds a WASM module and modestly larger wrapped-key / signature sizes; symmetric performance is unaffected.

## 6.14 Group-key wrapping (enterprise/family groups) **[P]**

Group sharing ([13 §13.9](13-sharing.md), [07 §7.11](07-key-and-device-management.md), [features/groups.md](https://github.com/Nyxite/Nyxite)) inserts a **middle key layer** into the envelope hierarchy so a whole group reads a file from **one** DEK-to-group wrap instead of one wrap per member. Build step **P4.4-WEB-1**; enrollment depends on key transparency ([07 §7.11](07-key-and-device-management.md), Phase 4.3).

```
personal key  →  wraps  →  group key  →  wraps  →  DEK (FK)  →  encrypts  →  file
```

- **No new primitive.** A group public key is **just another (hybrid) HPKE target**, exactly like a member or browser-enrollment public key — the same suite the rest of §6 pins. No new algorithm and no group-special WASM module: the group crypto reuses the same worker-hosted stack — **hpke-js** (`@hpke/core` + `@hpke/dhkem-x25519`) plus the **WASM PQC library** for the ML-KEM-768/ML-DSA-65 halves ([§6.13](#613-browser-specific-notes-webcrypto-availability--library-choice-p)) for the wrap, and **libsodium-wrappers-sumo** (+ the same PQC lib) for the group signing keypair — that [§6.2](#62-algorithm-table-must-match-server-exactly) already pins for every wrap ([§6.11](#611-the-crypto-worker)). All group wrap/unwrap runs **in the worker**, never on the main thread and never in any Next.js server runtime.
- **Group keypair.** `X25519 + ML-KEM-768` (hybrid HPKE recipient) + `Ed25519 + ML-DSA-65` (hybrid signature over the group's directory entry / grants), generated in the worker at group creation. The **public** halves are published; the **private** halves are stored **only wrapped**, once per member.

**Group-key grant blob (`group_privkey` HPKE-wrapped to a member).** The group private key is HPKE-sealed to a member's **hybrid HPKE public key** (X25519 + ML-KEM-768) using **exactly** the §6.5 suite and `enc ‖ ct ‖ tag` framing; the grant record wraps that ciphertext with routing/agility metadata (pinned in **P4.4-CORE-1**):

```
group_id(16) | scope_id(16) | member_id(16) | generation(4) | alg_id(2) | hpke_ct( enc(32) ‖ ct ‖ tag(16) )
```

- **`alg_id`** (2 bytes) tags the wrap algorithm on **every** group blob for crypto-agility: the group key wraps *many* DEKs, concentrating post-quantum blast radius, so v1 ships the **hybrid** wrap **from day one** and keeps the tag for future rotations (`alg_id` = **`X25519MLKEM768`**, the pinned hybrid X25519+ML-KEM-768 / HKDF-SHA256 / AES-256-GCM suite, [§6.2](#62-algorithm-table-must-match-server-exactly)). The client **refuses** an `alg_id` it does not implement and surfaces "update required" ([§6.3](#63-encrypted-object-framing)), never guessing.
- **Grants are append-only:** rotation bumps `generation` and appends a new row rather than mutating, preserving an auditable "who could read what when" trail. The blob carries **no plaintext key material** — the server stores opaque bytes only.

**DEK-to-group wrap.** To let a group read a file, the client HPKE-wraps that file's **FK** to the **group public key** (fetched from the directory, verified per [§6.10](#610-signing--verifying)) and stores one wrapped-key row whose **principal is a group** (per scope/generation), alongside the existing per-member wraps ([13 §13.9](13-sharing.md)). Same `enc ‖ ct ‖ tag` output; same `alg_id` tag.

**Unwrap chain (read path).** A member's browser:
1. HPKE-**unwraps the group private key** from its grant using the **identity HPKE private key** (hybrid X25519 + ML-KEM-768) (`unwrapWithIdentity`) — the group privkey surfaces as raw bytes held **in the worker only**, `.fill(0)` on drop ([§6.12](#612-implementation-rules)).
2. HPKE-**unwraps the file DEK** from the DEK-to-group row using that group private key → a non-extractable `FileKeyHandle` ([§6.7](#67-filekeyhandle)).
3. `open`s content as usual (§6.3). The group privkey is never persisted unwrapped and never leaves the worker.

**`CryptoEngine` additions** (worker-fronted, mirroring §6.4):

```ts
// Group keygen (hybrid: libsodium X25519/Ed25519 + WASM PQC ML-KEM-768/ML-DSA-65, in the worker)
generateGroupKeypair(): Promise<GroupKeyMaterial>; // {hpkePub, sigPub} out (hybrid X25519‖ML-KEM-768 / Ed25519‖ML-DSA-65); private halves stay in-worker

// Wrap the group PRIVATE key to a member (HPKE, target = member hybrid HPKE pub) → grant hpke_ct
wrapGroupKeyToMember(groupPriv: GroupPrivHandle, memberHpkePub: Uint8Array): Promise<Uint8Array>;

// Unwrap the group private key from a grant with the caller's identity (HPKE)
unwrapGroupKey(grantHpkeCt: Uint8Array, identity: IdentityPrivateKeys): Promise<GroupPrivHandle>;

// Wrap a file DEK to a group PUBLIC key (HPKE, target = group hybrid HPKE pub)
wrapDekToGroup(fk: FileKeyHandle, groupHpkePub: Uint8Array): Promise<Uint8Array>;

// Unwrap a DEK-to-group row with an already-unwrapped group private key (HPKE) → FileKeyHandle
unwrapDekWithGroup(wrapped: Uint8Array, groupPriv: GroupPrivHandle): Promise<FileKeyHandle>;
```

- `wrapGroupKeyToMember` / `wrapDekToGroup` are the same `wrapToPublicKey` call ([§6.4](#64-cryptoengine-interface)) pointed at a member vs a group public key — one HPKE code path, no group-special primitive.
- **Conformance ([18 §18.5](18-build-ci-testing.md)):** the group-key grant and DEK-to-group shapes are in the shared vectors — a group blob wrapped in web must unwrap in server/desktop/Android and vice-versa; `alg_id`, `enc` length, and framing drift fails the build (**P4.4-CORE-1**). No group fixture carries an unwrapped key.
