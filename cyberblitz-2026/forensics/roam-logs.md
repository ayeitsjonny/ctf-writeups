# ROAM LOGS

| | |
|---|---|
| **Challenge** | ROAM LOGS |
| **Category** | Forensics |
| **Points** | 50 |
| **Author** | @unablebagel |
| **Solves** | 43 (at time of solving) |
| **Flag** | `CYBLTZ2026{LAYLAROAMS}` |

## Challenge description

> Ten images were recovered from somewhere in the Land of Dawn. At first glance they seem like ordinary pictures, but a roamer always leaves a trail behind.
>
> Look beyond what can be seen and recover the hidden logs. The contents may not make much sense at first, but remember: every route leaves a mark. Follow each route carefully and piece together what was left behind.
>
> Flag format: `CYBLTZ2026{WORD}`, uppercase with no spaces, e.g. `CYBLTZ2026{EXAMPLE}`

Provided: an archive of ten JPEG images (`img1.jpg` to `img10.jpg`).

## Summary

Each image carries a hidden log in its EXIF `UserComment` field. The log contains a string of keyboard keys. Tracing the keys on a QWERTY layout draws one letter per image. Reading the letters in log order gives `LAYLA ROAMS`.

## Step 1: Look at the images

All ten images are 1280x591 screenshots of the hero Layla in different skins from Mobile Legends. They match the "Land of Dawn" reference in the prompt, but the pictures themselves contain nothing hidden. "Look beyond what can be seen" points to the metadata.

## Step 2: Check the file structure

No trailing data follows the JPEG end-of-image marker in any file, so nothing is appended. Parsing the JPEG segments shows that the EXIF (APP1) segment length differs between images, even though every image has the same five IFD entries. The extra bytes sit after the IFD, which means a `UserComment` field was added.

`exiftool` is the quickest way to read it. Without it, a short script parses the segment directly (see `solve.py`).

## Step 3: Extract the logs

The `UserComment` of each image has the format `Log NN/10 // <KEYS> // <flavor text>`:

| Log | Keys | Flavor text |
|---|---|---|
| 01/10 | `QAZXC` | Every route leaves a mark. |
| 02/10 | `ZSEDCSD` | The order was not chosen by accident. |
| 03/10 | `RGYGB` | Follow where the movement leads. |
| 04/10 | `TGBNM` | The turns matter as much as the destination. |
| 05/10 | `CFTGBFG` | A strange path can still have a familiar shape. |
| 06/10 | `VFRTYHGFB` | Start from the beginning. Do not skip the bends. |
| 07/10 | `WSXCVFRE` | Some patterns are easier seen than read. |
| 08/10 | `BHUJMHJ` | What matters is what the route leaves behind. |
| 09/10 | `XDRFTGB` | Nine routes. One destination. |
| 10/10 | `YTRFGHNBV` | Every journey has one final turn. |

The flavor text is the hint. The key strings are not words, but each consecutive pair of keys is adjacent on a QWERTY keyboard. That makes each string a path across the keyboard, and the hints say to follow the path in order and keep the turns.

## Step 4: Trace the paths

The QWERTY rows are staggered, so the layout used for tracing is:

```
Q W E R T Y U I O P
 A S D F G H J K L
   Z X C V B N M
```

Each path draws a letter:

| Log | Keys | Path | Letter |
|---|---|---|---|
| 01 | `QAZXC` | Down the left column, then right along the bottom row | **L** |
| 02 | `ZSEDCSD` | Up to E, down to C, then a crossbar from S to D | **A** |
| 03 | `RGYGB` | A V from R through G to Y, then back to G and down to B | **Y** |
| 04 | `TGBNM` | Down the T column, then right along the bottom row | **L** |
| 05 | `CFTGBFG` | Up to T, down to B, then a crossbar from F to G | **A** |
| 06 | `VFRTYHGFB` | Stem up to R, top bar to Y, bowl down and back to F, leg to B | **R** |
| 07 | `WSXCVFRE` | Down the left side, across the bottom, up the right side, back along the top | **O** |
| 08 | `BHUJMHJ` | Up to U, down to M, then a crossbar from H to J | **A** |
| 09 | `XDRFTGB` | Up to R, down to F, up to T, down to B (two peaks) | **M** |
| 10 | `YTRFGHNBV` | Top bar leftward, middle bar rightward, bottom bar leftward | **S** |

Reading the letters in log order gives:

```
L A Y L A  R O A M S
```

"Layla roams" matches the character in the images and the challenge name.

## Notes

- Log 07 is the only ambiguous trace. The path nearly closes, so it could be read as a U with a stray key. O is the only reading that gives a word.
- The flag format requires uppercase with no spaces, so `LAYLA ROAMS` becomes `LAYLAROAMS`.

## Flag

```
CYBLTZ2026{LAYLAROAMS}
```

## Tools

- Python 3 (JPEG segment parsing, `solve.py`)
- `exiftool` (optional, reads the same field directly)
