# NDK Threat Model

Non-normative. For each threat: attacker capability, impact, whether NDK raises or lowers
the risk versus using SLIP-0021 directly, specification and implementation mitigations,
and residual risk.

Legend for "NDK effect": ↑ increases risk, ↓ decreases, → neutral (general problem NDK
neither helps nor worsens).

| # | Threat | Attacker capability | Impact | NDK effect | Mitigation (spec / impl) | Residual |
|---|--------|---------------------|--------|-----------|--------------------------|----------|
| 1 | Master-secret compromise | Obtains master secret | Every derivable seed exposed | ↑ (one secret → many seeds) | Spec: "context separation, not compromise isolation"; recommend per-domain master secrets. Impl: never log secrets | High — inherent; use separate masters for high-value keys |
| 2 | Descriptor loss | Loses the exact descriptor | Cannot recover the seed | → | Spec: preserve exact descriptor publicly; fingerprint to verify | Low if descriptor stored |
| 3 | Descriptor mutation | Silently alters a byte | Wrong seed derived | ↑ (recovery is byte-exact) | Spec: reject-don't-normalize; fingerprint detects drift. Impl: stable validation | Medium |
| 4 | Subject rename / repo transfer | Renames the named entity | Silent loss of access | ↑ | Spec+SECURITY: use stable subject if resilience needed; document tradeoff | Medium — operator choice |
| 5 | Unicode non-NFC | Supplies decomposed form | Two "equal" strings, two seeds | ↑ | Spec: reject non-NFC (`not-nfc`). Impl: `s.normalize('NFC') === s` | Low |
| 6 | Homoglyph / confusable subject | Uses look-alike characters | Human targets wrong descriptor | ↑ | Partial: NFC only; NDK does not do confusable detection | Medium — display/review responsibility |
| 7 | Case confusion | Relies on case ambiguity | Different seed than intended | → | Spec: case is significant and preserved; documented | Low |
| 8 | Duplicate JSON keys | Crafts text with repeated key | Parser-dependent field value | ↑ (JSON footgun) | Spec §10: reject `duplicate-property` at text layer. Impl: strict parser at input boundary | Low if text parser strict |
| 9 | JSON parser inconsistency | Exploits lax parser | Divergent acceptance | ↑ | Spec: object API + strict text parser; conformance vectors | Low–medium (open question #3) |
| 10 | Malicious profile string | Supplies hostile `profile`/`type` | Downstream misuse | → | Spec: strict regex on both; core treats as opaque bytes | Low |
| 11 | Same-seed-different-key | Two tools interpret one seed differently | Wrong address / lost funds | ↓ (this is why `type` exists) | Spec: `type` names transform and is a derivation label; Profiles specifies bytes + vectors | Low for defined transforms |
| 12 | Weak master secret | Guesses low-entropy secret | Full compromise | → | Spec: forbid guessable secrets; no entropy added | High if ignored |
| 13 | Secret via CLI history | Reads shell history / `ps` | Master secret leak | → | Spec/CLI: no secret in argv; stdin/file/OS store | Low if followed |
| 14 | Secret via env vars | Reads environment | Master secret leak | → | CLI: warn on env use | Medium |
| 15 | Secret in logs / crash reports | Reads logs | Secret or seed leak | → | Spec §19 / SECURITY: never log secrets, nodes, seeds | Low if followed |
| 16 | Secret in process memory | Reads process memory | Secret leak | → | Impl: fresh buffers, no needless copies; JS cannot guarantee erasure | Medium |
| 17 | Dependency compromise | Poisons a dependency | Arbitrary behavior | ↓ | Impl: minimal/zero runtime deps (Node crypto only) | Low |
| 18 | Malicious transform adapter | Ships a wrong `type` implementation | Wrong/backdoored key | → | Spec: users must trust transform code; vectors let them verify | Medium |
| 19 | Cross-application seed reuse | Reuses one seed elsewhere | Correlation / key reuse | ↓ | Spec: distinct descriptors → distinct seeds; `type` prevents cross-protocol reuse | Low |
| 20 | Supply-chain (package) | Publishes malicious package | Compromise of consumers | → | Impl: signed releases, pinned CI, provenance | Medium |
| 21 | Hardware-wallet display limits | Truncates long label on device | User approves wrong context | ↑ (labels can be long) | Spec: `key:` prefixes aid display; keep subjects short | Medium |
| 22 | QR / transcription error | Mis-copies descriptor | Wrong seed | ↑ | Fingerprint to verify; store machine-readable | Low–medium |
| 23 | Backup / recovery failure | Loses master or descriptor | Permanent loss | → | Spec: preserve both; test recovery | Medium |

## Notes

* The threats NDK **increases** cluster around two design facts: one master secret feeds
  many seeds (#1), and recovery is byte-exact over a human-authored descriptor (#3–#8,
  #21–#22). These are inherent to a deterministic naming scheme and are mitigated by
  documentation and strict validation, not by cryptography.
* The threat NDK meaningfully **decreases** is #11 (same-seed-different-key), via the
  explicit `type` transform — the main reason NDK is more than "four label names".
