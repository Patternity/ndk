# NDK Governance

Non-normative. Describes who maintains what and how decisions are made.

## Ownership

- **The specification** (this repository, `Patternity/ndk-spec`) is owned and maintained by
  **Patternity**. It is implementation-independent and is the single normative source for
  descriptor validation, label compilation, and the NDK seed.
- **The first reference implementation** (`LexxXell/ndk`, npm
  `@lexxxell/named-deterministic-keys`) is maintained **independently** by **LexxXell**.
  The reference implementation does **not** define normative behavior; where it and the
  specification disagree, the specification wins.
- Independent implementations in any language are welcome and encouraged. No implementation
  is privileged as "the" implementation.

## Normative authority

The order of authority is:

1. **SLIP-0021** — for the underlying HMAC-SHA512 construction (`SPEC.md §24`).
2. **`SPEC.md`** — for descriptor validation, label compilation, and the 32-byte seed.
3. **`SPEC-PROFILES.md`** — for seed→key transforms. Versioned independently of the core.

Test vectors in `vectors/` are the machine-checkable expression of the above and are
authoritative for conformance.

## Decision process

- Changes are proposed as issues/pull requests per `CONTRIBUTING.md`.
- **Editorial** changes (clarifications, examples, more vectors) may be merged by the
  maintainers without a version bump beyond the document revision.
- **Normative core** changes (fields, label keys/order/encoding, normalization, algorithm,
  output length, interpretation of a field) require: a documented rationale, updated
  vectors, a compatibility note, and a new NDK major version if they change the seed.
- **Transform** changes go through `SPEC-PROFILES.md §6` and bump the NDK Profiles
  revision only — never the core major version.

## Versioning authority

- The `ndk` integer (the core major version) is controlled solely by changes to `SPEC.md`
  that alter the derived seed, per `SPEC.md §17`.
- The NDK Profiles revision is controlled by `SPEC-PROFILES.md` and moves independently.

## Releases

The specification follows the stages in `ROADMAP.md`. A `1.0.0` tag requires all
`ROADMAP.md` blockers closed, including at least one independent vector verification and a
security review of the reference implementation.

## Trademarks and naming

"NDK" is used as a technical acronym for Named Deterministic Keys and as the derivation
label `ndk:`. It is not a claim over the acronym in other domains (e.g. the Android NDK or
Nostr NDK). See `RATIONALE.md §7`.
