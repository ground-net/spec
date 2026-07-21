# Security

groundnet is a **protocol contract**, not a running service — but the contract itself is a security
artifact, and its invariants ([`spec.md` §6](spec.md#6-invariants-normative)) are load-bearing.

## What a protocol-level security report looks like

Report privately (do not open a public issue) if you find:

- A way an envelope or payload could carry an estate-specific identifier past a conforming
  de-identification check (**INV-2 / INV-3** bypass — a reconnaissance leak).
- A way a producer's reputation could cause a *conforming* consumer to act without local
  re-graduation (**INV-1** bypass — subordinate-not-authority defeated).
- A transparency-log construction that lets a producer present a valid-looking inclusion proof for an
  envelope that was never logged, or that leaks payload plaintext for a non-public-tier unit ([§5](spec.md#5-transparency-log)).
- A canonicalization ambiguity ([§2](spec.md#2-the-envelope-stable)) that lets the signed bytes differ
  from the logged bytes (signature/log split).
- Any path by which turning federation *off* (**INV-4**) still emits a unit.

## Coordinated disclosure

Please report privately and allow time for a fix before public discussion. Contact details will be
published here once the network reference implementation exists; until then, use the private contact on
the maintainer's profile.

## Scope

This repository contains **no** running code, no credentials, and no estate data by construction. The
CI publish path re-scans every push for accidental secret or real-identifier leakage and refuses to
publish on a survivor. If you find such a leak in the published mirror, it is a bug — please report it.
