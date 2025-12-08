# segOP Reference — Diagrams

This document gives a visual overview of the segOP transaction layout, the
marker/flag usage, and the P2SOP commitment output.

---

## 1. Extended transaction layout

segOP reuses the BIP141-style marker/flag mechanism and inserts a segOP section
between the witness data and `nLockTime`.

### 1.1 High-level view

```
+-----------------------------------------------------------+
| nVersion (4 bytes)                                        |
+-----------------------------------------------------------+
| marker (0x00)                                             |
+-----------------------------------------------------------+
| flag (bitfield: 0x01 = witness, 0x02 = segOP, etc.)       |
+-----------------------------------------------------------+
| tx_in count (varint)                                      |
+-------------------------+---------------------------------+
| inputs ...              |                                 |
+-------------------------+---------------------------------+
| tx_out count (varint)                                     |
+-------------------------+---------------------------------+
| outputs ...                                               |
+-----------------------------------------------------------+
| witness data (if flag & 0x01)                             |
+-----------------------------------------------------------+
| segOP section (if flag & 0x02)                            |
+-----------------------------------------------------------+
| nLockTime (4 bytes)                                       |
+-----------------------------------------------------------+
```

### 1.2 Marker and flag

marker = 0x00

flag bits:
    0x01  -> SegWit present
    0x02  -> segOP present
    0x04+ -> reserved for future extensions

Examples:
    flag = 0x01  -> SegWit only
    flag = 0x02  -> segOP only
    flag = 0x03  -> SegWit + segOP

## 2. segOP section layout

The segOP section appears after any witness data and before nLockTime.

```
+------------------------------+
| segop_marker   (0x53, 'S')   |
+------------------------------+
| segop_version  (0x01)        |
+------------------------------+
| segop_len      (CompactSize) |
+------------------------------+
| segop_payload  (TLV bytes)   |
+------------------------------+
```

segop_payload is a concatenation of TLV records:

```
+--------+----------------+------------------------+
| type   | length (varint)| value (length bytes)   |
+--------+----------------+------------------------+
| 1 byte | CompactSize    | raw bytes              |
+--------+----------------+------------------------+
```

The TLV stream MUST cover exactly segop_len bytes.

## 3. P2SOP commitment output

Each segOP-bearing transaction has exactly one P2SOP output that commits to the segOP payload. It is an OP_RETURN script with a tagged-hash commitment.

### 3.1 Script structure

```
scriptPubKey:

    OP_RETURN
    <37-byte push>
        "P2SOP" || segop_commitment
```

In hex (illustrative):

```
6a                        OP_RETURN
25                        PUSHDATA(37)
50 32 53 4f 50            'P' '2' 'S' 'O' 'P'
<32 bytes>                segop_commitment
```

### 3.2 Commitment construction

```
TAG = SHA256("segop:commitment")
segop_commitment = SHA256(TAG || TAG || segop_payload)
```

Any change to segop_payload changes the P2SOP output, Merkle root, and block hash, even if payload bytes are later pruned from local storage.

---

These diagrams are non-normative and are provided as an aid to reading the formal BIP and extended specification.
