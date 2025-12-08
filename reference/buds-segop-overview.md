# BUDS + segOP Overview

This document explains how the BUDS and segOP proposals relate to each other
and how they can be used independently or together.

---

## Roles

- **BUDS (Bitcoin Unified Data Standard)**  
  A neutral, non-consensus classification framework for describing
  non-consensus data inside Bitcoin transactions.
  - defines labels (e.g., `pay.standard`, `meta.indexer_hint`, `da.obfuscated`)
  - defines tiers (T0–T3) and optional ARBDA scoring
  - does not change transaction formats or consensus rules

- **segOP (Segregated OP_RETURN)**  
  A soft-fork proposal adding a dedicated, fully-priced, prunable data lane.
  - extends the transaction format with a post-witness segOP section
  - requires TLV-structured payloads
  - commits the payload via a P2SOP OP_RETURN output
  - charges 4 WU per segOP byte
  - allows optional pruning of payload bytes

---

## How they fit together

BUDS and segOP are **independent**:

- BUDS can be implemented today on existing transactions (no consensus change).
- segOP can be deployed without BUDS; payloads remain opaque.

When combined:

- segOP provides the **where**:  
  a single, explicit, structured lane for non-consensus data.

- BUDS provides the **what**:  
  a shared vocabulary for describing the contents (labels, tiers).

Optional BUDS TLVs inside the segOP payload allow:

- node implementations to classify data without deep inspection,
- explorers and monitoring tools to present consistent views,
- miners/mempools to build *local, non-consensus* policies around different
  data tiers if desired.

---

## Policy and future work (informative)

Neither BUDS nor segOP dictate fee policies or limits.  
They deliberately avoid encoding any specific stance on “spam”, inscriptions,
or arbitrary data.

Instead, they aim to:

- make non-consensus data explicit (segOP),
- make that data legible (BUDS).

This gives future node implementations, miners, and tooling the option—without
further consensus changes—to:

- apply differentiated fee policies,
- define clearer eviction rules,
- shape block templates with better information,
- and reason about long-term resource usage.

Any such policies would remain local and non-consensus.

---

BUDS and segOP are therefore best viewed as **infrastructure** proposals:
they provide structure and vocabulary, not enforcement.
