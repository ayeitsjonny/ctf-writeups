# Eight Registers, No Compiler

| | |
|---|---|
| **Category** | Reverse engineering |
| **Points** | 50 |
| **Solves** | 49 |
| **Author** | Anonymous |
| **Flag** | `CYBLTZ2026{31gh7_r3G5_0n3_v01c3}` |

## Challenge

> A compiler vanished, leaving an eight-register interpreter and a courier program. Work out what the program says.

The handout is three EVMC bytecode images, with no interpreter source:

- `hello.vm`: prints `HI`
- `math.vm`: ends with visible registers `r0=250 r1=11 r2=5 r3=14`
- `courier.vm`: the target program

## Approach

There is no VM binary to reverse, so the two samples with known behaviour act as a Rosetta stone. Each sample is tiny, which makes the opcode meanings easy to infer, and `courier.vm` is built from the same opcodes.

### 1. File format

Every image starts with the 4-byte magic `EVMC`. The rest is a stream of fixed-width 4-byte instructions:

```
[ op ] [ a ] [ b ] [ c ]
```

### 2. Decoding `hello.vm` (24 bytes)

```
17 00 48 00    LOADI r0, 0x0048     ; 'H'
8a 00 00 00    OUT   r0
...                                 ; same pattern for 'I'
ff 00 00 00    HALT
```

This gives three opcodes:

| Opcode | Meaning |
|--------|---------|
| `0x17` | `LOADI ra, imm16` (imm16 = `b | c << 8`) |
| `0x8a` | `OUT ra` (print register as a byte) |
| `0xff` | `HALT` |

### 3. Decoding `math.vm` (24 bytes)

```
17 00 fa 00    LOADI r0, 250
17 01 0b 00    LOADI r1, 11
39 02 00 01    r2 = r0 ? r1     ; result 5
52 03 02 01    r3 = r2 ? r1     ; result 14
```

Working out the two unknown operations from the final register values:

- `250 + 11 = 261`, and `261 mod 256 = 5`, so `0x39` is an **8-bit wrapping ADD**.
- `5 XOR 11 = 0b0101 ^ 0b1011 = 0b1110 = 14`, so `0x52` is **XOR**.

### 4. Emulating `courier.vm` (788 bytes)

With four opcodes known (`LOADI`, `ADD`, `XOR`, `OUT`) plus `HALT`, a short emulator is enough to run the courier program. The program loads constants into registers, mixes them with add and xor, and outputs the result one byte at a time.

```python
data = open("courier.vm", "rb").read()[4:]   # skip "EVMC" magic
r = [0] * 8
out = []

for i in range(0, len(data), 4):
    op, a, b, c = data[i:i + 4]
    if op == 0x17:            # LOADI
        r[a] = b | (c << 8)
    elif op == 0x39:          # ADD (mod 256)
        r[a] = (r[b] + r[c]) & 0xFF
    elif op == 0x52:          # XOR
        r[a] = r[b] ^ r[c]
    elif op == 0x8A:          # OUT
        out.append(r[a])
    elif op == 0xFF:          # HALT
        break

print(bytes(out).decode())
```

Output:

```
CYBLTZ2026{31gh7_r3G5_0n3_v01c3}
```

## Flag

```
CYBLTZ2026{31gh7_r3G5_0n3_v01c3}
```

## Takeaways

- When you only have bytecode, small samples with known output are the fastest way to recover the instruction set.
- Choose samples that exercise different operations: `hello.vm` gave load, output and halt, while `math.vm` gave add and xor.
- Final register values are enough to tell wrapping arithmetic from plain arithmetic (`250 + 11 = 5` only makes sense mod 256).
