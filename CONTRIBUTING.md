# Contributing to NDK

Thanks for your interest. NDK is maintained by Patternity. The goal is a **small,
auditable** convention — contributions that keep it small are especially welcome.

## Ground rules

- All specification text, schemas, vectors, examples, commit messages, and public text
  are in **English**.
- Prefer the smallest change that solves a concrete, demonstrated problem. "Nice to have"
  fields and options are usually declined.
- Every claim of interoperability must be backed by **test vectors**. No vectors, no
  normative change.
- Do not turn NDK into a wallet standard. Blockchain-specific key generation belongs in
  downstream projects, not here. Generic transforms belong in `SPEC-PROFILES.md`.

## What requires a new NDK major version (`ndk:2`)

Any change to descriptor fields, label keys, label order, label encoding, normalization
rules, the derivation algorithm, output length, or the interpretation of a core field. A
major-version change regenerates **all** vectors and is marked incompatible in
`CHANGELOG.md`.

## What does NOT require a core major version

- Adding or amending a transform in `SPEC-PROFILES.md` (it is versioned independently).
- Editorial changes, clarifications, new examples, more vectors.

## Proposing a change

1. Open an issue describing the concrete problem and who is affected.
2. For normative changes, include: the exact wording, updated/added vectors, and a note on
   compatibility (does the baseline seed change?).
3. For a new transform, follow `SPEC-PROFILES.md` §6 (token, exact bytes, failure
   conditions, standard basis, vectors).

## Vectors

- Regenerate vectors from a reference implementation and reproduce them with at least one
  **independent** check (a few lines of HMAC-SHA512) before merging.
- `vectors/` and `schemas/` are CC0 so implementations can embed them freely.

## Security issues

For a suspected specification-level security problem, contact the Patternity maintainers
privately before opening a public issue. Implementation bugs go to the relevant
implementation's tracker.

## Reference implementation

The TypeScript reference implementation is maintained independently by LexxXell and does
not define normative behavior. Independent implementations are encouraged; they should
cite the exact specification revision they target and pass the normative vectors.
