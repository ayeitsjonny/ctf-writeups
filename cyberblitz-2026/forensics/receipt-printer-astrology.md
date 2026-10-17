# Receipt Printer Astrology — CTF Writeup

| | |
|---|---|
| **Challenge** | Receipt Printer Astrology |
| **Points** | 466 |
| **Author** | Anonymous |
| **Solves** | 12 |
| **Category** | Forensics |
| **Files** | `uart_capture.txt`, `font_table.txt` |

**Flag:** `CYBLTZ2026{574r_pr1n73R_457r0l0gy}`

## Challenge description

A receipt printer sent static during the blackout, but its serial capture survived. We have to recover the "constellation" it tried to print.

We are given the logic capture (`uart_capture.txt`, one CSV sample per UART bit period) and the exact RP-5x7 printer font (`font_table.txt`).

**UART sampling**

- Each CSV row is exactly one UART bit period (no oversampling or baud-rate recovery needed).
- The capture begins with eight idle-high samples.
- Each frame is **8E1**: start = 0, eight data bits LSB-first, even parity, stop = 1.
- Every 11-bit frame is followed by exactly one idle-frame sample, so start bits are 12 samples apart.

**RAST format**

| Offset | Size | Meaning |
|---|---|---|
| `0x00` | 4 | ASCII `RAST` |
| `0x04` | 2 | width, u16 little-endian |
| `0x06` | 2 | height, u16 little-endian |
| `0x08` | … | packed 1-bit raster |

- Raster bits are packed continuously in row-major order, with no per-row padding.
- Exactly `width * height` bits are used; trailing bits in the last byte are zero.
- Each character is five columns followed by one blank separator column (5x7 cells).

## Solution

### 1. Decode the UART frames

Skip the 8 idle samples, then read an 11-bit frame every 12 samples. For each frame:

- check `start == 0` and `stop == 1`
- take bits 1–8 as the data byte, LSB-first
- check even parity over the 8 data bits plus the parity bit

### 2. The glitch frame (the trap)

Decoding naively gives the header `RAST 00 CC 00 07`, which would mean a width of `0xCC00` and a height of `0x0700` — nonsense.

The frame starting at sample 56 is a spurious `0x00` whose **parity bit is wrong** (data bits are all 0, parity bit is 1). It sits between the `RAST` magic and the width field and shifts every later byte by one. Dropping frames that fail the parity check fixes the alignment:

```
52 41 53 54 | CC 00 | 07 00 | ...179 raster bytes
R  A  S  T  | w=204 | h=7
```

Sanity check: `204 * 7 = 1428` bits = 178.5 bytes, rounded up to **179** bytes, which is exactly how many bytes remain.

### 3. Unpack the raster

Unpack the bits **MSB-first** per byte, in row-major order, and keep the first `204 * 7` bits. MSB-first renders clean glyph shapes; LSB-first produces garbage.

### 4. Match glyphs against the font

The width of 204 is 34 cells of 6 columns (5 glyph columns + 1 separator). For each cell, build the seven 5-bit rows and look them up in `font_table.txt`. All 34 cells match exactly:

```
CYBLTZ2026{574r_pr1n73R_457r0l0gy}
```

## Solve script

```python
import csv

# --- 1. read samples ---
samples = [int(r["level"]) for r in csv.DictReader(open("uart_capture.txt"))]

# --- 2. decode 8E1 frames, 12 samples apart, dropping bad frames ---
data = []
i = 8  # skip eight idle-high samples
while i + 11 <= len(samples):
    f = samples[i:i + 11]
    bits = f[1:9]
    parity = f[9]
    ok = f[0] == 0 and f[10] == 1 and (sum(bits) + parity) % 2 == 0
    if ok:
        data.append(sum(b << k for k, b in enumerate(bits)))  # LSB-first
    i += 12

blob = bytes(data)
assert blob[:4] == b"RAST"
w = int.from_bytes(blob[4:6], "little")
h = int.from_bytes(blob[6:8], "little")

# --- 3. unpack raster, MSB-first ---
bits = []
for byte in blob[8:]:
    for k in range(8):
        bits.append((byte >> (7 - k)) & 1)
grid = [bits[r * w:(r + 1) * w] for r in range(h)]

# --- 4. load font and match 5x7 cells ---
font = {}
for line in open("font_table.txt"):
    if line.startswith("#"):
        continue
    _, ch, rows = line.rstrip("\n").split("\t")
    font[rows] = ch

out = ""
for c in range(0, w, 6):
    rows = "/".join(
        "".join(str(grid[r][c + k]) for k in range(5)) for r in range(h)
    )
    out += font.get(rows, "?")

print(out)
```

## Takeaways

- Always validate framing and parity. The one corrupted frame was the whole challenge.
- If the decoded header values look absurd (width `0xCC00`), suspect a byte-alignment problem before suspecting the format.
- When the bit order is ambiguous, render both candidates and see which one produces recognizable shapes.
