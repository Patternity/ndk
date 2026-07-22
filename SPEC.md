# NDK-1: Named Deterministic Keys

## Deterministic seed derivation from named contexts

```text
Status:              Draft
Draft revision:      1.0
Specification owner: Patternity
Reference impl owner: LexxXell
Depends on:          SLIP-0021
```

> This document is the normative specification. Rationale, rejected alternatives,
> threat analysis and security guidance live in companion documents
> (`RATIONALE.md`, `THREAT-MODEL.md`, `SECURITY.md`). The seed→key transforms named by
> the `type` field are defined in the separate, independently-versioned
> [NDK Profiles specification](SPEC-PROFILES.md).

---

## 1. Abstract

NDK defines a compact, human-readable JSON descriptor and a deterministic method for
deriving a 32-byte seed from a master secret using SLIP-0021.

NDK does **not** introduce a new key derivation function. It standardizes:

* the descriptor structure and its validation rules;
* the conversion of descriptor fields into ordered SLIP-0021 labels;
* label ordering, encoding, and output length;
* interoperability test vectors.

The complete NDK-1 model is:

```text
master secret + NDK descriptor
    -> ordered SLIP-0021 labels
    -> 32-byte NDK seed
```

How the 32-byte seed becomes a downstream key (a private key, wallet seed, signing key,
symmetric key, …) is defined by a **profile transform**, named in the descriptor's
`type` field and specified in [NDK Profiles](SPEC-PROFILES.md). The NDK core stops at
the 32-byte seed.

---

## 2. Scope

This specification is authoritative for:

* descriptor validation;
* label compilation and ordering;
* the byte-exact NDK seed.

This specification is **not** authoritative for the cryptographic HMAC construction
(SLIP-0021 is — see §17) nor for seed→key transforms ([NDK Profiles](SPEC-PROFILES.md)
is).

---

## 3. Conformance terminology

The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `SHOULD`,
`SHOULD NOT`, `MAY`, and `OPTIONAL` are to be interpreted as described in RFC 2119 and
RFC 8174 when, and only when, they appear in all capitals.

---

## 4. Goals

1. Deterministic seed derivation from a named, human-readable context.
2. Interoperability between independent implementations, backed by test vectors.
3. Separation of concerns between *who* (`subject`), *why* (`purpose`), *where*
   (`profile`), and *how* (`type`).
4. Independence from JSON property order and formatting.
5. A small specification built on an existing cryptographic standard.
6. A descriptor that can be stored publicly and used for recovery.

## 5. Non-goals

NDK-1 does **not** define: generation of the master secret; mnemonic phrases; BIP-32
paths; wallet account structures; address generation; transaction construction; signing
or encryption algorithms; network communication; secret storage; a profile registry;
or conversion of the result to lengths other than 32 bytes.

The seed→key transforms are defined by [NDK Profiles](SPEC-PROFILES.md), a separate
document, so that the NDK core can remain frozen while transforms evolve.

---

## 6. Terminology

* **Master secret** — a secret byte sequence supplied to SLIP-0021. NDK does not define
  its origin, representation, or exact length. Security of every derived seed depends on
  the entropy of the master secret.
* **Descriptor** — a JSON object containing the named derivation context. Exactly five
  properties (§7).
* **Label** — a UTF-8 string of the form `key:value` used to derive a SLIP-0021 child
  node.
* **NDK seed** — the final 32-byte node key produced after processing all labels. Opaque
  seed material; NDK assigns it no cryptographic algorithm or object type.
* **Profile transform** — the algorithm that converts the 32-byte NDK seed into a
  downstream key, named by `type` and defined in [NDK Profiles](SPEC-PROFILES.md).

---

## 7. Descriptor format

A valid NDK-1 descriptor has exactly these five properties:

```json
{
  "ndk": 1,
  "subject": "github:Patternity/ndk",
  "purpose": "donations",
  "profile": "dash:mainnet",
  "type": "bip32-seed"
}
```

* All property names MUST be lowercase and MUST be exactly the five listed.
* Additional properties MUST be rejected (§10, §16).
* JSON property order MUST NOT affect derivation (§9).

Applications requiring metadata SHOULD place the descriptor inside an external wrapper;
the wrapper is not part of NDK derivation:

```json
{
  "descriptor": { "ndk": 1, "subject": "...", "purpose": "...", "profile": "...", "type": "..." },
  "metadata": { "name": "Example donations", "description": "Human label, not derived" }
}
```

---

## 8. Field validation

An implementation MUST reject a descriptor whose fields violate any rule below. Values
that pass validation MUST be preserved byte-for-byte: NDK MUST NOT change case, trim,
re-encode, or normalize a value (§11).

### 8.1 `ndk`

* MUST be the JSON integer `1`.
* The strings `"1"`, the number `1.0`, and the integers `0` and `2` are invalid.
* Compiled value: the ASCII decimal string `1` (no sign, leading zeros, or decimal
  point).

### 8.2 `subject`

Identifies the entity the seed is derived for. Free-form and human-readable by design.

* MUST be a string.
* MUST contain at least one `:`. The prefix before the first `:` MUST match
  `^[a-z][a-z0-9+.-]*$`.
* MUST be 1..1024 bytes after UTF-8 encoding.
* MUST be in Unicode NFC.
* MUST NOT contain NUL or any Unicode control character (C0 `U+0000..U+001F`, DEL
  `U+007F`, or C1 `U+0080..U+009F`).
* MUST NOT begin or end with whitespace. Internal spaces are permitted.

NDK MUST NOT silently normalize the subject: it MUST NOT change case, add or remove a
trailing slash, normalize a URL, decode percent-encoding, resolve redirects, or verify
that the referenced resource exists. The following are therefore distinct subjects that
MUST produce different seeds:

```text
github:Patternity/ndk
github:patternity/ndk
github:Patternity/ndk/
```

> The stability of the subject string is the operator's responsibility, not NDK's. A
> mutable name (e.g. a repository name that may be renamed or transferred) will produce
> a different seed if it changes. See `SECURITY.md` §"Subject stability".

### 8.3 `purpose`

Describes the stable role the seed plays.

* MUST match `^[a-z0-9]+(?:-[a-z0-9]+)*$`.
* MUST be 1..64 ASCII characters.

### 8.4 `profile`

Identifies the downstream context (network / coin / variant) in which the seed is
consumed. Opaque to NDK derivation.

* MUST match `^[a-z0-9]+(?:[.:-][a-z0-9]+)*$`.
* MUST be 1..128 ASCII characters.

A trailing `:N` **generation** segment MAY be used for key rotation by convention (e.g.
`dash:mainnet:2`). NDK treats the whole string as opaque bytes; a bumped generation
simply yields a different seed. Downstream consumers that parse the profile for network
parameters MUST ignore a trailing generation segment.

### 8.5 `type`

Names the seed→key transform (§NDK Profiles). Opaque to the NDK core.

* MUST match `^[a-z0-9]+(?:-[a-z0-9]+)*$`.
* MUST be 1..64 ASCII characters.
* The NDK core MUST NOT hardcode the set of valid transform names. The 32-byte NDK seed
  is derived for **any** syntactically valid `type`. Whether a `type` denotes a known
  transform is decided at the profile layer, not the derivation layer. An implementation
  that cannot turn the seed into a key because the transform is unknown MUST fail at the
  transform step, not at seed derivation.

---

## 9. JSON processing and property order

* An implementation MUST read fields by property name and compile labels in the
  normative order (§12). It MUST NOT derive labels by iterating over JSON properties in
  source order.
* JSON indentation, whitespace, line endings, and property order MUST NOT affect the
  seed. These two descriptors are equivalent:

```json
{ "ndk": 1, "subject": "s", "purpose": "p", "profile": "pr", "type": "t" }
```
```json
{ "type": "t", "profile": "pr", "purpose": "p", "subject": "s", "ndk": 1 }
```

---

## 10. Duplicate-key policy

A JSON object with a duplicate property name (e.g. two `purpose` keys) is
parser-dependent and MUST NOT be accepted with silent last-wins/first-wins behavior.

Because duplicate keys exist only in JSON *text* — a materialized JavaScript/JSON object
has already collapsed them — this rule applies to **JSON text validation**:

* An implementation that accepts JSON **text** MUST reject text containing a duplicate
  property name at any object level, with error code `duplicate-property`.
* An implementation whose core API accepts an already-materialized object MAY rely on a
  strict text parser at its input boundary (file/stdin/network) and document that the
  object API assumes text-level checks were already performed.

The reference implementation takes the second approach: the core `validateDescriptor`
operates on objects; strict duplicate-key detection is applied when parsing text.

---

## 11. Unicode policy

* `purpose`, `profile`, and `type` are restricted to ASCII by their regular expressions.
* `subject` MAY contain non-ASCII UTF-8 and MUST be in Unicode NFC. Non-NFC input MUST
  be **rejected**, not silently normalized (error code `not-nfc`). This keeps derivation
  reproducible across languages and platforms and prevents silent descriptor drift.
* NDK does not defend against visually confusable characters (homoglyphs) beyond the NFC
  requirement; see `THREAT-MODEL.md`.

---

## 12. Label compilation and ordering

Each field compiles to exactly one label `key:value`. The label order is fixed:

```text
1. ndk
2. subject
3. purpose
4. profile
5. type
```

Label keys MUST be lowercase and match `^[a-z][a-z0-9-]*$`. The colon after the key is
part of the label; when parsing a label, only the **first** colon separates key from
value (so `profile:dash:mainnet` is key `profile`, value `dash:mainnet`). Each label is
encoded independently as UTF-8 and MUST NOT contain a terminating NUL. The labels MUST
NOT be concatenated before derivation.

Normative pseudocode:

```text
function compileLabels(descriptor):
    validate(descriptor)
    return [
        "ndk:"     + decimalInteger(descriptor.ndk),
        "subject:" + descriptor.subject,
        "purpose:" + descriptor.purpose,
        "profile:" + descriptor.profile,
        "type:"    + descriptor.type
    ]
```

The compiler MUST NOT depend on JSON order, sort labels, change case, trim, normalize,
omit a label, or add implementation-specific labels.

---

## 13. SLIP-0021 derivation

NDK-1 uses SLIP-0021 without modification.

```text
function deriveNdkSeed(masterSecret, descriptor):
    labels = compileLabels(descriptor)
    node = HMAC-SHA512(key = UTF8("Symmetric key seed"), message = masterSecret)
    for label in labels:
        node = HMAC-SHA512(key = node[0:32], message = 0x00 || UTF8(label))
    return node[32:64]
```

`node[0:32]` is the child chain key; `node[32:64]` is the extracted key. The result MUST
be exactly 32 bytes (256 bits). The implementation MUST return the last 32 bytes only,
never the full node, so the chain key does not leak.

---

## 14. Output semantics

The NDK seed is opaque deterministic seed material. NDK does not claim it is directly a
private key, wallet seed phrase, BIP-32 extended key, Ed25519 private key, encryption
key, or authentication credential.

To obtain a downstream key, the seed is passed to the transform named by `type`
([NDK Profiles](SPEC-PROFILES.md)). Two implementations are NDK-interoperable if they
derive the same 32 bytes; they are additionally *key*-interoperable only if they apply
the same profile transform — which is why `type` is part of the descriptor and part of
derivation.

---

## 15. Descriptor fingerprint (optional, non-normative for the seed)

Implementations MAY expose a **fingerprint** to compare descriptors without revealing
the seed. It is defined as SHA-256 over the compiled labels joined by `U+000A` (newline
is a forbidden label character, so the join is unambiguous), truncated to the first 8
bytes and rendered as lowercase hex:

```text
fingerprint(descriptor) = hex( SHA-256( labels.join("\n") )[0:8] )
```

The master secret is not involved. The fingerprint is safe to log. It is not part of the
NDK seed and MUST NOT be used as key material.

---

## 16. Error behavior

Implementations SHOULD return structured errors of the form
`{ "path": "/purpose", "code": "invalid-format", "message": "..." }`. Messages are
non-normative; **codes SHOULD remain stable across patch releases**. The normative code
set:

| Code                 | Meaning                                             |
|----------------------|-----------------------------------------------------|
| `not-an-object`      | root value is not a JSON object                     |
| `invalid-json`       | input is not valid JSON (text layer)                |
| `duplicate-property` | duplicate property name in JSON text (text layer)   |
| `unknown-property`   | a property other than the five allowed              |
| `missing`            | a required property is absent                       |
| `invalid-type`       | a property has the wrong JSON type                  |
| `invalid-value`      | `ndk` is present but not the integer `1`            |
| `invalid-length`     | a value violates its length bounds                  |
| `invalid-format`     | a value violates its regular expression             |
| `not-nfc`            | a string is not in Unicode NFC                      |
| `control-char`       | a string contains NUL or a control character        |
| `edge-whitespace`    | a string begins or ends with whitespace             |
| `missing-scheme`     | `subject` has no `:`                                |
| `invalid-scheme`     | `subject` scheme prefix violates its regex          |

An implementation MUST NOT return a seed for an invalid descriptor.

---

## 17. Versioning

The `ndk` property is the major version and compiles into the root label `ndk:1`. A
backward-incompatible change MUST use a new integer and root label (`ndk:2` → `ndk:2`). A
new major version is REQUIRED when changing: descriptor fields, label keys, label order,
label encoding, normalization rules, the derivation algorithm, output length, or the
interpretation of a core field. Editorial changes do not require a new major version.

Adding or changing a **profile transform** does NOT require a new NDK major version,
because transforms live in the independently-versioned [NDK Profiles](SPEC-PROFILES.md)
document and `type` is validated only syntactically by the core.

---

## 18. Extensions

NDK-1 does not permit custom core fields. An application needing more information MAY:
place the descriptor in an external wrapper; encode downstream context in `profile`;
use a distinct `purpose`; define a new `type` transform in NDK Profiles; or propose a
future NDK major version. An implementation MUST NOT derive labels from unknown JSON
properties.

---

## 19. Security considerations (summary)

The descriptor is public; security MUST rest on the master secret, not on the secrecy of
`subject`/`purpose`/`profile`/`type`. NDK adds no entropy: a guessable value MUST NOT be
used as a master secret. NDK provides context separation, **not** compromise isolation —
compromise of the master secret exposes every derivable seed. Production code MUST NOT
log the master secret, node data, intermediate keys, the NDK seed, or downstream keys.
Full analysis: `SECURITY.md` and `THREAT-MODEL.md`.

---

## 20. Privacy considerations

Different descriptors produce different seeds, but this is not anonymity. Derived
accounts may still be linked via funding sources, transaction timing, UTXO
consolidation, common change addresses, public documentation, or user behavior. NDK
provides deterministic context separation, not privacy protection.

---

## 21. Conformance requirements

A conforming NDK-1 implementation MUST:

* validate the five descriptor properties per §8;
* reject additional core properties, missing properties, and wrong types;
* apply the JSON, duplicate-key, and Unicode policies (§9–§11);
* compile exactly five labels in the normative order (§12);
* preserve value bytes after validation;
* use UTF-8 and SLIP-0021 (§13) correctly;
* return exactly 32 bytes;
* pass all normative vectors in `vectors/ndk-1.json` and reject all entries in
  `vectors/invalid-descriptors.json` with the stated codes;
* pass the official SLIP-0021 vectors embedded in `vectors/ndk-1.json`.

A conforming implementation MAY expose convenience APIs, encodings, wrappers, a
fingerprint, and profile transforms, provided they do not change the NDK seed.

---

## 22. Test vectors

Normative machine-readable vectors are in [`vectors/ndk-1.json`](vectors/ndk-1.json)
(valid) and [`vectors/invalid-descriptors.json`](vectors/invalid-descriptors.json)
(invalid, with expected error codes). The JSON Schema in
[`schemas/ndk-1.schema.json`](schemas/ndk-1.schema.json) performs structural validation;
semantic rules that JSON Schema cannot enforce (UTF-8 byte length, NFC, duplicate keys)
are covered by §8–§11.

Baseline (do not treat as normative until independently reproduced):

```text
master : 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
labels : ndk:1 / subject:github:Patternity/ndk / purpose:donations /
         profile:dash:mainnet / type:bip32-seed
seed   : 1f39b1280a22d78b72f3b670768fdcd586d21d1a09315147b490845f231ff239
```

---

## 23. Open questions

Tracked in `RATIONALE.md` §"Open questions"; resolution is REQUIRED before `1.0`. In
brief: whether NDK Profiles needs a registry vs. a documentation convention; evidence of
real-world demand; and confirmation of duplicate-key handling across strict parsers in
Node and browsers.

---

## 24. Normative dependency

NDK-1 depends on SLIP-0021 for the HMAC derivation. When this document and SLIP-0021
disagree about the HMAC construction, SLIP-0021 is authoritative for the cryptographic
operation; this document is authoritative for descriptor validation and label
compilation.

* SLIP-0021: <https://github.com/satoshilabs/slips/blob/master/slip-0021.md>
