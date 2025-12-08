# Reference Implementations and Documentation

This repository contains the draft BIP text for **segOP — Segregated OP_RETURN
Data Lane**. The main implementation work and extended documentation live in
the dedicated segOP implementation repository:

- `segregated-op-return` (or equivalent segOP implementation repo)

## Reference implementation

The implementation repo is expected to host:

- the canonical segOP specification,
- wire format details and diagrams,
- reference code changes for node software,
- experimental policy rules and configuration,
- and test vectors / lab tooling.

## Relationship to BUDS

segOP is designed to be composable with the **Bitcoin Unified Data Standard
(BUDS)**, whose BIP and implementation live in separate repositories:

- `bip-buds` — BIP text for BUDS
- `bitcoin-unified-data-standard` — BUDS implementation and registry

BUDS provides a neutral taxonomy and classification framework for non-consensus
data, while segOP provides a dedicated on-chain data lane. Together, they aim
to make Bitcoin's non-payment data more explicit, analysable, and manageable,
without imposing global policy.
