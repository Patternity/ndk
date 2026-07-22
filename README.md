# NDK — Named Deterministic Keys

**A small, human-readable convention for deriving a deterministic 32-byte seed from a
master secret and a named context, using [SLIP-0021](https://github.com/satoshilabs/slips/blob/master/slip-0021.md)
without modification.**

```json
{
  "ndk": 1,
  "subject": "github:Patternity/ndk",
  "purpose": "donations",
  "profile": "dash:mainnet",
  "type": "bip32-seed"
}
```
```text
master secret + descriptor  ──SLIP-0021──▶  32-byte NDK seed  ──type transform──▶  key
```

This repository is the **specification**, maintained by **Patternity**. It is
implementation-independent. The first reference implementation is maintained
independently by **LexxXell** (see below); the implementation does not define normative
behavior.

## What NDK is

* A **naming convention + validator** on top of SLIP-0021. It fixes the descriptor
  fields, their validation, the label format, and the label order, so independent tools
  derive the *same* 32 bytes from the *same* named context.
* Two layers:
  * **[NDK-1 core](SPEC.md)** — `master + descriptor → 32-byte seed`. Small and frozen.
  * **[NDK Profiles](SPEC-PROFILES.md)** — `32-byte seed → key`, a separate,
    independently-versioned set of generic transforms (`secp256k1-raw`, `bip32-seed`,
    `ed25519-seed`, `hkdf-sha256`).

## What NDK is not

NDK is **not** a new cryptographic primitive, a wallet standard, a mnemonic scheme, a
KDF, or a source of entropy. It does not generate the master secret, define BIP-32 paths,
or make guessable values safe. The output is opaque seed material, **not** itself a
private key. See [SPEC.md §5](SPEC.md) and [SECURITY.md](SECURITY.md).

## Why it exists (honestly)

SLIP-0021 already provides labeled, domain-separated derivation. NDK adds exactly three
things on top: (1) a fixed, human-readable structure for the derivation context;
(2) a public, validated recovery descriptor with stable error codes; (3) machine test
vectors for cross-language interoperability, including a named seed→key transform so the
*same seed* cannot silently become *different keys* in two tools. If you need none of
those, use SLIP-0021 directly. The full argument, and the alternatives that were
rejected, are in [RATIONALE.md](RATIONALE.md).

## Repository layout

```text
SPEC.md                       normative core specification (NDK-1)
SPEC-PROFILES.md              normative seed→key transforms (independently versioned)
RATIONALE.md                  design decisions, rejected alternatives, open questions
SECURITY.md                   security guidance
THREAT-MODEL.md               enumerated threats and mitigations
ROADMAP.md                    release stages and the conditions that block 1.0
GOVERNANCE.md                 ownership, normative authority, decision process
CHANGELOG.md                  spec revision history
CONTRIBUTING.md               how to propose changes
LICENSE                       CC-BY-4.0 (docs); vectors/ and schemas/ are CC0 — see below
schemas/ndk-1.schema.json     structural JSON Schema
vectors/ndk-1.json            normative valid vectors + embedded SLIP-0021 vectors
vectors/ndk-profiles-1.json   transform output vectors
vectors/invalid-descriptors.json  invalid vectors with expected error codes
examples/                     illustrative descriptors (non-normative)
```

## Baseline vector

```text
master : 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
labels : ndk:1 / subject:github:Patternity/ndk / purpose:donations /
         profile:dash:mainnet / type:bip32-seed
seed   : 1f39b1280a22d78b72f3b670768fdcd586d21d1a09315147b490845f231ff239
```

Reproduce it before trusting it. A dependency-free check is 15 lines of HMAC-SHA512.

## Status

**Draft.** Labels, fields, and vectors may still change before `1.0`. Any change to
labels before `1.0` regenerates the vectors and is marked incompatible. Do not treat a
draft as a stable cryptographic protocol. Open questions that block `1.0` are listed in
[RATIONALE.md](RATIONALE.md).

## Reference implementation

The first reference implementation (TypeScript) is maintained independently by LexxXell:
`@lexxxell/named-deterministic-keys`. It implements this specification and is not itself
the standard. Other implementations are welcome and encouraged.

## Licensing

* **Documentation** (`*.md`): [CC-BY-4.0](LICENSE).
* **Test vectors and schemas** (`vectors/`, `schemas/`): dedicated to the public domain
  under **CC0-1.0**, so any implementation may embed them without attribution friction.

See [LICENSE](LICENSE).
