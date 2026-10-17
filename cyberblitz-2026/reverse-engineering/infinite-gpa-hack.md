# Infinite GPA Hack

| | |
|---|---|
| **CTF** | CYBLTZ2026 |
| **Challenge** | Infinite GPA Hack |
| **Author** | @canairo |
| **Points** | 487 |
| **Solves** | 8 |
| **Category** | Reverse Engineering / Obfuscation |
| **Files** | `hack.py` |

> WE MAKING IT OUT THE PROVOST'S LIST WITH THIS ONE 🔥🔥🔥🔥

## TL;DR

`hack.py` looks like a harmless, flashy "GPA hacker" CLI. A line of whitespace padding hides an `exec(base64.b64decode(...))` call off-screen. Decoding the base64 (without running it) reveals the flag hardcoded in a comparison against `flag.txt`.

**Flag:** `CYBLTZ2026{inf1n1t3_gp4_gl1tch}`

## Overview

The script prints a banner, asks the player to pick a module (`INF1001` to `INF1005`), runs a fake loading bar with silly "thinking" words, then prints `SUCCESSFULLY HACKED!`.

Nothing in the visible logic touches a flag:

- `banner()` only prints the module list.
- `loading_bar()` only animates a progress bar.
- `main()` looks up the module and prints a success message.

Since the script does nothing useful on its own, the flag has to be hidden somewhere else in the file.

## Finding the hidden code

The suspicious line is this one in `main()`:

```python
loading_bar();                                                                                                    exec(base64.b64decode(b"dHJ5OgogICAg...").decode())
```

(Whitespace shortened here.) After `loading_bar();` there is a very long run of spaces, which pushes the `exec(...)` call far off the right edge of most editors. Unless you scroll horizontally or enable word wrap, it is invisible.

Things that give it away:

- `base64` is imported but never used anywhere else in the file.
- A trailing `;` after `loading_bar()` is unusual in Python.
- `exec` on a decoded blob is a classic obfuscation pattern.

## Decoding the payload

Rather than running the script, decode the blob statically:

```bash
echo "dHJ5OgogICAgaWYgb3BlbigiLi9mbGFnLnR4dCIpLnJlYWQoKSA9PSAiQ1lCTFRaMjAyNntpbmYxbjF0M19ncDRfZ2wxdGNofSI6IHByaW50KCJUaGF0J3MgdGhlIGNvcnJlY3QgZmxhZyEiKTsgc3lzLmV4aXQoMCkKZXhjZXB0OgogICAgcGFzcwo=" | base64 -d
```

Output:

```python
try:
    if open("./flag.txt").read() == "CYBLTZ2026{inf1n1t3_gp4_gl1tch}":
        print("That's the correct flag!")
        sys.exit(0)
except:
    pass
```

The payload compares the contents of `./flag.txt` to a hardcoded string. That string is the flag.

## Solution

1. Open `hack.py` with word wrap on, or search for `exec(`.
2. Extract the base64 string passed to `b64decode`.
3. Decode it with `base64 -d` (or CyberChef's *From Base64*).
4. Read the flag from the string comparison.

## Takeaways

- Always check for **whitespace-hidden code** on a single line. Word wrap or `grep -n "exec\|eval"` finds it quickly.
- **Never run unknown `exec`/`eval` payloads.** Decode them statically first.
- Unused imports (like `base64` here) are a good hint that something is hiding.

## Flag

```
CYBLTZ2026{inf1n1t3_gp4_gl1tch}
```
