# SOUNDCHECK AT 1:59

| | |
|---|---|
| **Category** | Forensics |
| **Points** | 466 |
| **Solves** | 12 |
| **Author** | Anonymous |
| **File** | `soundcheck.wav` |
| **Flag** | `CYBLTZ2026{cu3_7h3_r34L_ch0ru5}` |

## Challenge description

> The 1:59 recording sounds like a test tone, but its cue display shows the chorus before the intro. Recover the message hidden in the WAV.

## TL;DR

The audio is a decoy. The flag is split into base64 fragments stored as **cue point labels** in the WAV's RIFF metadata. Sort the cues by their **sample position** (playback order), concatenate the labels, and base64-decode the result.

## Solution

### 1. Inspect the file

```
$ file soundcheck.wav
soundcheck.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 8 bit, mono 8000 Hz
```

8 kHz, 8-bit mono, ~952 KB, so about 119 seconds, which matches the "1:59" in the title. Listening to it (or checking a spectrogram) gives nothing useful, so the next step is to walk the RIFF chunks instead of looking at the sample data.

### 2. Walk the RIFF chunks

A WAV file is a list of chunks (`fmt `, `data`, ...). This one has extra metadata chunks around the audio:

| Chunk | Contents |
|---|---|
| `LIST` / `adtl` | 10 `labl` sub-chunks, each a cue ID plus a short text label |
| `cue ` | 10 cue points: cue ID plus a sample offset into the `data` chunk |
| `data` | the 8 kHz test tone |
| `LIST` / `INFO` | `INAM` = "1:59 scratch take", `ICMT` = "Rehearsal ledger order: 7,11,19,29,41,53,61,71,89,97" |

The labels look like base64 fragments:

```
FRaMj   N2gz   cnU1   NExf   Y2gw
Q1lCT   fQ==   jdTNf  AyNnt  X3Iz
```

### 3. The decoy: the "ledger order"

The `ICMT` comment gives an order for the cue IDs: `7,11,19,29,41,53,61,71,89,97`. That is just the IDs sorted numerically. Joining the labels in that order gives:

```
Y2gwAyNntfQ==N2gzNExfQ1lCTcnU1jdTNfX3IzFRaMj
```

This is not valid base64 (there is `==` in the middle), and it decodes to garbage. The challenge text hints at this: the cue *display* shows the "chorus before the intro", meaning the listed order is not the playback order.

### 4. The real order: sample position

Each cue point in the `cue ` chunk carries a sample offset, which is where it sits on the timeline. Sorting by that offset instead of by ID:

| Sample offset | Cue ID | Label |
|---:|---:|---|
| 40000 | 53 | `Q1lCT` |
| 128000 | 97 | `FRaMj` |
| 216000 | 11 | `AyNnt` |
| 304000 | 71 | `jdTNf` |
| 392000 | 29 | `N2gz` |
| 480000 | 89 | `X3Iz` |
| 568000 | 41 | `NExf` |
| 656000 | 7 | `Y2gw` |
| 744000 | 61 | `cnU1` |
| 832000 | 19 | `fQ==` |

Concatenated:

```
Q1lCTFRaMjAyNntjdTNfN2gzX3IzNExfY2gwcnU1fQ==
```

This is valid base64 with correct padding at the end.

### 5. Decode

```
$ echo 'Q1lCTFRaMjAyNntjdTNfN2gzX3IzNExfY2gwcnU1fQ==' | base64 -d
CYBLTZ2026{cu3_7h3_r34L_ch0ru5}
```

The flag reads "cue the real chorus" in leetspeak.

## Solver script

`solve.py` parses the chunks by hand, so no third-party libraries are needed:

```python
#!/usr/bin/env python3
"""Recover the flag from soundcheck.wav by reassembling cue-point labels."""
import base64
import struct
import sys

path = sys.argv[1] if len(sys.argv) > 1 else "soundcheck.wav"
data = open(path, "rb").read()

labels = {}   # cue ID -> label text
cues = []     # (cue ID, sample offset)

pos = 12  # skip "RIFF <size> WAVE"
while pos + 8 <= len(data):
    cid = data[pos:pos + 4]
    size = struct.unpack("<I", data[pos + 4:pos + 8])[0]
    body = data[pos + 8:pos + 8 + size]

    if cid == b"cue ":
        count = struct.unpack("<I", body[:4])[0]
        for i in range(count):
            entry = body[4 + 24 * i:4 + 24 * (i + 1)]
            cue_id = struct.unpack("<I", entry[:4])[0]
            sample_offset = struct.unpack("<I", entry[20:24])[0]
            cues.append((cue_id, sample_offset))

    elif cid == b"LIST" and body[:4] == b"adtl":
        p = 4
        while p + 8 <= len(body):
            sub = body[p:p + 4]
            sub_size = struct.unpack("<I", body[p + 4:p + 8])[0]
            sub_body = body[p + 8:p + 8 + sub_size]
            if sub == b"labl":
                lid = struct.unpack("<I", sub_body[:4])[0]
                labels[lid] = sub_body[4:].rstrip(b"\0").decode()
            p += 8 + sub_size + (sub_size & 1)

    pos += 8 + size + (size & 1)

# Order labels by playback position, NOT by cue ID / chunk order
ordered = [labels[cid] for cid, off in sorted(cues, key=lambda c: c[1])]
b64 = "".join(ordered)
print("base64:", b64)
print("flag:  ", base64.b64decode(b64 + "=" * (-len(b64) % 4)).decode())
```

Run it:

```
$ python3 solve.py soundcheck.wav
base64: Q1lCTFRaMjAyNntjdTNfN2gzX3IzNExfY2gwcnU1fQ==
flag:   CYBLTZ2026{cu3_7h3_r34L_ch0ru5}
```

## Key takeaways

- WAV files are RIFF containers, and anything outside the `data` chunk (`cue `, `LIST/adtl`, `LIST/INFO`) is invisible when you just play the audio.
- Cue labels and cue positions are stored in separate chunks and joined by cue ID, so the order you read them in is not guaranteed to be the order they play.
- Check for decoys. A "helpful" metadata comment (the ledger order) can send you down the wrong path, and a base64 string with `==` in the middle is a sign the order is wrong.
- Tools that would show this quickly: `exiftool`, `ffprobe`, Audacity (label track), or any RIFF chunk parser.
