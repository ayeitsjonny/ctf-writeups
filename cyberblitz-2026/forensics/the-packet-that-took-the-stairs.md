# THE PACKET THAT TOOK THE STAIRS

| | |
|---|---|
| **Challenge** | THE PACKET THAT TOOK THE STAIRS |
| **Category** | Forensics |
| **Points** | 115 |
| **Author** | Anonymous |
| **Solves** | 38 (at time of solving) |
| **Flag** | `CYBLTZ2026{fr4gm3n75_74k3_7h3_5741rS}` |

## Challenge

> A maintenance command crossed a low-MTU tunnel in pieces. Use the corrected incident time from 94 Seconds Earlier to recover its receipt.

The title and description point at IP fragmentation. A command was split across many small packets, and the receipt has to be rebuilt from them.

## Summary of the solve

1. List the packets and find that most are 8-byte IP fragments.
2. Reassemble the fragments per source host. Every fragment shares IP ID 38058, so grouping by ID alone mixes senders.
3. Read the 13 `STAIR1|...` messages that come out.
4. Verify each message with its `crc32` field. Most receipts fail.
5. Use the 94-second duration and the desk start time to pick the right case.

## Step 1: Look at the capture

The capture has 313 packets, all UDP to `10.26.3.80`. Three kinds of traffic appear:

| Traffic | Source | Notes |
|---|---|---|
| Fragmented datagrams | `10.26.3.21` to `.34` | 8 bytes of payload per fragment, 21 or 22 fragments each |
| `tunnel-health` probes | `10.26.3.1:5000` to `:5001` | Plain text, `state=normal`. Noise. |
| One unfragmented datagram | `10.26.3.47:44147` | Single 208-byte packet with a full message |

Packet 18 shows the first fragment header with the UDP header embedded. Only the first fragment carries the UDP header, so the first 8 bytes of the reassembled payload must be dropped.

```
src port 44191, dst port 44101, length 0xac, checksum 0
```

## Step 2: Reassemble per source

All fragments use IP ID `38058`. The usual reassembly key of (src, dst, id, proto) still separates senders because the source address differs. A handler that keys on ID only would merge 13 different messages into one.

Each source sends fragments out of order. Sorting by fragment offset restores the data.

## Step 3: Read the messages

Each reassembled payload has this form:

```
STAIR1|case_id=7319|desk_start_ms=1782902525000|duration_ms=94000|zone=lobby-east|command=BLACKOUT_TEST|receipt=CYBLTZ2026{...}|crc32=4d77a334
```

There are 13 of these, spread over 5 case IDs, and several hosts send near-identical copies of the same case with small changes in the receipt. For example, case 7319 appears from six sources with six different receipt spellings.

## Step 4: Verify with crc32

The `crc32` field is computed over everything before `|crc32=`. Checking it removes most candidates.

```python
body, crc = text.rsplit("|crc32=", 1)
ok = format(zlib.crc32(body.encode()) & 0xffffffff, "08x") == crc
```

Case 7319 sends six different receipts claiming the same crc `4d77a334`. Only one matches it:

| Source | Receipt | crc matches |
|---|---|---|
| `10.26.3.22` | `frAgm3n75_74k3_7h3_5T41rS` | no |
| `10.26.3.24` | `fr4gm3nT5_7Ak3_7h3_5741rS` | no |
| `10.26.3.26` | `fr4gm3n7S_74k3_7h3_5741r5` | no |
| `10.26.3.27` | `fr4gm3n75_74k3_7h3_5741rS` | **yes** |
| `10.26.3.29` | `fr4gm3n75_74kE_Th3_5741rS` | no |
| `10.26.3.32` | `fr4gm3n75_74k3_7h3_ST41rS` | no |

## Step 5: Pick the case

Five cases have a valid crc:

| Case | Zone | Duration | Desk start (ms) | Note |
|---|---|---|---|---|
| 7421 | lobby-west | 94000 | 1782902525000 | Wrong zone |
| 7319 | lobby-east | 94000 | 1782902525000 | Fragmented `BLACKOUT_TEST` |
| 7350 | lobby-east | 87000 | 1782902525000 | Duration is not 94 s |
| 7991 | lobby-east | 94000 | 1782816125000 | One day earlier |
| 7104 | atrium | 94000 | 1782902400123 | Different zone and start |

The challenge asks for a maintenance command that crossed the tunnel in pieces, with a 94-second correction. Case 7319 is the fragmented message that matches `duration_ms=94000` at the main burst time of `1782902525000`.

The unfragmented packet from `10.26.3.47` also has case 7319 and a valid crc. Its command is `READINESS_CHECK`, and it arrived in one piece, so it does not fit the description. Its receipt is a decoy.

## Solve script

```python
from scapy.all import rdpcap, IP
import zlib, collections

pkts = rdpcap("stairs.pcap")
frags = collections.defaultdict(dict)

for p in pkts:
    ip = p[IP]
    if ip.frag or ip.flags.MF:
        frags[ip.src][ip.frag] = bytes(ip.payload)

for src, f in frags.items():
    data = b"".join(f[k] for k in sorted(f))[8:]   # drop UDP header
    text = data.decode()
    body, crc = text.rsplit("|crc32=", 1)
    ok = format(zlib.crc32(body.encode()) & 0xffffffff, "08x") == crc
    if ok:
        print(src, text)
```

## Flag

```
CYBLTZ2026{fr4gm3n75_74k3_7h3_5741rS}
```

## Notes and lessons

- IP fragments from different hosts can share an IP ID. Always group by source as well.
- A checksum field in a challenge is a built-in oracle. Use it to reject decoys instead of comparing spellings by eye.
- The fragment offset, not arrival order, decides the byte order.
- Only the first fragment carries the UDP header.
