# Singlish as a Service

| | |
|---|---|
| **Event** | CYBLTZ 2026 |
| **Category** | Crypto / Encoding |
| **Author** | Anonymous |
| **Points** | 466 (dynamic scoring, started at 487) |
| **Solves** | 12 at time of solving |
| **Flag format** | `CYBLTZ2026{...}` |

## Challenge description

> A station kiosk turned familiar phrases into tiny tokens. The morning and evening cards survived; recover the missing midnight announcement.

Two files were provided:

- `phrase_...` : the surviving cards, each with a plaintext line and its token encoding
- `midnight.sas` : the encoded midnight announcement

## TL;DR

The kiosk uses a simple phrase-to-token substitution. The surviving cards act as known plaintext, which is enough to rebuild the token table. Decoding `midnight.sas` with that table gives the flag directly.

```
CYBLTZ2026{d0n7_pL4y_pL4y}
```

## Analysis

### 1. Inspecting the files

Both files are plain text, so no extraction or unpacking was needed.

The phrase file contains five cards:

```
CARD 0 (RETIRED) PLAIN: play
CARD 0 (RETIRED) CODE:  P7
CARD 1 PLAIN: d0n7|pL4y
CARD 1 CODE:  K2|P7
CARD 2 PLAIN: CYBLTZ2026{|pL4y|}
CARD 2 CODE:  L4|P7|R9
CARD 3 PLAIN: d0n7|_|pL4y
CARD 3 CODE:  K2|SP|P7
CARD 4 (RETIRED) PLAIN: dont|play
CARD 4 (RETIRED) CODE:  K2|P7
```

The midnight file contains a single line:

```
L4|K2|SP|P7|SP|P7|R9
```

In both files, `|` is only a separator between tokens. It is not part of the decoded output.

### 2. Spotting the decoys

Cards 0 and 4 are marked `RETIRED`, and they contradict the active cards:

- Card 0 says `P7` decodes to `play`, but cards 1 to 3 say `P7` decodes to `pL4y`.
- Card 4 says `K2|P7` decodes to `dont|play`, but card 1 says the same code decodes to `d0n7|pL4y`.

Since the same code cannot map to two different plaintexts, the retired cards are stale entries meant to mislead. Only cards 1 to 3 are used.

### 3. Rebuilding the token table

Matching each active card's plaintext segments against its code segments gives:

| Token | Plaintext |
|-------|-----------|
| `K2`  | `d0n7` |
| `P7`  | `pL4y` |
| `L4`  | `CYBLTZ2026{` |
| `R9`  | `}` |
| `SP`  | `_` |

- Card 1 (`K2|P7` = `d0n7|pL4y`) fixes `K2` and `P7`.
- Card 2 (`L4|P7|R9` = `CYBLTZ2026{|pL4y|}`) fixes `L4` and `R9`, and confirms `P7`.
- Card 3 (`K2|SP|P7` = `d0n7|_|pL4y`) fixes `SP` and confirms `K2` and `P7`.

Every mapping is supported by at least one active card, and the overlaps agree with each other.

### 4. Decoding the midnight announcement

Every token in `midnight.sas` already appears in the table, so no guessing was needed:

```
L4          K2     SP   P7     SP   P7     R9
CYBLTZ2026{ d0n7   _    pL4y   _    pL4y   }
```

Joined together:

```
CYBLTZ2026{d0n7_pL4y_pL4y}
```

## Solve script

The same decode in a few lines of Python:

```python
table = {
    "K2": "d0n7",
    "P7": "pL4y",
    "L4": "CYBLTZ2026{",
    "R9": "}",
    "SP": "_",
}

midnight = "L4|K2|SP|P7|SP|P7|R9"
print("".join(table[t] for t in midnight.split("|")))
```

Output:

```
CYBLTZ2026{d0n7_pL4y_pL4y}
```

## Flag

```
CYBLTZ2026{d0n7_pL4y_pL4y}
```

## Takeaways

- When samples contradict each other, check whether the odd ones out are labeled as outdated or invalid (here, `RETIRED`) and build the model from the consistent set.
- Known plaintext/ciphertext pairs are often enough to reconstruct a substitution scheme without breaking anything.
- Check that every token in the target appears in the recovered table before assuming anything is "missing".
