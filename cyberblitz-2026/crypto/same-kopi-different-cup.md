# Same Kopi, Different Cup

| | |
|---|---|
| **Category** | Crypto |
| **Points** | 50 |
| **Author** | Anonymous |
| **Solves** | 46 |
| **Flag** | `CYBLTZ2026{54m3_571ck_n3w_r3c31pT}` |

## Challenge

> A kopitiam terminal reused something across encrypted receipts. Recover the midnight order from the known order and its matching receipt.

**Files provided**

- `known_order.txt`: a plaintext order
- `receipts.txt`: four hex-encoded receipts (`known`, `morning`, `noon`, `midnight`)

```
ORDER|table=07|drink=kopi-o-kosong|status=paid|
```

## TL;DR

The terminal encrypts every receipt by XORing it with the **same keystream** (a two-time pad). Because we know one plaintext and its ciphertext, we can recover the keystream and decrypt every other receipt.

## Analysis

A stream cipher or XOR cipher computes:

```
C = P XOR K
```

If `K` is reused across messages, then for any two receipts:

```
C1 XOR C2 = P1 XOR P2
```

The keystream cancels out. With one known pair `(P_known, C_known)`, the keystream falls out directly:

```
K = P_known XOR C_known
```

That keystream then decrypts any other receipt:

```
P_other = C_other XOR K
```

### Observations

- The `known` ciphertext is 47 bytes, which matches the length of the known order without the trailing newline.
- The other receipts are shorter (39, 38 and 34 bytes), so the recovered keystream covers all of them.
- Comparing the receipts byte by byte shows they share a long common prefix, which is consistent with a reused keystream and similar plaintext structure.
- The keystream repeats every 16 bytes.

## Solution

```python
#!/usr/bin/env python3

known_pt = open("known_order.txt", "rb").read().rstrip(b"\n")

receipts = {}
for line in open("receipts.txt"):
    if "=" in line:
        name, hexstr = line.strip().split("=", 1)
        receipts[name] = bytes.fromhex(hexstr)

# Recover the keystream from the known plaintext/ciphertext pair
keystream = bytes(c ^ p for c, p in zip(receipts["known"], known_pt))

# Decrypt every other receipt
for name in ("morning", "noon", "midnight"):
    ct = receipts[name]
    pt = bytes(c ^ k for c, k in zip(ct, keystream))
    print(f"{name:9}: {pt.decode()}")
```

### Output

```
morning  : ORDER|table=03|drink=teh-c|status=paid|
noon     : ORDER|table=11|drink=milo|status=void|
midnight : CYBLTZ2026{54m3_571ck_n3w_r3c31pT}
```

The morning and noon receipts decrypting to well-formed orders confirms the recovered keystream is correct.

## Flag

```
CYBLTZ2026{54m3_571ck_n3w_r3c31pT}
```

## Takeaways

- Never reuse a keystream, nonce or one-time pad key. Reuse breaks confidentiality completely.
- A single known plaintext is enough to decrypt every other message encrypted under the same keystream.
- Use an authenticated cipher such as AES-GCM or ChaCha20-Poly1305 with a unique nonce per message.
