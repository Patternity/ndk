# NDK Roadmap and Release Readiness

Non-normative. Tracks how NDK moves from Draft to a stable `1.0`, the release stages, and
the concrete conditions that **block** `1.0`.

## Release stages

A conservative pre-release path. Each stage may repeat (`alpha.2`, `beta.2`, …).

```text
0.1.0-alpha.N   Draft. Labels/fields/vectors may change. Not for production secrets.
0.1.0-beta.N    Feature-frozen draft. No label changes without a documented incompat note.
1.0.0-rc.N      Release candidate. All 1.0 blockers closed; only fixes.
1.0.0           Stable. Labels, order, normalization, and vectors are frozen.
```

Any change to labels before `1.0` **regenerates all vectors** and is marked incompatible
in `CHANGELOG.md`. A Draft revision is not a stable cryptographic protocol.

## Requirements before `1.0.0`

These MUST all be true before tagging `1.0.0`:

- [ ] **Descriptor fields stable** — the five fields and their validation are frozen.
- [ ] **Label order stable** — `ndk, subject, purpose, profile, type` frozen.
- [ ] **Normalization policy stable** — reject-non-NFC, no silent normalization.
- [ ] **Vectors frozen** — `vectors/ndk-1.json`, `vectors/ndk-profiles-1.json`,
      `vectors/invalid-descriptors.json` final.
- [ ] **Independent verification** — at least one implementation or vector check that does
      not share code with the reference implementation reproduces every vector.
- [ ] **Security review** — a focused review of the reference implementation is completed
      and its findings addressed.
- [ ] **Versioning rules documented** — `CHANGELOG.md` + `SPEC.md §17` describe what forces
      a new major version. (Done.)
- [ ] **Naming conflict resolved** — package scope chosen and documented; `ndk:` label
      retained as a domain tag (see `RATIONALE.md §7`). (Decided.)
- [ ] **Duplicate-key handling confirmed** — strict text-layer rejection verified in Node
      and in at least one browser JSON path.

## Open questions that block `1.0` (from `RATIONALE.md §9`)

1. **Profile registry vs. convention** — does `SPEC-PROFILES.md` need ownership-namespaced
   registration, or is the documentation convention sufficient? A registry improves
   interop but adds governance load. **Decision required.**
2. **Real-world demand** — at least one concrete adopter beyond the author, or an explicit
   decision to publish as a convention regardless. **Evidence required.**
3. **`bip32-seed` path pinning** — should each profile pin a BIP-32/44 derivation path, or
   leave it to the application? Pinning improves key-interop but expands scope. **Decision
   required.**
4. **`subject` length bound** — confirm 1024 UTF-8 bytes is appropriate, or reduce.

## Explicitly out of scope for `1.0`

- A profile registry service or website.
- Blockchain-specific key-generation algorithms in the core.
- Variable-length or HKDF-expanded core output (use the `hkdf-sha256` transform instead).
- A binary descriptor encoding.

## Current status

**Draft 1.0** (spec revision), targeting reference implementation `0.1.0-alpha.1`. The
crypto core (SLIP-0021) and all vectors reproduce byte-for-byte; the blockers above are
process/decision items, not cryptographic ones.
