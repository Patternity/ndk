# NDK Rationale

Non-normative. This document explains *why* NDK-1 is shaped the way it is, and records
the alternatives that were considered and rejected. The normative rules are in
[SPEC.md](SPEC.md).

## 1. Why anything beyond SLIP-0021?

SLIP-0021 already gives labeled, domain-separated symmetric derivation. A skeptical
reading is correct that NDK adds **no cryptography**. NDK earns its place only if the
following are worth standardizing:

1. **Deterministic context → bytes.** SLIP-0021 says "a label is bytes" but not how a
   human context (project, purpose, network) becomes *those exact* bytes. Two well-meaning
   tools diverge on case, order, Unicode form, or a trailing slash. NDK fixes this.
2. **A public, validated recovery descriptor** with stable error codes.
3. **A named seed→key transform** so one seed cannot silently become different keys in
   different tools (the interop gap that opaque "profiles" leave open).

If a user needs none of these, they should use SLIP-0021 directly. NDK does not pretend
otherwise. This honesty is deliberate: the recommendation for NDK is a *convention plus
vectors plus a small utility*, not a heavyweight standard.

## 2. Field model: `subject` / `purpose` / `profile` / `type`

* **`subject` stays free-form and human-readable.** An earlier draft considered forcing a
  stable anchor (DID/UUID/numeric id) to survive renames. Rejected: it destroys the
  entire value proposition — legible contexts. Instead, subject stability is the
  operator's responsibility, documented in [SECURITY.md](SECURITY.md). NDK's job is to be
  deterministic about whatever string it is given, and to refuse to silently alter it.
* **`purpose` = stable role**, not a transient action. Different independent uses SHOULD
  use different purposes so they derive different seeds.
* **`profile` = downstream context** (network/coin/variant), opaque to the core.
* **`type` = the transform** (how the seed becomes a key). Added because a bare
  32-byte seed under an opaque `profile` is the central interoperability hazard: the same
  seed can be interpreted as a raw scalar, a BIP-32 seed, or an Ed25519 seed, yielding
  different addresses. Making the transform explicit and part of derivation closes this.

### Why `type` is a derivation label, not downstream metadata

If `type` did not affect the seed, one 32-byte value could feed several transforms →
cross-protocol key reuse (a real footgun). As a label, each transform gets a distinct
seed, and reuse is structurally impossible.

### Why the transforms live in a separate document

If the enum of transforms were hardcoded in the core, adding `x25519-seed` would be a
core change → a new NDK major version → new root label → all vectors invalidated. By
validating `type` only syntactically in the core and defining transforms in the
independently-versioned [NDK Profiles](SPEC-PROFILES.md), the core can be frozen while
transforms grow. This also keeps NDK from sliding into being a wallet standard: the
transforms are generic (RFC 5869 / BIP-32 / RFC 8032 / SEC 1), not blockchain-specific.

## 3. Key rotation via a profile generation suffix

Rotation is expressed as a trailing `:N` on `profile` (e.g. `dash:mainnet:2`), not as a
separate descriptor field. Rationale: the core treats `profile` as opaque bytes, so a
bumped suffix simply yields a new seed with no parser changes and no extra field. An
earlier option added a dedicated `generation` integer field; it was dropped to keep the
descriptor at five fields and avoid a second version axis. Downstream consumers that read
network parameters from `profile` must ignore the trailing generation segment.

## 4. Label format keeps the `key:` prefixes

`subject:github:...` is redundant with the fixed label position and produces a double
colon. It is kept deliberately, not by inertia: the prefixes make intermediate labels
self-describing on constrained hardware-wallet displays and guard against positional
shift if a future major version reorders or inserts fields. The "first colon separates
key from value" rule makes the double colon unambiguous.

## 5. Canonicalization: reject, do not normalize

For `subject`, non-NFC input is **rejected**, never silently normalized. Silent NFC
folding would make the seed depend on which platform/library performed normalization,
breaking reproducibility across languages. Rejecting keeps derivation a pure function of
the bytes the user actually wrote. `purpose`/`profile`/`type` are ASCII by regex, so the
question does not arise for them. URLs are not normalized either (no case folding, no
trailing-slash changes, no percent-decoding).

## 6. Output length: exactly 32 bytes, no HKDF in the core

Downstream needs (Ed25519 seed, secp256k1 scalar, BIP-32 seed, symmetric key) are all
satisfied by ≤32 bytes or by expansion performed downstream. Adding variable length or
HKDF-expansion to the *core* would be a new construction without a proven need. Callers
who want more bytes use the `hkdf-sha256` transform in the Profiles layer. The core
returns `node[32:64]` only, never the full node, so the chain key never leaks.

## 7. Naming

The acronym "NDK" collides with the Android NDK and with Nostr's NDK library, and the
unscoped npm name `ndk` is taken. Consequence: poor search discoverability for the bare
string. The *derivation label* `ndk:1` is nonetheless retained, because renaming it is a
protocol change that invalidates every vector, and the label is a domain tag, not a
brand. The npm package is therefore published under a scope
(`@lexxxell/named-deterministic-keys`), and documentation distinguishes the `ndk:` label
from Android/Nostr NDK. A full rename was rejected: the discoverability gain does not
justify breaking cryptographic compatibility.

## 8. Rejected alternatives (summary)

| Alternative | Verdict |
|---|---|
| Use SLIP-0021 directly, no spec | Viable if you need no shared structure/vectors; NDK targets those who do |
| Descriptor labels without `key:` prefixes | Rejected: loses self-describing display and positional-shift safety |
| Silent NFC normalization | Rejected: breaks cross-language reproducibility |
| Hardcoded transform enum in the core | Rejected: makes transforms a core-version concern |
| Opaque `profile` with no `type` | Rejected: leaves the same-seed-different-key interop gap open |
| Dedicated `generation` field | Rejected: profile suffix achieves rotation without a new field |
| Mandatory stable `subject` anchor | Rejected: kills human-readability, the core value |
| Variable-length / HKDF core output | Rejected: unnecessary new construction; use the hkdf transform |
| Full project rename away from "NDK" | Rejected: breaks `ndk:1` label and all vectors |

## 9. Open questions (block `1.0`)

1. Does NDK Profiles need a **registry** (ownership-namespaced, reverse-DNS) or is a
   documentation convention enough? A registry improves interop but adds governance.
2. Is there **evidence of real-world demand** beyond the author? NDK should not become a
   standard around an attractive idea without users.
3. **Duplicate-key handling** must be confirmed across strict JSON parsers in both Node
   and browsers (the core relies on a text-layer check).
4. Should `bip32-seed` pin a derivation **path** per profile, or leave it to the
   application? Pinning improves key-interop but expands scope toward a wallet standard.
5. Is the `subject` limit (1024 UTF-8 bytes) appropriate, or should it be smaller?
