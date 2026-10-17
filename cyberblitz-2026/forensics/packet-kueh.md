# Packet Kueh

| | |
|---|---|
| **Category** | Forensics |
| **Points** | 453 |
| **Solves** | 14 |
| **Author** | Anonymous |
| **File** | `packet_kueh.pcap` |

## Challenge

> Seven wrappers crossed the wire, but not every one belongs to the courier's order. Recover the message inside the authentic wrappers.
>
> The courier's assembly card reads: start with record ID 0 and follow each record's next field until 255 (END).

## TL;DR

The pcap contains custom `PKUEH|id|next|data|checksum` records over UDP. Every ID has a forged twin. The authentic one is the record whose last field is the CRC32 of the rest of the record. Walk the chain `0 → 4 → 1 → 6 → 2 → 5 → 3 → 255`, join the data fields, and base64-decode.

**Flag:** `CYBLTZ2026{l4y3r5_b3f0r3_f1ll1nG}`

## Step 1: Look at the capture

The file is small (15 packets, all UDP over IPv4). There is no tshark or scapy needed, so I parsed it with plain Python (`struct`). The packets fall into three groups:

| Port | Payload | Meaning |
|---|---|---|
| `6262` | `PKUEH\|id\|next\|data\|checksum` | The "wrappers" (14 packets) |
| `5353` | `delivery-status=warming` | Repeated noise |
| `6262` (ID 20) | `PKUEH\|20\|255\|dHJhaW5pbmctb3JkZXI=\|...` | Decoy chain |

The ID 20 record decodes from base64 to `training-order`, which is a decoy.

```python
import struct

d = open('packet_kueh.pcap', 'rb').read()
off = 24  # skip pcap global header
while off < len(d):
    ts, us, cl, ol = struct.unpack('<IIII', d[off:off + 16])
    off += 16
    pkt = d[off:off + cl]
    off += cl
    ip = pkt[14:]
    ihl = (ip[0] & 15) * 4
    udp = ip[ihl:]
    payload = udp[8:]
    print(payload)
```

## Step 2: Parse the records

Each wrapper has the form:

```
PKUEH|<id>|<next>|<data>|<checksum>
```

Records for IDs 0 to 6 each appear **twice** with different data, so one copy of each is forged:

| ID | next | data (copy A) | data (copy B) |
|---|---|---|---|
| 0 | 4 | `Q1lCTFR` | `Y29sZC1` |
| 1 | 6 | `tsNHkz` | `dyb25n` |
| 2 | 5 | `FtZXIt` | `NmMHIz` |
| 3 | 255 | `wxbkd9` | `1rdWVo` |
| 4 | 1 | `rdWVoLX` | `aMjAyNn` |
| 5 | 3 | `Y29sZC` | `X2YxbG` |
| 6 | 2 | `cjVfYj` | `LXN0ZW` |

The `next` fields are identical across both copies, so following the chain alone cannot separate real from fake. The checksum has to.

## Step 3: Work out the checksum

The 8-hex-character checksum looked like a CRC32. Brute-forcing a few likely inputs showed that it is the CRC32 of everything before the last `|`:

```python
import zlib

rec = "PKUEH|0|4|Q1lCTFR"
print("%08x" % zlib.crc32(rec.encode()))  # d2bea34d
```

`d2bea34d` matches the checksum on the first copy of ID 0, and the other copy fails. Applying this check to all records gives exactly one valid copy per ID:

| ID | Authentic data |
|---|---|
| 0 | `Q1lCTFR` |
| 4 | `aMjAyNn` |
| 1 | `tsNHkz` |
| 6 | `cjVfYj` |
| 2 | `NmMHIz` |
| 5 | `X2YxbG` |
| 3 | `wxbkd9` |

The ID 20 record also has a valid CRC, but it is not reachable from ID 0, so the assembly card never visits it.

## Step 4: Follow the assembly card

Starting at ID 0 and following `next` until 255:

```
0 -> 4 -> 1 -> 6 -> 2 -> 5 -> 3 -> 255 (END)
```

Concatenating the authentic data fields in that order:

```
Q1lCTFR aMjAyNn tsNHkz cjVfYj NmMHIz X2YxbG wxbkd9
```

## Step 5: Decode

```python
import base64

s = 'Q1lCTFR' + 'aMjAyNn' + 'tsNHkz' + 'cjVfYj' + 'NmMHIz' + 'X2YxbG' + 'wxbkd9'
print(base64.b64decode(s))
# b'CYBLTZ2026{l4y3r5_b3f0r3_f1ll1nG}'
```

## Full solve script

```python
import struct
import zlib
import base64

d = open('packet_kueh.pcap', 'rb').read()
off = 24
records = []  # (id, next, data)

while off < len(d):
    _, _, cl, _ = struct.unpack('<IIII', d[off:off + 16])
    off += 16
    pkt = d[off:off + cl]
    off += cl

    ip = pkt[14:]
    ihl = (ip[0] & 15) * 4
    payload = ip[ihl + 8:]  # UDP payload

    if not payload.startswith(b'PKUEH|'):
        continue

    body, _, checksum = payload.decode().rpartition('|')
    if '%08x' % zlib.crc32(body.encode()) != checksum:
        continue  # forged wrapper

    _, rid, nxt, data = body.split('|')
    records.append((int(rid), int(nxt), data))

by_id = {rid: (nxt, data) for rid, nxt, data in records}

out = ''
cur = 0
while cur != 255:
    nxt, data = by_id[cur]
    out += data
    cur = nxt

print(base64.b64decode(out).decode())
```

## Takeaways

- Custom protocols in a pcap are often self-describing. Look for a magic prefix (`PKUEH`) and field separators.
- A checksum that cannot be verified from context alone is usually a plain CRC32, so try it over the record minus the checksum field.
- Ignore packets that do not belong to the chain (port 5353 noise and the ID 20 decoy).

**Flag:** `CYBLTZ2026{l4y3r5_b3f0r3_f1ll1nG}`
