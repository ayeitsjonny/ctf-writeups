# FRAUDE-CODE — CTF Writeup

| | |
|---|---|
| **Challenge** | FRAUDE-CODE |
| **Author** | @canairo |
| **Category** | Reverse Engineering |
| **Points** | 466 |
| **Solves** | 12 |
| **File** | `fraude` (ELF 64-bit, PIE, stripped) |
| **Flag** | `CYBLTZ2026{please_bro_just_ten_more_subagents___please_just_five_million_dollars_more_in_tokens_bro}` |

> *"opus 6.7 make no mistakes"*

## TL;DR

`fraude` is a parody ncurses "autonomous coding agent". It hides a three-layer matryoshka of ELF binaries:

1. **Stage 1 (`fraude`)** decrypts an embedded ELF with a rolling XOR and runs it from an in-memory file (`memfd_create`).
2. **Stage 2 ("subagent")** takes a 100-character argument, splits it into 25 chunks of 4 characters, and spawns 25 embedded ELFs, one per chunk. All 25 must exit `0`.
3. **Stage 3 ("sub-subagents")** are 25 identical programs that differ only in one 32-bit constant. Each runs an invertible xorshift chain over its 4-byte chunk and compares the result with that constant.

Because every step of the chain is invertible, we read the 25 constants and run the chain backwards. The concatenated result is the flag.

## 1. Initial triage

```console
$ file fraude
fraude: ELF 64-bit LSB pie executable, x86-64, dynamically linked, stripped
$ strings -n 6 fraude
...
FraudeCodeChudGPTv6.7_AgentKey!!
subagent
/proc/self/fd/%d
Please enter your chud ass prompt
! ! !  SAFEGUARD TRIGGERED  ! ! !
* * *  25 SUBAGENTS APPROVED  * * *
Your query passes our safeguards! Your answer:
```

Things that stand out:

- Imports of `getmouse`, `mvprintw`, `wgetch`, and similar mean an ncurses UI. This is mostly decoration.
- `memfd_create`, `ftruncate`, `fork`, `execl`, `waitpid`, and the format string `/proc/self/fd/%d` suggest an embedded executable that is written to an anonymous in-memory file and executed.
- The 32-character string `FraudeCodeChudGPTv6.7_AgentKey!!` looks like a key.
- "25 SUBAGENTS APPROVED" hints at the number of child processes.
- There is an unusually large custom section:

```console
$ readelf -S -W fraude | grep rsrc
  [17] .rsrc  PROGBITS  00000000000042b0  0042b0  065050  00  A
```

`.rsrc` is about 413 KB, which is far too big for the UI text. It is the payload.

## 2. Stage 1: decrypting `.rsrc`

The function at `0x2379` is a simple byte-wise decryptor. The key is at `0x40e0`, and the loop index is `i`:

```text
k      = (key[i mod 32] + i) & 0xff
out[i] = in[i] ^ k
```

The function at `0x23f0` allocates a buffer of `0x69300 - 0x42b0 = 0x65050` bytes (the size of `.rsrc`) and decrypts into it. It then calls `memfd_create`, `ftruncate`, and `write` to place the decrypted bytes into an in-memory file, and `fork`s. The child runs `execl("/proc/self/fd/N", "subagent", <prompt>, NULL)`. The parent `waitpid`s and treats exit code `0` as success, which makes the UI show "SUBAGENTS APPROVED".

So the "prompt" you type is just argv[1] of the decrypted program. Decrypting offline:

```python
d   = open("fraude", "rb").read()
key = b"FraudeCodeChudGPTv6.7_AgentKey!!"
rs  = d[0x42b0 : 0x42b0 + 0x65050]
out = bytes(b ^ ((key[i % 32] + i) & 0xff) for i, b in enumerate(rs))
open("sub.elf", "wb").write(out)
```

The result is a valid, non-stripped ELF (`subsubagent.c` appears in its symbol table, which is a hint that there is another layer).

## 3. Stage 2: the subagent

Reading `main` of `sub.elf`:

- Requires `argc == 2` and `strlen(argv[1]) == 0x64` (**100 characters**).
- Computes `size = 0x463450 - 0x402010 = 0x61440` and divides by 25 (the `0xa3d70a3d70a3d70b` magic multiply), giving **25 blobs of 15,936 bytes each**, stored in `.rodata` from `0x402010`.
- For `i` in `0..24`:
  - creates a memfd and writes blob `i` to it,
  - `fork`s and runs it as `execl("/proc/self/fd/N", ..., chunk_i, NULL)`, where `chunk_i` is the 4 bytes at `argv[1] + 4*i`, NUL-terminated.
- Waits for all 25 children. Success requires every child to exit with status `0`.

So the flag is 100 characters, and each 4-character slice is validated independently by its own binary.

```python
d = open("sub.elf", "rb").read()
base = 0x2010                       # file offset of vaddr 0x402010
n = (0x463450 - 0x402010) // 25     # 15936
for i in range(25):
    open(f"b{i}", "wb").write(d[base + i*n : base + (i+1)*n])
```

## 4. Stage 3: the 25 checkers

The 25 extracted binaries are the same program. Comparing them byte for byte shows only two differences: the 20-byte build-id (`0x378..0x38b`) and the **4-byte immediate** in a `cmp eax, imm32` at file offset `0x122e`.

Disassembly of the checker's `main`:

```asm
cmp   argc, 2            ; else exit(-1)
cmp   strlen(argv[1]), 4 ; else exit(-1)
mov   eax, [argv[1]]     ; x = uint32 little-endian of the 4 chars

; step 1: for i = 13 down to 2:  x ^= x >> i
; step 2: for i = 25 down to 2:  x ^= x << i
; step 3: for i = 7  down to 2:  x ^= x >> i

cmp   x, 0xbe47d89f      ; <- per-binary constant (this one is b0)
jne   fail               ; exit(-1)
xor   eax, eax           ; exit(0)
```

In pseudo-C, with 32-bit unsigned arithmetic:

```c
uint32_t x = *(uint32_t *)argv[1];
for (int i = 13; i > 1; i--) x ^= x >> i;
for (int i = 25; i > 1; i--) x ^= x << i;
for (int i = 7;  i > 1; i--) x ^= x >> i;
return x == TARGET ? 0 : -1;
```

## 5. Inverting the xorshifts

Each elementary operation `x ^= x >> i` or `x ^= x << i` is a bijection on 32-bit integers. It is a lower- or upper-triangular linear map over GF(2), with ones on the diagonal. We can therefore invert the whole chain by undoing the steps in reverse order.

The inverse of a single `y = x ^ (x >> i)` can be found by fixed-point iteration, because the top `i` bits of `x` equal the top `i` bits of `y`, and each further iteration fixes `i` more bits:

```python
x = y
for _ in range(32 // i + 1):
    x = y ^ (x >> i)
```

The same works for `<<`, with a 32-bit mask.

## 6. Solve script

```python
#!/usr/bin/env python3
import struct

M = 0xFFFFFFFF

def fwd(x):
    for i in range(13, 1, -1): x ^= x >> i
    for i in range(25, 1, -1): x = (x ^ (x << i)) & M
    for i in range(7, 1, -1):  x ^= x >> i
    return x

def inv_shr(y, i):
    x = y
    for _ in range(32 // i + 1):
        x = y ^ (x >> i)
    return x

def inv_shl(y, i):
    x = y
    for _ in range(32 // i + 1):
        x = (y ^ (x << i)) & M
    return x

def inv(y):
    x = y
    for i in range(2, 8):  x = inv_shr(x, i)   # undo step 3
    for i in range(2, 26): x = inv_shl(x, i)   # undo step 2
    for i in range(2, 14): x = inv_shr(x, i)   # undo step 1
    return x

# --- unpack the layers ---
d   = open("fraude", "rb").read()
key = b"FraudeCodeChudGPTv6.7_AgentKey!!"
rs  = d[0x42b0 : 0x42b0 + 0x65050]
sub = bytes(b ^ ((key[i % 32] + i) & 0xff) for i, b in enumerate(rs))

base = 0x2010
n    = (0x463450 - 0x402010) // 25          # 15936 bytes per checker

flag = b""
for k in range(25):
    blob   = sub[base + k*n : base + (k+1)*n]
    target = struct.unpack("<I", blob[0x122e:0x1232])[0]
    x      = inv(target)
    assert fwd(x) == target
    flag  += struct.pack("<I", x)

print(flag.decode())
```

Output:

```text
CYBLTZ2026{please_bro_just_ten_more_subagents___please_just_five_million_dollars_more_in_tokens_bro}
```

## 7. Verification

Running the decrypted stage-2 binary directly with the recovered string makes all 25 checkers pass:

```console
$ ./sub.elf 'CYBLTZ2026{please_bro_just_ten_more_subagents___please_just_five_million_dollars_more_in_tokens_bro}'
$ echo $?
0
```

Entering the same string as the "prompt" in `fraude` gives the "25 SUBAGENTS APPROVED" screen.

## Flag

```text
CYBLTZ2026{please_bro_just_ten_more_subagents___please_just_five_million_dollars_more_in_tokens_bro}
```

## Takeaways

- A large non-standard section (`.rsrc`) together with `memfd_create` + `execl("/proc/self/fd/%d")` is a strong sign of an embedded, self-unpacking payload.
- You do not need to run the binary. The decryptor is short enough to reimplement offline in a few lines of Python.
- Nested layers can be peeled one at a time. A stripped outer layer wrapped a non-stripped inner one, and the inner symbol names (`subsubagent.c`) hinted at the next layer.
- The checker's xorshift chain looks scary but is a bijection. When a check is "transform input, compare with constant", look for invertibility before reaching for a solver.
