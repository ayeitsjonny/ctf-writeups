# Ghost Rider

| | |
|---|---|
| **Challenge** | GHOST RIDER |
| **Category** | OSINT / Forensics |
| **Points** | 50 |
| **Author** | @unablebagel |
| **Solves** | 64 |
| **Files** | `car.jpg` |

# GHOST RIDER: Writeup

**Event:** CYBLTZ2026
**Flag:** `CYBLTZ2026{RD3262G}`

## Challenge description

> I caught this driverless shuttle waiting at a pick-up point, but someone got a little too happy with the marker and wiped its licence plate off.
>
> I need your help to recover its carplate number.
>
> Flag format: `CYBLTZ2026{PLATENUMBER}`, uppercase with no spaces, e.g. `CYBLTZ2026{SAB1234A}`

We are given a single image, `car.jpg` (720x1280): a purple shuttle photographed from behind, branded **"Autonomous Shuttle"** and **"Powered by Grab x WeRide"**, with a hotline number on the bumper. The rear plate is covered by an opaque green scribble.

## Step 1: Is the scribble reversible?

The first idea was that the edit might have left the original data behind, so I checked the file metadata and structure.

```
$ exiftool car.jpg
...
File Type                       : JPEG
JFIF Version                    : 1.01
Image Width                     : 720
Image Height                    : 1280
Encoding Process                : Baseline DCT, Huffman coding
...

$ binwalk car.jpg
DECIMAL       HEXADECIMAL     DESCRIPTION
0             0x0             JPEG image data, JFIF standard 1.01

$ strings car.jpg
(only compressed-data noise, no readable markers)
```

Nothing there:

- No EXIF block, so no embedded thumbnail of the unedited image.
- No appended or embedded files according to `binwalk`.
- No comments or readable strings.

The image was re-encoded after the edit, so the scribble is baked into the pixels and the file itself won't give us the plate.

## Step 2: Look at the plate pixels

I cropped and enlarged the plate region. The scribble is fully opaque, so the middle of the plate can't be recovered. However, a character at the **bottom-right corner** peeks out from under it and looks like a **G**. In Singapore, the last character of a plate is a checksum letter, so this gives us a useful constraint.

## Step 3: Identify the vehicle and the plate format

The branding identifies the shuttle as a **Grab x WeRide "Ai.R" GXR** van, which operates in Punggol, Singapore. The fleet is small, so the plate can likely be found from other sources.

A press photo of the same kind of shuttle (AsiaOne, "3 fixed-route autonomous shuttle services to be progressively deployed in Punggol from 2026") shows a legible front plate reading **RD3256A**. That gives the plate format for this fleet:

```
RD + 4 digits + checksum letter
```

## Step 4: Verify the format with the Singapore checksum

Singapore plates end in a checksum letter calculated as follows:

1. Take the last two letters of the prefix and convert them to numbers (A=1 ... Z=26).
2. Pad the digits to 4 digits.
3. Multiply the six numbers by the weights `9, 4, 5, 4, 3, 2` and sum them.
4. Take the sum modulo 19 and look it up in `AZYXUTSRPMLKJHGEDCB`.

```python
W = [9, 4, 5, 4, 3, 2]
C = "AZYXUTSRPMLKJHGEDCB"

def checksum(prefix, digits):
    nums = [ord(c) - 64 for c in prefix[-2:]]
    nums += [int(d) for d in f"{digits:04d}"]
    return C[sum(a * b for a, b in zip(nums, W)) % 19]

print(checksum("RD", 3256))   # A  -> RD3256A is valid
print(checksum("RD", 3262))   # G  -> RD3262G is valid
```

`RD3256` gives `A`, which matches the plate in the press photo, so the method and the format are confirmed.

## Step 5: Narrow the candidates using the fleet list

The fleet appears to have been registered as one batch. Land Transport Guru's page on the Punggol autonomous shuttle services lists the ten 5-seater GXR units:

```
RD3255C  RD3256A  RD3257Y  RD3258U  RD3259S
RD3260L  RD3261J  RD3262G  RD3264B  RD3265Z
```

Only **one** of these ends in **G**: **RD3262G**. That matches the visible character in the corner of the scribbled plate, and the checksum for `RD3262` is indeed `G`.

## Caveats

I did not match this specific van to a photograph of RD3262G. The identification rests on the visible last character plus the fleet list, so it depends on that character really being a G. If it had been misread, the other fleet plates would be the fallback candidates (a G can resemble a 6 or a C at this resolution).

## Flag

```
CYBLTZ2026{RD3262G}
```

## Takeaways

- Check the file for hidden data first (`exiftool`, `binwalk`, `strings`), but be ready to move on when it comes up empty.
- Partial information, even a single visible character, can be enough when combined with a **checksum** and a small candidate set.
- The checksum scheme for the plate format turned a fuzzy visual guess into a verifiable answer.

## Tools used

- `exiftool`, `binwalk`, `strings`
- Python (Pillow for cropping and enlarging; a short checksum script)
- Web search (OSINT) for the fleet's plate list
