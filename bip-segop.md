BIP: TBD
Title: segOP — Segregated OP_RETURN Data Lane
Author: Defenwycke
Status: Draft
Type: Standards Track
License: BSD-2-Clause
Created: 2025-12-08

# Abstract

segOP is a soft-fork proposal that introduces a dedicated, prunable, fully-priced
data lane for non-consensus data in Bitcoin transactions.

It adds:

- a post-witness **segOP section** (TLV-structured),
- a dedicated **P2SOP** (Pay-to-segOP) OP_RETURN output committing to the payload,
- strict consensus rules binding the payload to the P2SOP commitment,
- bounded, fee-paying, structured payload bytes (4 WU/byte),
- optional pruning of payload bytes after a validation window.

The goal is to give metadata, proofs, protocol anchors, and arbitrary payloads
a **single explicit home**, rather than mixing them with validation-critical
surfaces such as witness and script.

segOP is designed to be fully composable with the Bitcoin Unified Data Standard
(BUDS), which provides optional classification for non-consensus data. segOP
itself does not require or impose any specific policy.

# Motivation

Bitcoin is and remains a peer-to-peer electronic cash system. Over time,
legitimate monetary constructions have required small structured anchors,
including:

- Lightning hints and channel state coordination,
- vault and covenant-like constructions,
- rollup roots and L2 proofs,
- payment metadata for multi-party protocols,
- compact audit receipts.

Because Bitcoin does not provide a dedicated data lane, these anchors are today
mixed with:

- large opaque binary payloads (JPEGs, vendor blobs, “runes” and similar),
- text notes and inscriptions,
- arbitrary metadata from a variety of external systems,
- and content that may present legal or operational risk to node operators.

The network has **no native distinction** between:

- validation-critical data,
- small protocol metadata,
- large arbitrary payloads,
- or potentially harmful content.

From the node’s perspective, all of these appear as undifferentiated byte arrays
in witness, script, or OP_RETURN.

This causes several long-standing issues:

- **Policy ambiguity:** fee rules, eviction rules, and mempool limits cannot
  distinguish structured protocol metadata from large arbitrary blobs.

- **Witness discount distortion:** arbitrary payloads can exploit discounted
  surfaces, impairing fee-market clarity.

- **Resource uncertainty:** nodes must store and relay opaque, potentially large
  or harmful data with no clear boundary.

- **No structured pruning:** node operators lack tools to manage storage without
  risking validation correctness.

segOP addresses these issues with minimal, well-scoped changes:

1. A **single explicit lane** for all non-consensus payloads.  
2. **Fully-priced** bytes (4 WU/byte), eliminating incentive distortions.  
3. **Mandatory TLV structure**, ensuring payloads are well-formed and bounded.  
4. A **binding commitment** via P2SOP, ensuring integrity even after pruning.  
5. A path for **safe optional pruning**, lowering storage burden.

segOP is intentionally not designed to be cheaper or easier than existing embedding techniques. The goal is to provide an explicit, policy-defined location for non-monetary data with predictable validation and retention semantics, even at higher cost. For some applications, the incentive is reduced policy risk and clearer operator intent—not fee minimization.

segOP does not classify or judge data—its purpose is structural clarity and safe
handling.

# Design Goals

segOP aims to:

- cleanly separate non-consensus metadata from validation-critical surfaces,
- ensure all payload bytes are fully priced,
- define predictable maximum sizes per transaction,
- allow optional pruning without weakening consensus guarantees,
- preserve `txid` and `wtxid` semantics,
- avoid new script opcodes,
- be compatible with BUDS metadata (but not dependent on it).

# Non-Goals

segOP does not:

- censor or disallow any data types,
- provide fee multipliers or policy rules,
- assert opinions on “spam” or inscriptions,
- require BUDS or any other classification system,
- change script or UTXO semantics.

The BIP explicitly avoids mandating policy.

# High-Level Overview

Extended transactions use the BIP141-style `marker = 0x00` and a new flag bit:

- `0x01` — SegWit (existing)
- `0x02` — segOP present
- Higher bits reserved

A segOP-bearing transaction MUST contain:

- **exactly one segOP section**, post-witness, containing TLV-encoded payload,
- **exactly one P2SOP OP_RETURN output**, committing to the payload.

Payload bytes are charged **4 WU per byte** (no discount). Nodes validate and
may prune payloads after a defined window.

# Specification

## 1. Extended Transaction Format

When `marker = 0x00`, the next byte is a flag bitfield.

segOP extends the BIP141 layout:

- `nVersion`
- `marker = 0x00`
- `flag` (bitfield: 0x01 = witness, 0x02 = segOP)
- `vin`
- `vout`
- witness (if `flag & 0x01`)
- **segOP section** (if `flag & 0x02`)
- `nLockTime`

## 2. segOP Section Layout

If `(flag & 0x02) != 0`, the segOP section MUST appear between witness and
`nLockTime`.

Format:

- `segop_marker` — 1 byte, constant `0x53` (`'S'`)
- `segop_version` — 1 byte, constant `0x01`
- `segop_len` — CompactSize integer
- `segop_payload` — TLV stream (`segop_len` bytes)

If segOP is signaled but the section is absent or malformed, the transaction is
invalid.

## 3. TLV Structure

Each TLV record has:

- `type` — 1 byte
- `length` — CompactSize
- `value` — byte array

Consensus rules:

- TLVs MUST be well-formed,
- Total size MUST equal `segop_len`,
- `segop_len` MUST NOT exceed `MAX_SEGOP_TX_BYTES`,
- No TLV may overflow or mis-encode.

Types are not interpreted by consensus.

## 4. P2SOP Commitment Output

A segOP-bearing transaction MUST contain one P2SOP output.

Script:

```
OP_RETURN <37-byte push>
  "P2SOP" || segop_commitment
```

`segop_commitment` is the tagged hash:

```
TAG = SHA256("segop:commitment")
segop_commitment = SHA256(TAG || TAG || segop_payload)
```

This binds payload → commitment → tx → block merkle root.

## 5. 1:1 Mapping Rules

If segOP is present:

- MUST have exactly one segOP section.  
- MUST have exactly one P2SOP output.  
- Commitment MUST match segop_payload.  
- TLV must be valid.  
- Size must be within consensus limits.

If segOP is absent:

- MUST NOT include any P2SOP output.

## 6. Fees and Weight

All segOP bytes are charged:

**4 weight units per byte.**

This ensures:

- fairness between payment transactions and data-heavy use cases,
- no witness-discount exploitation,
- predictable mempool and block construction behaviour.

segOP does not eliminate the possibility of users placing data in witness;
rather, it provides a fully-priced, structured alternative and a clearer target
for future policy.

## 7. Pruning

Nodes:

- MUST validate full segOP payloads on block acceptance,  
- MUST retain them for a validation window (e.g., 288 blocks, ~2 days),  
- MAY prune payload bytes thereafter,  
- MUST keep all consensus-critical data (tx header, Merkle root, P2SOP).

Shorter windows are not implied to be reorg-safe and are not specified by this proposal.

Pruned payloads may be fetched from archival peers via dedicated P2P messages
(as defined in the extended spec).

### Optional payload omission during IBD (non-normative; future work)
The P2SOP anchor and commitment-hash structure makes it possible in principle to allow nodes to defer retrieval of historical segOP payload bytes during initial block download (IBD), once the commitment has been validated.

However, this proposal does not specify the semantics required for such a mode (commitment-only validation rules, reorg handling, availability guarantees, DoS considerations, etc.). Therefore, as written, this proposal treats segOP payload processing as required during IBD.

Any future payload-omission mechanism would be opt-in and role-dependent (e.g., archive nodes retain full payloads; operator nodes may choose to omit historical payload bytes).

# Interaction with BUDS (Informative)

segOP is fully compatible with the Bitcoin Unified Data Standard (BUDS).

Optional TLVs MAY include:

- **BUDS Tier Marker TLV** (T0–T3),
- **BUDS Label TLV** (canonical BUDS classification),
- **ARBDA metadata**, if chosen by implementations.

These TLVs:

- are **non-consensus**,
- have no effect on segOP validity,
- allow miners/mempools/wallets to classify payloads without parsing arbitrary
  data,
- provide a clean foundation for differentiated fee or eviction policies.

segOP does not depend on BUDS. Nodes that do not implement BUDS ignore these
TLVs.

# How segOP + BUDS Enable Future Policy (Informative)

segOP and BUDS do **not** impose any limits or fee multipliers themselves.

However, they create the structural separation needed for future improvements
in Bitcoin policy.

By providing:

- a **dedicated data lane** (segOP),
- **full-fee accounting** (segOP),
- **structured TLV framing** (segOP),
- **optional semantic classification** (BUDS),

Core and other node implementations can—*in the future if desired*—apply
differentiated mempool or block assembly rules *without touching consensus* and
without relying on brittle heuristics.

Examples (non-consensus, optional):

- different block-weight limits for segOP vs witness,
- different miner preferences for specific data tiers,
- eviction rules sensitive to data class rather than raw size,
- saner handling of large arbitrary payloads.

This BIP explicitly does **not** prescribe such rules; it merely enables them.

# Backwards Compatibility

Legacy nodes:

- ignore segOP section entirely,
- treat P2SOP as a standard OP_RETURN,
- see segOP-bearing blocks as valid.

`txid` and `wtxid` remain unchanged.

This satisfies soft-fork safety: all segOP-valid blocks are legacy-valid.

# Security Considerations

- **Validation cost** — TLV parsing and one tagged hash per segOP tx; bounded.  
- **DoS resistance** — strict size limits prevent pathological payloads.  
- **Binding** — altering any payload byte changes the P2SOP output and block hash.  
- **P2P** — payload relay is isolated from consensus logic; bandwidth-limited.

# Deployment

Activation via versionbits (BIP8/BIP9 style). Parameters left to ecosystem
consensus.

# Reference Implementation

Canonical extended specification, tests, and node changes:

https://github.com/defenwycke/segregated-op-return

# License

BSD-2-Clause License.
