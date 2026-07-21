# groundnet protocol — contract v0 (draft)

**Status:** draft. **Envelope version:** `gn/0`. **Payload schema version:** `wisdom/0`.

This document specifies the wire contract two groundnet participants use to exchange a re-validated
remediation unit. It defines the **stable envelope**, the **versioned payload**, the **attestation**
and **transparency-log** model, and the **invariants** a compliant node MUST uphold. It does *not*
specify a network topology, a discovery mechanism, or any node's internals — those are out of scope
for v0 (see [§9](#9-non-goals-and-future-scope)).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as
in RFC 2119 / RFC 8174.

All examples use documentation-reserved identifiers only (`example.net`, addresses from
`192.0.2.0/24` per RFC 5737). A conforming payload MUST NOT contain any real estate identifier
([§6, INV-2](#6-invariants-normative)).

---

## 1. Model

A groundnet exchange moves exactly one **unit** from a producing node to any consuming node. A unit is:

```
Envelope  ──carries──▶  Payload (a de-identified "wisdom chunk")
   │
   ├── signed by a stable Pseudonym (a keypair)
   └── recorded in a public, append-only Transparency Log
```

- The **envelope** is stable across the protocol's life and evolves only under [§7](#7-compatibility).
- The **payload** is versioned and expected to evolve freely.
- Trust is **not** conferred by receipt. A consumer re-validates and re-graduates every unit through
  its own local governance before the unit may influence anything ([INV-1](#6-invariants-normative)).

## 2. The envelope (stable)

The envelope is a JSON object. Field names and semantics in this section are **stable**: they may be
*added to* under [§7](#7-compatibility) but not removed or repurposed.

```json
{
  "gn":        "gn/0",
  "payload":   { "...": "the versioned wisdom chunk — see §4" },
  "producer":  "gnpub:ed25519:9f2c…7ab1",
  "created":   "2026-07-21T00:00:00Z",
  "sig":       "base64(ed25519 detached signature over the canonical envelope, sig field excluded)",
  "log":       {
    "log_id":  "the transparency log's public key identity",
    "index":   1234,
    "proof":   "base64(inclusion proof — see §5)"
  }
}
```

| Field | Rule |
|-------|------|
| `gn` | MUST be the envelope version. A consumer MUST reject an envelope whose major version it does not implement. |
| `payload` | The versioned payload ([§4](#4-the-payload-versioned)). Opaque to the envelope layer except for its `v` field. |
| `producer` | The producer's **pseudonymous** public-key identity ([§3](#3-attestation-pseudonymous)). MUST be a `gnpub:` URI. MUST NOT be a real-world or estate identity ([INV-3](#6-invariants-normative)). |
| `created` | RFC 3339 UTC timestamp of signing. Coarse to the hour SHOULD be preferred to reduce timing correlation. |
| `sig` | A detached signature by `producer`'s private key over the canonical serialization of the envelope with `sig` and `log` excluded. A consumer MUST verify `sig` before considering the payload. |
| `log` | The transparency-log inclusion record ([§5](#5-transparency-log)). A consumer MUST verify the inclusion proof before granting the unit any reputation weight. |

**Canonicalization.** The bytes signed and logged MUST be a deterministic serialization (JCS,
RFC 8785). Two nodes MUST produce byte-identical canonical forms for the same logical envelope.

## 3. Attestation (pseudonymous)

- A producer is identified **only** by a stable keypair. Its public identity is a `gnpub:` URI:
  `gnpub:<alg>:<fingerprint>` (v0 REQUIRES `ed25519`).
- Reputation ([§8](#8-reputation)) accrues to the `gnpub:` identity, never to a person, an
  organization, or an estate.
- A node MAY rotate its pseudonym; doing so starts a fresh reputation history (this is the intended
  cost of rotation, not a defect).
- **Privacy tradeoff (documented, not yet specified):** a stable pseudonym gives reputation continuity
  at the cost of linkability across a node's contributions. A future mode MAY offer unlinkable
  per-contribution signatures (ring/group signatures) for maximum privacy, trading away continuity.
  v0 is stable-pseudonym only.

No field of the envelope or payload may carry a real-world identity. This is enforced as
[INV-3](#6-invariants-normative).

## 4. The payload (versioned)

The payload is the **generalizable wisdom chunk**: what an incident *class* was, how it was diagnosed,
which operation *class* resolved it, and the mechanically-verified outcome — with **zero** estate
specifics. The payload body is versioned by its `v` field and MAY evolve; consumers MUST ignore
unknown fields (forward-compatibility) and MUST NOT fail on them.

```json
{
  "v":            "wisdom/0",
  "alert_class":  "service-down/http",
  "diagnosis":    "process exited; port unbound; no upstream dependency implicated",
  "op_class":     "restart-service",
  "reversible":   true,
  "blast_class":  "single-host",
  "outcome": {
    "verifier":   "mechanical",
    "verdict":    "clean",
    "method":     "post-condition re-check: service active, port bound",
    "n":          7
  },
  "artifact": {
    "kind":       "runbook",
    "ref":        "sha256:3c96…a0df",
    "graduated":  true
  }
}
```

| Field | Meaning |
|-------|---------|
| `v` | Payload schema version. |
| `alert_class` | A **generalized** alert category — never a specific host, address, or rule id. |
| `diagnosis` | The generalized root-cause finding. Free text, MUST be de-identified. |
| `op_class` | The **class** of remediation operation (e.g. `restart-service`), never a concrete command line targeting a real host. |
| `reversible`, `blast_class` | Governance-relevant properties of the op-class. |
| `outcome` | The **verified** result. `verifier` MUST be `mechanical` (an LLM-free, deterministic check) — the producer MUST NOT self-adjudicate. `verdict` ∈ {`clean`, `partial`, `deviation`}. `n` is how many independent applications back this outcome. |
| `artifact` | Optional. A content-addressed reference to a graduated artifact (runbook/skill/rubric). The artifact itself is fetched out of band; `ref` is a hash, never an estate URL. `graduated` states whether the *producer* graduated it (still subordinate — see [INV-1](#6-invariants-normative)). |

A payload MUST NOT contain: hostnames, IP/MAC addresses, topology, credentials or secret references,
raw incident traces, ticket ids, or any organization name. This is [INV-2](#6-invariants-normative).

## 5. Transparency log

Provenance is a **signed, append-only, tamper-evident transparency log** with independent witnesses —
the Certificate-Transparency / Sigstore-Rekor model. It is **not** a blockchain.

- A producer MUST submit the canonical envelope hash to a groundnet transparency log and include the
  returned `{log_id, index, proof}` in the envelope before distributing it.
- A consumer MUST verify the inclusion proof against a log whose `log_id` it trusts, and SHOULD verify
  cross-witness consistency, before granting the unit any reputation weight.
- The log records **that a pseudonym said a thing at an index** — it is an ordering-and-tamper-evidence
  primitive, not a statement of truth. Truth is established locally by each consumer's own
  verification ([INV-1](#6-invariants-normative)).
- The log MUST NOT store payload plaintext beyond the envelope hash unless the payload is of the
  zero-estate-content public tier.

Rationale for *not* a blockchain: groundnet requires no global consensus on a single truth
(subordinate-not-authority makes local re-validation authoritative), and a public chain would leak
incident metadata, add latency, and introduce token / smart-contract attack surface for no benefit.
A local node's own hash-chained governance ledger is already a "chain of one"; the groundnet log is
its federated, multi-witness, signed extension.

## 6. Invariants (normative)

A conforming implementation MUST uphold all of the following. These are the contract's safety core.

- **INV-1 — Subordinate, not authority.** A received unit is a *hint*. It MUST pass the consuming
  node's own full governance (access-list, never-auto floor, mode chokepoint, local verification) and
  re-graduate against the consumer's own verified outcomes before it earns any local trust. No unit,
  and no producer reputation, may cause a consumer to act.
- **INV-2 — De-identified payload.** A payload MUST carry no estate-specific identifier ([§4](#4-the-payload-versioned)). A producer MUST strip the estate layer before signing; a consumer SHOULD
  reject a unit that fails a de-identification check.
- **INV-3 — Pseudonymous only.** No envelope or payload field may carry a real-world or organizational
  identity. Reputation accrues to the `gnpub:` pseudonym alone.
- **INV-4 — Default-off.** A node MUST NOT emit any unit until an operator has explicitly enabled
  federation. Absent/failed configuration MUST fail closed to *no sharing*.
- **INV-5 — Verified-outcome only.** A shared `outcome` MUST come from a mechanical, LLM-free verifier;
  a producer MUST NOT share an outcome it adjudicated with the same reasoning system that proposed the
  remediation.
- **INV-6 — Stable envelope.** The envelope contract ([§2](#2-the-envelope-stable)) changes only under
  [§7](#7-compatibility). Payloads evolve; the envelope does not break.

## 7. Compatibility

- The envelope version `gn/<major>` changes **major** only for a breaking envelope change; consumers
  MUST reject unknown majors. Additive envelope fields are minor and MUST be ignored when unknown.
- The payload `v = wisdom/<major>` evolves independently. Consumers MUST ignore unknown payload fields
  and MUST NOT fail on them (forward-compatibility).
- The **adapter seam** (a node's emit/ingest boundary) is the only component that binds to this
  contract. A node's internals never appear here, so a node can evolve freely as long as its adapter
  keeps producing conforming envelopes.

## 8. Reputation (informative, v0)

Reputation is the federated aggregation of signed, pseudonymous, **verified-outcome** attestations —
"your shared op-class, when others applied it, was confirmed clean by *their* mechanical verifier."
It is a conflict-free, monotone roll-up over transparency-logged attestations (a CRDT-style
accumulation), never an on-chain vote or a token. A concrete reputation function is out of scope for
v0 and will be specified once real units exist to calibrate it. Until then: reputation is advisory
input to a consumer's *ranking* of hints, and — by INV-1 — never a gate on action.

## 9. Non-goals and future scope

Explicitly **out of scope for v0** (named so a node reserves the seam without over-building):

- Network topology, peer discovery, transport, and membership admission.
- The unlinkable per-contribution signature mode ([§3](#3-attestation-pseudonymous)).
- The concrete reputation function ([§8](#8-reputation)).
- Artifact distribution (only the content-addressed `ref` is in scope; fetch is out of band).
- Any node-internal representation. This document names no node internals by design.

## Changelog

- **v0 (draft)** — initial contract: stable `gn/0` envelope, `wisdom/0` payload, pseudonymous
  attestation, transparency-log provenance, and invariants INV-1…INV-6.
