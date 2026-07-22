# Changelog

All notable changes to the NDK specification are documented here. The document revision
(e.g. "Draft 1.0") is editorial and is NOT part of derivation. A change to labels,
fields, order, encoding, normalization, the derivation algorithm, or output length
requires a new **NDK major version** (`ndk:2`) and regenerated vectors.

Format loosely follows Keep a Changelog. Dates are ISO 8601.

## [Draft 1.0] — 2026-07-22

Draft revision 1.0. **Incompatible** with the earlier four-field Draft 0.3: the baseline
seed changed.

### Added
- `type` descriptor field naming the seed→key transform; it is a derivation label and is
  validated syntactically only by the core.
- Separate, independently-versioned **NDK Profiles** specification defining the generic
  transforms `secp256k1-raw`, `bip32-seed`, `ed25519-seed`, `hkdf-sha256`, with vectors.
- Non-normative descriptor **fingerprint** (SHA-256 over labels, first 8 bytes).
- Normative **duplicate-key** policy (§10) and consolidated **Unicode** policy (§11).
- Machine-readable **invalid-descriptor** vectors with stable error codes.
- `THREAT-MODEL.md`, `RATIONALE.md`, `SECURITY.md`, JSON Schema, and examples.

### Changed
- Descriptor is now **five** fields (`ndk, subject, purpose, profile, type`); label order
  is `ndk → subject → purpose → profile → type`.
- Baseline seed is now
  `1f39b1280a22d78b72f3b670768fdcd586d21d1a09315147b490845f231ff239`
  (was `729b48cc…8566` in Draft 0.3). The change reflects both the added `type` label and
  the baseline example `subject` moving from the placeholder `github:Patternity/example`
  to the real repository `github:Patternity/ndk`.
- Key rotation is expressed as a `profile` generation suffix (`dash:mainnet:2`) rather
  than a dedicated field.
- `subject` is explicitly kept free-form/human-readable; a mandatory stable anchor was
  considered and rejected.

### Notes
- Internal spaces in `subject` are permitted; only leading/trailing whitespace and
  control characters are rejected.
- Status remains **Draft**. Labels and vectors may still change before `1.0`.

## [Draft 0.3] — prior

Initial four-field model (`ndk, subject, purpose, profile`), baseline seed
`729b48cc14cefc21ee47bcaf34a668b0c478779e2ed09de8e44a1db349858566`. Superseded.
