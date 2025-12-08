# segOP Reference — Examples

This document gives small worked examples of segOP payloads and the P2SOP
commitment output, plus a few "vector-style" validity cases.

---

## 1. Example TLV payload

A simple segOP payload containing two TLVs:

1. A UTF-8 note (`type = 0x01`)
2. A 32-byte rollup root (`type = 0x20`)

```
+------+------+-------------------------------+
| type | len  | value                         |
+------+------+-------------------------------+
| 0x01 | 0x0b | "hello-world" (11 bytes)      |
| 0x20 | 0x20 | <32-byte rollup_root>         |
+------+------+-------------------------------+
```

In pseudo-hex:

```
01 0b          // type = 0x01, length = 11
68 65 6c 6c 6f 2d 77 6f 72 6c 64   // "hello-world"

20 20          // type = 0x20, length = 32
<32 bytes of rollup_root>
```

`segop_len` in the header must equal the total number of payload bytes.

---

## 2. Example P2SOP output

Given `segop_payload` as above, the P2SOP commitment is:

```
TAG = SHA256("segop:commitment")
segop_commitment = SHA256(TAG || TAG || segop_payload)
```

The P2SOP output script:

```
OP_RETURN
PUSHDATA(37)
  "P2SOP" || segop_commitment
```

In hex (illustrative):

```
6a
25
50 32 53 4f 50
<32-byte segop_commitment>
```

---

## 3. Validity scenarios (vector-style)

These examples describe when a segOP transaction is valid or invalid. They are
not full raw test vectors, but they map directly onto the consensus checks.

### 3.1 Valid segOP tx (minimal example)

- `marker = 0x00`
- `flag = 0x02` (segOP only)
- 1 input, 2 outputs:
  - 1 normal payment output,
  - 1 P2SOP output with correct commitment.
- segOP section present with:
  - `segop_marker = 0x53`
  - `segop_version = 0x01`
  - TLV payload of length `segop_len`
  - `segop_len <= MAX_SEGOP_TX_BYTES`
- P2SOP commitment matches `segop_payload`.

**Result:** Valid under segOP rules.

---

### 3.2 Invalid: segOP flag set but P2SOP missing

- `marker = 0x00`
- `flag = 0x02`
- segOP section present and TLV-correct,
- **No** P2SOP output.

**Result:** Invalid — segOP requires exactly one P2SOP output when the flag is set.

---

### 3.3 Invalid: P2SOP present but segOP flag unset

- `marker = 0x00`
- `flag` does **not** have bit `0x02` set,
- No segOP section,
- Output script matches P2SOP pattern.

**Result:** Invalid — P2SOP outputs are only allowed when segOP is signalled.

---

### 3.4 Invalid: TLV framing error

- segOP flag set,
- segOP section present,
- `segop_len` claims 64 bytes,
- TLV records decode to only 60 bytes (or overrun).

**Result:** Invalid — TLV must consume exactly `segop_len` bytes.

---

These examples are intentionally small and conceptual.  
The extended spec and implementation repo contain full serialization-level test
cases and end-to-end vectors.
