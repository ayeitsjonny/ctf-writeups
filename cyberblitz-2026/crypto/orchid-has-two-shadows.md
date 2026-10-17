# Orchid Has Two Shadows

**Category:** Crypto | **Points:** 50 | **File:** `orchid_transcript.json`

```text
CYBLTZ2026{0n3_n4m3_7w0_r3g157rY_5l075}
```

## Challenge

An Orchid operator appears twice in a signed share registry. The challenge says to keep the public handouts from *Same Kopi, Different Cup* and *The Rain Has a Public Key* to recover the token.

The share protocol:

- Each operator has an independent degree-1 polynomial `f(slot) = secret + slope * slot mod p`, with `p = 2^127 - 1`.
- The registry assigns each raw UTF-8 identity a new slot. Operators are authenticated by their **Unicode-canonical** identity, so only shares with the same canonical identity belong to the same polynomial.
- Share-wrap key: `SHA256(ASCII(decimal(x)) || first 16 raw keystream bytes)`, where `x` is the signing key from *The Rain Has a Public Key* and the keystream comes from *Same Kopi, Different Cup*.
- For each record, `HMAC-SHA256(wrap key, "ORCHID-SHARE-V2" || 0x00 || UTF-8(canonical identity))` gives a raw digest. Read it as a big-endian integer, reduce mod `p`, and XOR it with `masked_share` to get the share.
- The secret must satisfy `SHA256("ORCHID-CHECK-V2" || 0x00 || secret.to_bytes(16, "big")).hexdigest() == secret_commitment`.
- The sealed token is base64 of the flag XORed with the repeating `SHA256(secret.to_bytes(16, "big")).digest()`.

## Inputs from the earlier challenges

| Value | Source | Result |
|-------|--------|--------|
| First 16 keystream bytes | Known-plaintext XOR of the known receipt in *Same Kopi, Different Cup* | `93b2fe61649799aa1cda98e2e0eaab80` |
| Signing key `x` | Related-nonce Schnorr attack in *The Rain Has a Public Key* | `17151493350269726687387740635858725324408677815697464406168450594924795787459` |

## Analysis

The registry has four records that all look like "orchid":

| Slot | Raw identity | NFKC form |
|------|--------------|-----------|
| 2 | `ＯＲＣＨＩＤ` (fullwidth) | `ORCHID` |
| 4 | `orchid` | `orchid` |
| 6 | `ORCHID` | `ORCHID` |
| 8 | `ｏｒｃｈｉｄ` (fullwidth) | `orchid` |

The tempting move is to casefold all four into one identity and interpolate through four points. That fails: the unmasked shares are not collinear, and no three of them are either.

NFKC normalization alone is the right canonicalization. It collapses the fullwidth forms but keeps `ORCHID` and `orchid` distinct, so there are two operators with two shares each:

- `ORCHID`: slots 2 and 6
- `orchid`: slots 4 and 8

Two points determine a degree-1 polynomial, so each pair gives a candidate secret as `f(0)`. Only the `ORCHID` pair matches `secret_commitment`.

## Solution

```python
import json, hmac, hashlib, base64, unicodedata

x = 17151493350269726687387740635858725324408677815697464406168450594924795787459
legacy = bytes.fromhex("93b2fe61649799aa1cda98e2e0eaab80")

d = json.load(open("orchid_transcript.json", encoding="utf-8"))
p = d["modulus"]
wrap = hashlib.sha256(str(x).encode() + legacy).digest()

def mask(canon):
    msg = b"ORCHID-SHARE-V2" + b"\x00" + canon.encode("utf-8")
    return int.from_bytes(hmac.new(wrap, msg, hashlib.sha256).digest(), "big") % p

# Group records by NFKC-canonical identity and unmask each share
groups = {}
for r in d["registry"]:
    canon = unicodedata.normalize("NFKC", r["raw_identity"])
    share = r["masked_share"] ^ mask(canon)
    groups.setdefault(canon, []).append((r["slot"], share))

for canon, pts in groups.items():
    if len(pts) < 2:
        continue
    (x1, y1), (x2, y2) = pts[:2]
    slope = (y2 - y1) * pow(x2 - x1, -1, p) % p
    secret = (y1 - slope * x1) % p          # f(0)

    check = hashlib.sha256(
        b"ORCHID-CHECK-V2" + b"\x00" + secret.to_bytes(16, "big")
    ).hexdigest()
    if check == d["secret_commitment"]:
        key = hashlib.sha256(secret.to_bytes(16, "big")).digest()
        sealed = base64.b64decode(d["sealed"])
        print(canon, secret)
        print(bytes(b ^ key[i % 32] for i, b in enumerate(sealed)))
```

Output:

```text
ORCHID 147654563376870329779110725285388645954
b'CYBLTZ2026{0n3_n4m3_7w0_r3g157rY_5l075}'
```

## Pitfalls

- **`.hexdigest()` applies to the hash output.** My first attempt hashed the hex string of the secret bytes, so the commitment never matched. The commitment hashes the raw 16 secret bytes, and the digest is then hex-encoded for comparison.
- **Casefold is not the canonical form.** Casefolding merges all four records into one operator, which produces non-collinear shares. NFKC alone gives two operators, and the `ORCHID` pair is the one that checks out.

## Takeaways

1. Canonicalization is a security boundary. Whether two strings count as the same identity depends on exactly which normalization form is applied, and the wrong choice changes which shares belong together.
2. Verify intermediate results against the commitment instead of trusting a single path. Checking collinearity and the commitment is what ruled out the casefold interpretation.
3. Read each spec literally. A raw byte string and its hex encoding hash to completely different values.
