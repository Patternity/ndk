# NDK Security Considerations

Non-normative guidance. Enumerated threats are in [THREAT-MODEL.md](THREAT-MODEL.md).

## 1. The descriptor is public; the master secret is everything

An NDK descriptor is not a secret. Security MUST rest entirely on the master secret, not
on the secrecy of `subject`, `purpose`, `profile`, `type`, or the labels. Anyone who
knows or guesses a descriptor and holds the master secret can derive the seed.

## 2. Master-secret quality

NDK adds **no entropy**. A password, URL, project name, username, or any guessable value
MUST NOT be used directly as a master secret. Use a high-entropy secret (e.g. 32 random
bytes from a CSPRNG, or a hardware-backed secret). For informative sourcing patterns —
including deriving the master secret from a BIP-39 mnemonic — see `SPEC.md` Appendix A.

## 3. Context separation, not compromise isolation

NDK gives **context separation**: different descriptors yield different seeds. It does
**not** give compromise isolation: if the master secret leaks, every seed for every known
or guessed descriptor is exposed. Consequences:

* For high-value, long-lived keys (release signing, treasury), consider a **separate
  master secret per trust domain** rather than deriving everything from one. The master
  secret is an input to NDK, so using several is fully supported — see the reference
  CLI's multi-secret handling.
* Do not treat NDK as a way to safely explode one secret into unlimited independent
  credentials. It is a naming layer, not a compartmentalization boundary.

## 4. Subject stability (recovery)

The master secret contains no record of previously used descriptors. Recovery depends on
reproducing the **exact** descriptor bytes. Because `subject` is free-form:

* A mutable name (e.g. a repository slug that can be renamed or transferred) will derive a
  **different** seed after it changes — potentially locking you out of funds or an
  identity. This is a property of determinism, not a bug.
* If rename-resilience matters, choose a stable `subject` you control (a DID, a UUID, a
  domain you own, or a numeric id). This is a recommendation, not a requirement — NDK will
  faithfully derive from whatever you provide.
* Preserve the exact descriptor (store it publicly alongside the project). Changing any
  character may change the seed. The `fingerprint` (SPEC §15) lets you detect drift
  without exposing the seed.

## 5. Rotation

To rotate a key, bump the profile generation suffix (`dash:mainnet` → `dash:mainnet:2`).
Deterministic recovery and rotation coexist: the old and new descriptors both remain
recoverable, and derive independent seeds.

## 6. Logging and memory

Production implementations MUST NOT log the master secret, SLIP-0021 node data,
intermediate node keys, the NDK seed, or downstream private keys. Zeroize secret buffers
where the platform allows. The reference implementation returns a fresh array for the
seed and never mutates the caller's master-secret buffer, but JavaScript cannot guarantee
secret erasure from memory — treat this as best-effort.

## 7. Command-line interfaces

A CLI SHOULD NOT accept a master secret as a positional argument (shell history, process
listings). Prefer standard input, a protected file, an OS secret store, or hardware-backed
derivation. Environment variables are acceptable only with an explicit warning. A CLI MUST
NOT print secrets except when the user explicitly invokes a derivation command, and MUST
keep diagnostics on a separate stream from machine-readable output.

## 8. Profile / transform trust

`profile` is an identifier, not a security guarantee: the name `dash:mainnet` does not
prove the downstream implementation builds a correct Dash key. `type` names a transform
defined in [NDK Profiles](SPEC-PROFILES.md); users must still trust the code that
implements that transform. NDK guarantees the *seed*, not the correctness of what consumes
it.

## 9. Reporting

Report suspected specification-level security issues via the process in
[CONTRIBUTING.md](CONTRIBUTING.md). Implementation vulnerabilities should be reported to
the relevant implementation's maintainer.
