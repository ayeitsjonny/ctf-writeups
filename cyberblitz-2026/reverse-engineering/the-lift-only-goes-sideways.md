# The Lift Only Goes Sideways

| | |
|---|---|
| **Category** | Reverse Engineering |
| **Points** | 472 |
| **Solves** | 11 |
| **Author** | Anonymous |
| **Flag** | `CYBLTZ2026{51d3w4yS_57473_m4ch1n3}` |

## Challenge

> The firmware image contains a per-frame CAN handler, not a player for the supplied log. Reverse its transition logic, replay the captured frames from the controller's zero-initialized state, and recover the maintenance message from the emitted labels.

Files provided:

- `controller_firmware.bin`: the firmware image
- `lift_can.log`: captured CAN traffic (`timestamp can_id_hex data`)
- `maintenance_card.txt`: a table mapping opaque label bytes to display characters, with a CRC32 of the controller table

## TL;DR

The firmware is a small x86-64 PE with a custom `.lift` section holding an XOR-encrypted state-transition table. Decrypt it, verify its CRC32 against the card, then replay every `0x321` frame through the table starting from state `0`. The emitted label bytes, mapped through the card, spell the flag.

## 1. Triage

```
$ file controller_firmware.bin
controller_firmware.bin: PE32+ executable (console) x86-64, for MS Windows, 4 sections
```

Sections: `.text` (0x1f5 bytes), `.rdata`, `.data` (zero-size, so BSS), and a non-standard **`.lift`** (0x21d bytes). `strings` shows the marker `LIFTST8`.

The `.lift` section begins with the header `4c 49 46 54 53 54 38 84 00`:

| Offset | Bytes | Meaning |
|---|---|---|
| 0 | `LIFTST8` | Magic |
| 7 | `0x0084` | Entry count (little-endian) = **132** |
| 9 | ... | Table data, `count * 4` bytes |
| 9 + count*4 | 4 bytes | CRC32 (little-endian) |

Total size: 9 + 132*4 + 4 = 541 bytes, which matches the section size exactly.

## 2. Reversing the firmware

The `.text` section contains three small functions.

### CRC32 (`0x140001000`)

A standard bitwise CRC32 using the reflected polynomial `0xEDB88320`, with init `0xFFFFFFFF` and a final NOT.

### Table loader (`0x140001050`)

1. Reads the 16-bit entry count from `.lift + 7`.
2. Rejects the table if `count * 4` exceeds the buffer size.
3. Copies `count * 4` bytes from `.lift + 9`, **XORing each with `0xA5`**.
4. Reads the stored CRC32 at `.lift + 9 + count*4` and compares it with the CRC32 of the **decrypted** table. It returns the count on success and an error code on mismatch.

### Per-frame handler (`0x140001110`)

This is the interesting part. Reconstructed:

```c
int handle_frame(uint8_t state, uint16_t can_id, uint8_t *data,
                 uint8_t *table, uint16_t count,
                 uint8_t *out_next_state, uint8_t *out_label)
{
    if (can_id != 0x321) return 0;      // only ID 0x321 is processed
    uint8_t event = data[0];            // event = first data byte

    for (int i = 0; i < count; i++) {
        uint8_t *e = &table[i * 4];
        if (e[0] == state && e[2] == event) {   // match (state, event)
            *out_next_state = e[1];
            *out_label      = e[3];
            return 1;
        }
    }
    return 0;                           // no match: state unchanged
}
```

Each table entry is 4 bytes:

```
[ current_state | next_state | event | output_label ]
```

`main` stores the new state in `.data+8` and the output label in `.data+9`. State starts at zero because `.data` is BSS.

Key behaviours to replicate:

- Only CAN ID `0x321` frames matter. IDs `0x322`, `0x301`, `0x5a0` and `0x18f` are noise.
- The **first** matching table entry wins.
- A frame with no matching entry is ignored: no state change and no output.

## 3. Decrypting and validating the table

```python
import struct, zlib

d = open('controller_firmware.bin', 'rb').read()
s = d[0x800:0x800 + 0x21d]            # .lift raw data (file offset 0x800)
n = s[7] | (s[8] << 8)                # 132
t = bytes(b ^ 0xA5 for b in s[9:9 + n*4])
crc = struct.unpack('<I', s[9 + n*4:9 + n*4 + 4])[0]
print(hex(crc), hex(zlib.crc32(t)))   # 0x71c52b35 0x71c52b35
```

The CRC32 is `71c52b35`, which matches the `LIFTST8 / CRC32=71c52b35` header in `maintenance_card.txt`. That confirms the table was decrypted correctly.

## 4. Replaying the log

```python
ents = [tuple(t[i:i+4]) for i in range(0, len(t), 4)]

# Label byte -> display character
card = {}
for line in open('maintenance_card.txt'):
    if '=' in line and not line.startswith('#') and len(line.split('=')[0]) == 2:
        k, v = line.rstrip('\n').split('=', 1)
        card[int(k, 16)] = v

state, out = 0, []
for line in open('lift_can.log').read().split('\n')[1:]:
    p = line.split()
    if len(p) < 3 or p[1] != '321':
        continue
    event = int(p[2][:2], 16)
    for cur, nxt, ev, label in ents:
        if cur == state and ev == event:
            state = nxt
            out.append(label)
            break

print(''.join(card.get(o, '?') for o in out))
```

Result: 34 transitions (17 `0x321` frames had no matching entry and were ignored), producing the labels:

```
d4 77 26 ce b0 dc 61 10 aa 0a af f2 9e 30 45 7f 3c 8c 4f a9 ef b6 85 ad 2a 02 1f 8d c0 19 9c 6c 2e ba
```

Mapped through the maintenance card:

```
CYBLTZ2026{51d3w4yS_57473_m4ch1n3}
```

## Flag

```
CYBLTZ2026{51d3w4yS_57473_m4ch1n3}
```

The leetspeak reads "sideways state machine", which fits the title.

## Takeaways

- A custom PE section with a magic string and header is often a data blob you can parse directly. You don't need to run the binary.
- The embedded CRC32 and the card's CRC32 act as a built-in check that decryption was right.
- Only the handler logic matters: which CAN ID, which byte is the event, and the first-match-wins rule. Frames for other IDs are distractors.
- The card's "opaque label" mapping hides the output, so the emitted bytes are meaningless until translated.
