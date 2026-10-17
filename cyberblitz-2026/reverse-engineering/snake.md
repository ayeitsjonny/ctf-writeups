# SNAKE

| | |
|---|---|
| **CTF** | CyberBlitz (organised by SIT) |
| **Category** | Reverse Engineering |
| **Points** | 50 |
| **Solves** | 44 |
| **Author** | @canairo |
| **Flag** | `CYBLTZ2026{s1mplyth000n}` |

> *i actually wanted to make this a proper obfuscator but i got lazy*

**Attachment:** `snake.pyc`

---

## TL;DR

`snake.pyc` is a compiled Python 3.12 module wrapped in a light obfuscator. Stripping the junk opcodes reveals a loader that decodes a hidden script (`swap pairs → base64 → XOR`) and `exec`s it. The hidden script is RC4 with a hardcoded key, compared against a fixed ciphertext. Because RC4 is symmetric, running the same routine on that ciphertext gives the flag.

---

## 1. Recon

```bash
$ file snake.pyc
snake.pyc: Byte-compiled Python module for CPython 3.12 or newer, timestamp-based
```

It's a `.pyc`, so the first step is to load the code object and disassemble it. The first 16 bytes are the header (magic, flags, timestamp, source size), and the rest is a `marshal`-serialised code object.

```python
import marshal, dis

with open("snake.pyc", "rb") as f:
    f.read(16)                  # skip header
    code = marshal.load(f)

dis.dis(code)
print(code.co_consts)
print(code.co_names)
```

> Use the same minor version of Python that compiled the file (3.12 here). Bytecode and the marshal format change between versions.

## 2. Cutting through the noise

The disassembly is long, but most of it is junk that repeats the same shape:

```
LOAD_CONST   0
POP_JUMP_IF_FALSE  -> skip
LOAD_CONST   0
POP_JUMP_IF_TRUE   -> skip
<random opcodes: UNARY_INVERT, CHECK_EXC_MATCH, GET_AITER, ...>
JUMP_FORWARD -> skip
```

These are **opaque predicates**. The condition is a constant that always sends execution to the same place, and the instructions in between are never reached. They can be ignored.

What's left is small:

- a long base64-looking string `s`
- a SHA-1 hex string `q`: `da39a3ee5e6b4b0d3255bfef95601890afd80709` (the hash of the empty string, used here as a 20-byte key)
- two helper functions with scrambled names (`_0x5f3759df_0x7c2a4b8e` and `_0x8b7c6a5d_0x4e3d2c1b`)
- a final call that `exec`s something

## 3. The two helper functions

**`_0x5f3759df_0x7c2a4b8e(x)`** converts the input to a list and loops `range(0, len-1, 2)`, swapping `l[i]` and `l[i+1]`. This **swaps adjacent characters**. It explains why the base64 string ends in `=M` instead of the usual `M=` padding.

**`_0x8b7c6a5d_0x4e3d2c1b(data, key)`** builds a list of `data[i] ^ key[i % len(key)]` and returns it as a `bytearray`. This is a **repeating-key XOR**.

## 4. Reconstructing the loader

Following the `CALL` order in the main module, the original line was:

```python
exec(xor(base64.b64decode(swap(s).encode()), bytes.fromhex(q)))
```

So the string `s` is an encoded Python script. **Do not run `exec` on it.** Rebuild the pipeline and print the result instead:

```python
import base64

def swap(x):
    l = list(x)
    for i in range(0, len(l) - 1, 2):
        l[i], l[i + 1] = l[i + 1], l[i]
    return "".join(l)

def xor(a, k):
    return bytes(a[i] ^ k[i % len(k)] for i in range(len(a)))

payload = xor(base64.b64decode(swap(s).encode()), bytes.fromhex(q))
print(payload.decode())
```

## 5. The hidden script

```python
k = b"opcodeopcodeopco"

def enc(key: bytes, data: bytes) -> bytes:
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]

    i = j = 0
    out = bytearray()
    for byte in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]

        k = S[(S[i] + S[j]) % 256]
        out.append(byte ^ k)

    return bytes(out)

a = input("flag > ")
b = enc(k, a.encode())

if b.hex() == "6b71030aee4b9241454b681386778f3a615b660d0d2f47f4":
    print("correct!")
else:
    print("false!")
```

This is **RC4**. The first loop is the key-scheduling algorithm (KSA), and the second is the pseudo-random generation algorithm (PRGA) that produces a keystream XORed with the input. The key is hardcoded as `opcodeopcodeopco`, and the program checks whether the encrypted input matches a fixed hex string.

## 6. Solving

RC4 just XORs the data with a keystream, and the keystream depends only on the key. XOR is its own inverse, so **encrypting the ciphertext with the same key gives back the plaintext**. There is no need to brute-force or reverse any state.

```python
ct = bytes.fromhex("6b71030aee4b9241454b681386778f3a615b660d0d2f47f4")
print(enc(b"opcodeopcodeopco", ct))
```

```
b'CYBLTZ2026{s1mplyth000n}'
```

## Flag

```
CYBLTZ2026{s1mplyth000n}
```

---

## Takeaways

- **Opaque predicates are easy to spot and skip.** Repeating always-true or always-false jumps around garbage opcodes are noise. Work out what the real instructions do and ignore the rest.
- **Don't `exec` unknown payloads.** Re-implement the decoding steps and print the output instead.
- **Stream ciphers like RC4 are symmetric.** With the key and a ciphertext, the same function decrypts it.
- A hardcoded key plus a fixed comparison value means the check can be inverted directly. No bruteforce needed.

## Tools used

- Python 3.12 (`marshal`, `dis`, `base64`)
