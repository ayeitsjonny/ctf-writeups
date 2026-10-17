# SCYTALE

| | |
|---|---|
| **CTF** | CyberBlitz 2026 |
| **Challenge** | SCYTALE |
| **Category** | Crypto |
| **Points** | 50 |
| **Author** | @unablebagel |
| **Solves** | 48 (at time of solving) |
| **Flag** | `CYBLTZ2026{Scytale_R0ds!Unwrap#42}` |

## Challenge description

> An intercepted courier has vanished, leaving three unreadable message strips behind. The rods used to decipher them are hidden somewhere across two rooms. Search the scene, recover the missing pieces, and use the workbench to uncover the courier's secret before it disappears for good.

The challenge is a small browser game. You explore two rooms, collect three **rods** (A, B, C) and three **strips** (1, 2, 3), then combine them at a **workbench**. The final answer is submitted in the format `Location / Time / Password`.

## TL;DR

The workbench simulates a **scytale**, a classical transposition cipher. Each strip only decodes correctly on one specific rod. Trying all pairings reveals three plaintext messages, which give the location, time and password. Submitting them unlocks the flag.

## Background: how a scytale works

A scytale is a transposition cipher. The sender wraps a strip of parchment around a rod, writes the message along the length of the rod, then unwraps the strip. The letters are all still there, just rearranged. To read it, the receiver needs a rod of the **same diameter**.

- **Key:** the rod's diameter, which sets how many letters fit in each row around the rod.
- **Type:** transposition, not substitution. The letters themselves are unchanged.
- **Padding:** extra letters (here `X`) fill the last row so the message fits.

## Solution

### 1. Collect the pieces

Searching both rooms gives three rods (A, B, C) and three strips (1, 2, 3).

### 2. Try the pairings on the workbench

Drag one strip and one rod onto the bench, then rotate the wrapped rod to read every row. A wrong pairing gives gibberish. A correct pairing reads as English.

Each rod produces a different row length:

| Rod | Letters per row |
|-----|-----------------|
| A | 12 |
| B | 6 |
| C | 16 |

### 3. Decode each strip

| Strip | Rod | Rows (read in order) | Plaintext |
|-------|-----|----------------------|-----------|
| 3 | A | `THESECRETPASSWO` / `RDISSILVERFALCO` / `NSAYITTOTHECOUR` / `IERATTHEMEETING` | The secret password is silver falcon. Say it to the courier at the meeting. |
| 1 | B | `THEMEE` / `TINGPL` / `ACEIST` / `HEOLDC` / `LOCKTO` / `WERXXX` | The meeting place is the Old Clock Tower. |
| 2 | C | `ARRIVEATNINEPMAN` / `DWAITFORTHECOURI` / `ERAFTERDARKXXXXX` | Arrive at nine PM and wait for the courier after dark. |

Trailing `X` characters are padding and are left out of the answer.

### 4. Assemble the final answer

| Clue | Answer |
|------|--------|
| Location | OLD CLOCK TOWER |
| Time | 9 PM |
| Password | SILVER FALCON |

Final answer entered in the box:

```
OLD CLOCK TOWER / 9 PM / SILVER FALCON
```

This returned the flag:

```
CYBLTZ2026{Scytale_R0ds!Unwrap#42}
```

## Lessons learned

- **The key space is tiny.** A scytale only has as many keys as there are possible rod diameters. With 3 rods, there are just 9 strip/rod pairings to test.
- **Transposition keeps letter frequencies.** The ciphertext has the same letter distribution as normal English, so a correct decryption is obvious on sight, and an automated scorer (counting common English words) can pick the right key.
- **A wrong rod still shows structure.** Wrong pairings gave gibberish, but the letters were still the real plaintext letters, only scrambled.

## Possible automation

The manual process can be scripted: for a ciphertext of length `n`, try every row length that divides `n`, rebuild the message by reading the grid in the right order, and score each result by how many common English words it contains. The highest score is the likely plaintext.

## Notes

- Instance URLs are omitted since challenge servers usually go offline after the event.
