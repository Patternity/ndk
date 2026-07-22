# NDK Profiles

## Seed→key transforms for the NDK 32-byte seed

```text
Status:              Draft
Draft revision:      1.0
Specification owner: Patternity
Depends on:          NDK-1 (SPEC.md), RFC 5869, BIP-32, RFC 8032, SEC 1
Versioning:          Independent of the NDK core major version
```

> This document is **separate from and independently versioned relative to** the NDK
> core ([SPEC.md](SPEC.md)). The NDK core stops at a 32-byte opaque seed and validates
> the `type` field only syntactically. This document defines what each `type` value
> means: how the 32 bytes become a downstream key. Adding a transform here never changes
> the NDK core or any NDK seed.

---

## 1. Why this layer exists

An NDK seed is 32 uniformly random bytes. The *same* 32 bytes can be turned into
*different* keys by different downstream algorithms (raw scalar vs. BIP-32 seed vs.
Ed25519 seed). Without a named, specified transform, two implementations can consume one
seed and produce different addresses. NDK closes this by making the transform:

1. **explicit** — named in the descriptor's `type` field;
2. **part of derivation** — `type` is an NDK label, so different transforms yield
   different seeds and cross-protocol reuse of one seed is impossible;
3. **specified here** — with exact bytes and test vectors.

`type` names the **transform** (how). `profile` names the **downstream context** (which
network / coin / address version, plus an optional generation suffix). They are
complementary and both participate in NDK derivation.

---

## 2. Conformance terminology

RFC 2119 / RFC 8174 keywords apply as in the NDK core specification.

Let `S` denote the 32-byte NDK seed produced by [SPEC.md §13](SPEC.md). All transforms
below take `S` as input and MUST treat it as opaque bytes (no endianness assumption
except where a transform explicitly defines one).

---

## 3. Registry of transforms

NDK Profiles defines a small set of **generic** transforms. They are deliberately
blockchain-agnostic: a transform produces a key or key-seed; network-specific details
(coin type, address version) belong to the consuming application, guided by `profile`.

| `type`           | Output                                  | Basis            |
|------------------|-----------------------------------------|------------------|
| `secp256k1-raw`  | secp256k1 private key (scalar)          | SEC 1            |
| `bip32-seed`     | BIP-32 master key + chain code          | BIP-32           |
| `ed25519-seed`   | Ed25519 private key seed                | RFC 8032         |
| `hkdf-sha256`    | L symmetric key bytes (default L=32)    | RFC 5869         |

An implementation MAY support a subset. When asked to produce a key for an unsupported
`type`, it MUST fail with a clear error at the transform step (the 32-byte NDK seed is
still well-defined and MAY be returned).

### 3.1 `secp256k1-raw`

Interpret `S` as a 256-bit big-endian integer `d`. Let `n` be the secp256k1 group order.

* If `d == 0` or `d >= n`, the transform MUST fail. The operator SHOULD bump the profile
  generation suffix (e.g. `…:2`) and re-derive. (Failure probability ≈ 2⁻¹²⁸;
  effectively never.)
* Otherwise the secp256k1 private key is `d`. The public key is `d·G` per SEC 1.

`secp256k1-raw` yields a single key, not a hierarchy. Use `bip32-seed` if you need a
tree.

### 3.2 `bip32-seed`

Use `S` as the BIP-32 seed:

```text
I  = HMAC-SHA512(key = "Bitcoin seed", message = S)
IL = I[0:32]   ; master private key
IR = I[32:64]  ; master chain code
```

* If `IL == 0` or `IL >= n` (secp256k1 order), the master key is invalid per BIP-32 and
  the transform MUST fail; bump the generation and re-derive.
* The application then derives child keys per BIP-32 / BIP-44. The derivation **path**
  (e.g. coin type) is chosen by the application from `profile`; this document does not
  fix a path. A profile MAY pin a path in its own documentation.

> `"Bitcoin seed"` is the literal BIP-32 constant for secp256k1 and is used regardless of
> which secp256k1 chain the profile targets. It is not a claim about Bitcoin.

### 3.3 `ed25519-seed`

`S` **is** the 32-byte Ed25519 private key seed as defined by RFC 8032 §5.1.5. The public
key and signatures follow RFC 8032. No pre-processing of `S` is performed by NDK; the
clamping described by RFC 8032 happens inside the Ed25519 algorithm, not at this layer.

### 3.4 `hkdf-sha256`

Produce `L` symmetric key bytes with HKDF-SHA256 (RFC 5869):

```text
OKM = HKDF-SHA256(IKM = S, salt = "" , info = "", L)   ; default L = 32
```

* `salt = ""` means HKDF-Extract uses `HashLen` zero bytes, per RFC 5869 §2.2.
* Domain separation is already provided by the NDK labels (including `type` and
  `profile`), so `info` is empty by default.
* An application requiring more than 32 bytes, or several independent subkeys, MAY
  specify `L` and a non-empty `info` **in its own profile documentation**; such use is
  outside the default transform and MUST be documented to remain interoperable.

Directly using `S` as a symmetric key without HKDF is discouraged: `hkdf-sha256` gives a
clean, length-flexible, standards-based path and keeps `S` from being copied verbatim
into multiple subsystems.

---

## 4. Test vectors

All vectors use the NDK core baseline master secret and the baseline descriptor with the
`type` set as shown:

```text
master  : 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
subject : github:Patternity/ndk
purpose : donations
profile : dash:mainnet
```

### 4.1 `secp256k1-raw`

```text
NDK seed = private key (big-endian scalar):
  4b456b3c3b81f371b414d98c4b8b039bc84522856c37b14395fbae9150f43640
```

### 4.2 `bip32-seed`

```text
NDK seed            : 1f39b1280a22d78b72f3b670768fdcd586d21d1a09315147b490845f231ff239
BIP-32 master priv  : 926a8ecf73693a8b0a7860a81cb3e709da80bc18b6be8f34628f7da403c59e2a
BIP-32 chain code   : 748a72e6097185070634c59dba05968299c31c489474fc481eaa130f51e12b85
```

### 4.3 `ed25519-seed`

```text
NDK seed = Ed25519 private seed:
  0b3cdd91f5f8852c2ff06a14b1fc6323f201f38ad0ecce9f80f35ebbf16deee0
Ed25519 public key (RFC 8032):
  6a85b4c08969c2fa6988df63ebfd257eb1cd732f3f3811cdb500acc63510b717
```

### 4.4 `hkdf-sha256`

```text
NDK seed (IKM)      : 5ce6c26cac0585bc415f0e962e8df0a09a93c492801641d506910667b4b1ece5
OKM, L=32, salt="", info="":
  490b59e6dde3f0cce7bda5bf930b9bc71563ded92b5af0c83517d0093f0f1eb3
```

These vectors are also machine-readable in
[`vectors/ndk-profiles-1.json`](vectors/ndk-profiles-1.json).

---

## 5. The `profile` string and generation

`profile` is opaque to the NDK core. This document recommends, but does not enforce, the
following convention so profiles stay legible:

```text
<namespace>[:<network>][:<variant>][:<generationN>]
```

Examples: `dash:mainnet`, `dash:testnet`, `solana:mainnet`, `minisign`,
`ssh:ed25519`, `application:vault:v1`, `dash:mainnet:2` (generation 2).

A downstream consumer that reads network parameters from `profile` MUST ignore a trailing
`:N` generation segment when present.

---

## 6. Adding a transform

A new transform is added by amending this document (not the NDK core):

1. Choose a lowercase `type` token matching `^[a-z0-9]+(?:-[a-z0-9]+)*$`.
2. Specify exact byte behavior, failure conditions, and the standard it is based on.
3. Add machine-readable vectors.
4. Bump the NDK Profiles revision. The NDK core major version does **not** change.

Because `type` is an NDK derivation label, a *new* `type` value automatically derives a
*new* seed and cannot collide with existing keys.

---

## 7. Security notes specific to transforms

* `secp256k1-raw` and `bip32-seed` share the secp256k1 order check; an implementation
  MUST perform it and MUST NOT clamp `d` into range (that would break determinism and
  reduce entropy). Fail and bump generation instead.
* `ed25519-seed` and `secp256k1-raw` derive from *different* NDK seeds even for the same
  subject/purpose/profile, because `type` is part of derivation. Do not reuse one
  transform's seed for another algorithm.
* `hkdf-sha256` with a non-default `info`/`L` is only interoperable if that parameter set
  is documented by the profile. Keep it explicit.

---

## 8. Normative dependencies

* RFC 5869 — HKDF: <https://www.rfc-editor.org/rfc/rfc5869>
* BIP-32 — <https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki>
* RFC 8032 — EdDSA/Ed25519: <https://www.rfc-editor.org/rfc/rfc8032>
* SEC 1 — Elliptic Curve Cryptography: <https://www.secg.org/sec1-v2.pdf>
