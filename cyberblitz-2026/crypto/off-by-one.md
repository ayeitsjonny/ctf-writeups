# Off By One

| | |
|---|---|
| **Category** | Crypto |
| **Points** | 453 |
| **Author** | @sbakiras |
| **Solves** | 14 |
| **Files** | `distrib.txt` |

## Challenge

> Our ECDSA signing service (with SHA-256 and NIST's P-256 curve) passed its audit with flying colors: "Nonces are never reused." We checked, they aren't. But there's something off... check out the two given signatures and retrieve the private signing key. The flag is obtained by XORing the given ciphertext with the retrieved key.

## Given data

`distrib.txt` contains two signatures over two known messages, plus a ciphertext:

```
m1 = This is message 1
r1, s1 = ...
m2 = This is message 2
r2, s2 = ...
ciphertext = 2b6612d7...1bf1ad0b1
```

## Analysis

The hint is in the title and the description. The nonces are never reused, but they are not independent either. The service uses **consecutive nonces**:

```
k2 = k1 + 1
```

This is just as fatal as reusing a nonce, because it gives us a linear relation between the two unknown nonces.

### Math

ECDSA signing, with private key `d`, group order `n` and `h = SHA-256(m)`:

```
s = k^-1 * (h + r*d)  (mod n)
=> k = s^-1 * (h + r*d)  (mod n)
```

So for the two signatures:

```
k1 = (h1 + r1*d) / s1
k2 = (h2 + r2*d) / s2
```

Substituting into `k2 - k1 = 1`:

```
(h2 + r2*d)/s2 - (h1 + r1*d)/s1 = 1  (mod n)
```

Collect the terms in `d`:

```
d * (r2/s2 - r1/s1) = 1 - h2/s2 + h1/s1  (mod n)
```

This is linear in `d`, so we solve it with a modular inverse:

```
d = (1 - h2/s2 + h1/s1) / (r2/s2 - r1/s1)  (mod n)
```

## Solve script

```python
import hashlib

n = 0xffffffff00000000ffffffffffffffffbce6faada7179e84f3b9cac2fc632551

r1 = 84918391022880580917404599665324292434691745381102244941556045001372735351319
s1 = 87741392621874592924723018144529307098849418024912183482695319500250268744009
r2 = 34231198634407520436945515392215999690091853063969785984537880200814831292997
s2 = 53371404715937007600535556326097711593280131994602709883282028166572336213092
ct = bytes.fromhex("2b6612d7406f3a21d64291885994eaec3502845602262323d33cfb46bf1ad0b1")

h1 = int.from_bytes(hashlib.sha256(b"This is message 1").digest(), "big")
h2 = int.from_bytes(hashlib.sha256(b"This is message 2").digest(), "big")

inv = lambda x: pow(x, -1, n)

# Try k2 = k1 + 1 (delta = 1) and k2 = k1 - 1 (delta = -1)
for delta in (1, -1):
    A = (r2 * inv(s2) - r1 * inv(s1)) % n
    B = (delta - h2 * inv(s2) + h1 * inv(s1)) % n
    d = B * inv(A) % n
    key = d.to_bytes(32, "big")
    print(delta, hex(d), bytes(a ^ b for a, b in zip(ct, key)))
```

## Result

With `k2 = k1 + 1`, the recovered private key is:

```
0x683f509b14350811e474eac636e0b5bf5a5dd6376c424c4e8c729428dc7fa3cc
```

XORing it (as 32 big-endian bytes) with the ciphertext gives readable text. The other direction (`k2 = k1 - 1`) gives garbage, so `+1` is the right one.

## Flag

```
CYBLTZ2026{Not_So_Random_Nonces}
```

## Takeaways

- "Nonces are never reused" is not the same as "nonces are unpredictable". Any known relationship between nonces (consecutive, a fixed offset, a linear function) leaks the private key.
- Prevention: use deterministic nonces per RFC 6979, or a strong CSPRNG, and never derive a nonce from the previous one.
