# BIP: segOP — Segregated OP_RETURN Data Lane

This repository contains the draft BIP specification for **segOP**, a proposed
Bitcoin soft fork that introduces a dedicated, prunable, fully-priced data lane
for non-consensus payloads.

segOP extends the transaction format using the BIP141-style marker/flag
mechanism and adds:

- a post-witness **segOP section** containing a TLV-structured payload,
- a dedicated **P2SOP** OP_RETURN output committing to that payload,
- strict consensus rules linking the payload ↔ commitment,
- full-fee accounting (4 WU per segOP byte),
- optional pruning of payloads after a validation window.

The purpose is to provide a clear, explicit home for metadata, proofs, and
arbitrary payloads—separate from validation-critical surfaces like script and
witness—while remaining fully backward compatible with legacy nodes.

segOP does **not** classify or restrict data. It is a structural improvement
that enables sane, predictable policy in the future.

---

## Why segOP?

Bitcoin currently mixes many unrelated classes of data inside the same surfaces:

- validation-critical witness data,
- normal payment scripts,
- legitimate protocol metadata (e.g. rollup roots, channel hints),
- large arbitrary payloads (JPEGs, inscriptions, vendor blobs, “runes”),
- and potentially harmful or legally sensitive content.

Nodes have no native way to distinguish these categories. This leads to:

- policy ambiguity (fees, eviction, limits),
- reliance on fragile heuristics,
- distorted incentives via the witness discount,
- long-term storage uncertainty.

segOP introduces:

- **a single explicit lane** for all non-consensus data,
- **bounded, fully-priced bytes**,
- **mandatory TLV framing** ensuring payloads are well-formed,
- **a binding commitment** in P2SOP,
- **optional pruning** after validation.

These changes allow node implementations, miners, researchers, and explorers to
handle data safely and predictably, without altering consensus semantics or
introducing censorship.

---

## Relationship to BUDS (Informative)

segOP is designed to be **composable** with the **Bitcoin Unified Data Standard
(BUDS)**.

- segOP provides the on-chain plumbing (lane, structure, commitment).
- BUDS provides optional semantic markers for classification.

BUDS metadata (tier/label TLVs) may be included in segOP payloads, but they are
strictly **non-consensus** and ignored by nodes that do not implement BUDS.

Together, segOP + BUDS give policy and tooling a clean foundation for future
improvements without requiring consensus changes.

---

## Documents in This Repository

- [`bip-segop.md`](./bip-segop.md) — the full BIP text (primary specification)

---

## Reference Implementation and Extended Specification

The canonical implementation work for segOP—including detailed specification,
P2P behaviour, diagrams, examples, and test vectors—is maintained in:

**https://github.com/defenwycke/segregated-op-return**

That repository includes:

- the full segOP-extended transaction specification,
- TLV format details,
- P2SOP construction and validation,
- node implementation changes,
- pruning model,
- P2P payload relay messages,
- test suite and examples.

This `bip-segop` repository intentionally contains **only the BIP text**, keeping
review and discussion focused on the proposal itself.

---

## License

BSD-2-Clause License.
