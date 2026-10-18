# A RECORDING FROM SPACE

| | |
|---|---|
| **Category** | Forensics (audio / SSTV) |
| **Points** | 194 (dynamic; 212 when I started) |
| **Author** | @commit2main |
| **Solves** | 34 |
| **Files** | `chall_172608.flac` |

> We received a strange recording in the direction of the Sun...
>
> Could you figure out what it is?

**Flag:** `CYBLTZ2026{1c4rus_fL3w_t00_cL0s3}`

---

## TL;DR

Apollo 11 audio with a **Scottie S1 SSTV** image hidden in the left channel → the image is a **QR code** → it decodes to a **Base58 string (a Solana address)** → the address's latest transaction has a **Memo** instruction containing the flag.

```
FLAC → SSTV (Scottie S1) → QR → Base58 / Solana address → on-chain memo → flag
```

---

## 1. Recon

```
$ ffprobe chall_172608.flac
  TXXX   : NASA/JSC- Houston Audio Control Room
  GENRE  : Historical Audio
  ALBUM  : Apollo 11
  ARTIST : NASA
  Duration: 00:01:59.01, 48000 Hz, stereo, s16
```

No useful strings or appended data. The metadata is only flavour. LSBs look random, so no simple LSB stego.

Spectrograms of each channel showed:

- **Left:** a constant band of tones between ~1500 and ~2300 Hz with short 1200 Hz ticks about every 0.43 s. That is the textbook signature of **SSTV**. Pixel brightness is encoded as 1500–2300 Hz and 1200 Hz is the sync pulse.
- **Right:** just the Apollo chatter, plus a few narrow tones (see "Red herrings").

## 2. Identifying the SSTV mode

FM-demodulating the left channel (band-pass 1000–2700 Hz, Hilbert transform, instantaneous frequency) and looking at the header around t ≈ 1.4 s:

```
1200 Hz start bit
1300 1300 1100 1100 1100 1100 1300 1300   (1300 = 0, 1100 = 1, LSB first)
1200 Hz stop bit
```

Data bits `0011110` (+ even parity 0) = **VIS code 60 → Scottie S1**.

Scottie S1 timing: 320×256 pixels, 428.22 ms per line.

```
sep 1.5 ms | G 138.24 ms | sep 1.5 ms | B 138.24 ms | SYNC 9 ms | porch 1.5 ms | R 138.24 ms
```

The sync pulse sits in the middle of the line, between blue and red.

## 3. Decoding the image

No SSTV decoder was at hand, so I wrote a small one in numpy.

Steps:

1. Demodulate the left channel to an instantaneous-frequency track.
2. Detect the 9 ms 1200 Hz sync of every line with a narrow band-pass.
3. Fit the sync times with a robust linear fit. This corrects clock drift: my fit gave a period of 0.428253 s versus the nominal 0.42822 s, and without it the image shears into a diagonal.
4. Sample each pixel with the **median** frequency over the middle of its time window (robust against Apollo chatter bursts).
5. Map `1500 Hz → 0` and `2300 Hz → 255`.

```python
import numpy as np
from scipy.signal import butter, sosfiltfilt, hilbert

fs = 48000
x = left_channel  # int16 samples as float

# FM demod
sos = butter(6, [1000, 2700], btype="band", fs=fs, output="sos")
a = hilbert(sosfiltfilt(sos, x))
f = np.diff(np.unwrap(np.angle(a))) * fs / (2 * np.pi)
f = np.append(f, f[-1])

# per-line sync detection (1200 Hz, 9 ms), then linear fit -> t0, P
# (see full script below for the fit)

chan = 0.13824  # seconds per colour channel
def pixel(t_start, p):
    a = int((t_start + (p + 0.15) * chan / 320) * fs)
    b = int((t_start + (p + 0.85) * chan / 320) * fs)
    return np.median(f[a:b])

for line in range(256):
    t = t0 + line * P                  # time of this line's sync pulse
    start = {"R": t + 0.0105,          # after 9 ms sync + 1.5 ms porch
             "B": t - chan,            # blue ends where sync begins
             "G": t - 2 * chan - 0.0015}
    # img[line, p, channel] = (pixel(start[c], p) - 1500) / 800 * 255
```

Because the picture is black and white, taking the **median of R, G and B** per pixel gives a much cleaner image than any single channel.

The result is a QR code, 33×33 modules (version 4), slightly noisy where the Apollo voice bursts overlapped.

## 4. Reading the QR

I fitted the 33×33 module grid by maximising agreement with the three finder patterns and the timing patterns (row and column 6), then sampled each module by majority vote. The few corrupted row bands were fixed by QR error correction, and OpenCV decoded it:

```python
import cv2
cv2.QRCodeDetector().detectAndDecode(big_image_with_quiet_zone)
# -> '83ThRwrmBavUgykzMdqfdWu2aisXRqaNzcynYKw7W3AQ'
```

## 5. What is that string?

```
83ThRwrmBavUgykzMdqfdWu2aisXRqaNzcynYKw7W3AQ
```

- 44 characters, no `0`, `O`, `I`, `l` → **Base58**.
- Decodes to exactly **32 bytes**: `68a3281568f57c64e903ddfdbfc1c41afebc548392e4ce362e9bab1a4b376811`. The bytes are random (not text, not zlib, not a single-byte XOR).
- 32-byte Base58 is the format of a **Solana public key**.

Searching it on Solscan / Solana Explorer shows a live **mainnet** account with two incoming transfers from `2Q9KFUMQ…fS98eZPi`:

| Date | Signature | Instructions |
|---|---|---|
| Sep 15, 2026 | `5ZGiw…p2Dsh` | System Program: Transfer |
| Oct 13, 2026 | `obLz9…BVfkt` | System Program: Transfer **+ Memo Program: Memo** |

## 6. The memo

Opening `obLz9…BVfkt` on Solana Explorer, the instruction list is:

1. Compute Budget: Set Compute Unit Price
2. System Program: Transfer (0.00001 SOL)
3. **Memo Program: Memo**, data (UTF-8): `CYBLTZ2026{1c4rus_fL3w_t00_cL0s3}`
4. Compute Budget: Set Compute Unit Limit

The memo is the flag: *"icarus flew too close"*, which fits a recording sent toward the Sun.

---

## Red herrings (all checked)

| Observation | What it is |
|---|---|
| FSK burst right after the image (1900/2100 Hz, 22 ms per bit) | The SSTV software's **FSK ID**: header `0x20 0x2A`, then `NOCALL`, then `0x01`. The default callsign, so no hint. |
| Narrow right-channel tones near 2467 / 2515 Hz | Quindar-style tones from the original Apollo comm loop |
| Thin line at ~15.73 kHz and 60 Hz hum | NTSC horizontal-line frequency and mains hum from the original recording chain |
| Odd tones in the first ~0.8 s | Leftover Apollo audio and tone bursts, with no data pattern |

## Takeaways

- Audio challenges: always look at the **spectrogram of each channel separately**, and zoom into the 0–5 kHz band. A steady 1500–2300 Hz band with 1200 Hz ticks means SSTV.
- The VIS header tells you the mode. Decode it by hand if no tool is around (1100 Hz = 1, 1300 Hz = 0, LSB first).
- Clock drift shears SSTV images. Fit the sync positions instead of trusting the nominal line time.
- A 32-byte Base58 string is very likely a Solana address, so check the explorer and **read the Memo instruction** of each transaction.

## Tools

`ffmpeg` / `ffprobe`, Python (`numpy`, `scipy`, `opencv-python`, `Pillow`), Solana Explorer / Solscan.
