# The Rain Has a Public Key

| | |
|---|---|
| **Category** | Crypto |
| **Points** | 50 |
| **Author** | Anonymous |
| **Solves** | 42 |
| **Files** | `rain_reports.json` (plus `known_order.txt` and `receipts.txt` from *Same Kopi, Different Cup*) |
| **Flag** | `CYBLTZ2026{dry_53n50R_w37_51gn47ur3}` |

## Challenge

> A weather station's dry-sensor fault may have changed more than its rainfall reports. Keep the public receipts from Same Kopi, Different Cup to open the seal.

The station's signing and sealing scheme:

```text
Signature: R = g^k mod p; s = (k + e*x) mod q.
Challenge bytes = UTF-8(message) || ASCII('|') || ASCII(decimal(R), no leading zeroes).
e = int.from_bytes(SHA256(challenge bytes).digest(), 'big') mod q; no newline is hashed.
Dry-sensor fault: k2-k1 = (rainfall2-rainfall1)*65537 mod q.
Legacy binding = first 16 bytes of the XOR keystream recovered from the known receipt in Same Kopi, Different Cup.
Receipt key = SHA256(ASCII(decimal(x)) || legacy binding).digest() (raw 32 bytes).
The sealed field is base64 of the flag bytes XORed with the repeating receipt key.
```

`rain_reports.json` contains the group parameters (`p`, `q`, `g`), the public key, two signed rain reports, and the `sealed` blob.

## Solution overview

1. Recover the Schnorr private key `x` from the two signatures using the known nonce relationship.
2. Recover the legacy binding from the keystream reuse in *Same Kopi, Different Cup*.
3. Derive the receipt key and decrypt `sealed`.

## Step 1: Recovering the private key

Both reports are Schnorr-style signatures, so each satisfies `s = k + e*x (mod q)`. For two reports:

```text
s1 = k1 + e1*x
s2 = k2 + e2*x
```

Subtracting gives:

```text
s2 - s1 = (k2 - k1) + (e2 - e1)*x   (mod q)
```

Normally `k1` and `k2` are unknown random nonces, so this reveals nothing. Here the dry-sensor fault leaks their difference: `k2 - k1 = (rainfall2 - rainfall1) * 65537 mod q`. The reports have rainfall 0 and 7, so `k2 - k1 = 7 * 65537`. That leaves `x` as the only unknown:

```text
x = (s2 - s1 - 7*65537) * (e2 - e1)^-1   (mod q)
```

Each challenge `e` is computed as specified: SHA256 of `message || '|' || decimal(R)`, interpreted big-endian, reduced mod `q`.

Result:

```text
x = 17151493350269726687387740635858725324408677815697464406168450594924795787459
```

Sanity check: `pow(g, x, p)` equals the `public` value in the JSON.

## Step 2: The legacy binding

In *Same Kopi, Different Cup*, the receipts are XOR-encrypted with a reused keystream. The known receipt and its plaintext (`known_order.txt`) give that keystream directly:

```text
keystream = known_order XOR known_receipt
```

`known_order.txt` has a trailing newline, but the known receipt is 47 bytes, so only the first 47 plaintext bytes are used. The legacy binding is the first 16 bytes of the keystream:

```text
93b2fe61649799aa1cda98e2e0eaab80
```

## Step 3: Opening the seal

```text
key  = SHA256(ASCII(decimal(x)) || legacy_binding).digest()   # 32 raw bytes
flag = base64decode(sealed) XOR key repeated
```

The sealed blob is 36 bytes, so the 32-byte key wraps around for the last 4 bytes.

## Solve script

```python
import json, hashlib, base64

d = json.load(open("rain_reports.json"))
p, q, g, pub = d["p"], d["q"], d["g"], d["public"]

def challenge(r):
    b = r["message"].encode() + b"|" + str(r["R"]).encode()
    return int.from_bytes(hashlib.sha256(b).digest(), "big") % q

r1, r2 = d["reports"]
e1, e2 = challenge(r1), challenge(r2)
dk = (r2["rainfall"] - r1["rainfall"]) * 65537

x = ((r2["s"] - r1["s"] - dk) * pow(e2 - e1, -1, q)) % q
assert pow(g, x, p) == pub

# Keystream from Same Kopi, Different Cup
known_order = open("known_order.txt", "rb").read()
known_receipt = None
for line in open("receipts.txt"):
    name, val = line.strip().split("=")
    if name == "known":
        known_receipt = bytes.fromhex(val)

keystream = bytes(a ^ b for a, b in zip(known_order, known_receipt))
binding = keystream[:16]

key = hashlib.sha256(str(x).encode() + binding).digest()
ct = base64.b64decode(d["sealed"])
print(bytes(c ^ key[i % 32] for i, c in enumerate(ct)).decode())
```

## Takeaways

- Schnorr signatures are only safe if every nonce is fresh and unpredictable. A known (or even partially known) relationship between two nonces turns the signing equations into a solvable linear system for the private key.
- Reusing an XOR keystream across messages lets anyone with one known plaintext recover the keystream for all of them.
- Chaining the two leaks is what opened the seal: one gave the key material, the other gave the binding.
