# Unreliable Narrator

**Event:** CyberBlitz CTF
**Category:** _(fill in: Misc / Forensics)_
**Points:** 73
**Author:** @unablebagel
**Flag:** `CYBLTZ2026{ERRATA_IN_THE_MARGIN}`

> Got drunk at the ePUB and now the pages are blurry, the plot makes no sense, and the narrator is telling me things that never happened. I'm starting to think the copy I was handed isn't the only printing.

---

## TL;DR

The EPUB contains **two packages** (a standard edition and a hidden "collector's edition"). The collector's edition has an extra page holding a scrambled flag. A **custom font with a permuted `cmap`** is meant to render it readable, but the CSS never applies the font, so you have to decode it yourself.

---

## 1. Reading the description

Each phrase in the blurb is a clue:

| Phrase | What it pointed to |
|---|---|
| "the pages are blurry" | something wrong with how text is rendered, so check the fonts |
| "the plot makes no sense" / "tells me things that never happened" | the text shown is not the text stored |
| "the copy I was handed isn't the only printing" | a second edition hidden in the file |

## 2. EPUB is just a zip

```bash
cp unreliable-narrator.epub unreliable-narrator.zip
unzip unreliable-narrator.zip -d x
```

Layout:

```
META-INF/container.xml
EPUB/main.opf
EPUB/alternate.opf        <-- not used by most readers
EPUB/nav.xhtml
EPUB/css/book.css
EPUB/fonts/narrator.ttf   <-- custom font
EPUB/text/chapter01.xhtml
EPUB/text/chapter02.xhtml
EPUB/text/printer.xhtml   <-- not in main.opf
EPUB/notes/note1..10.xhtml
EPUB/assets/colophon.txt
```

## 3. Two packages in `container.xml`

`container.xml` lists **two** `<rootfile>` entries:

```
EPUB/main.opf
EPUB/alternate.opf
```

Most readers open only the first. `main.opf` ("The Unreliable Narrator") has two chapters and a font. `alternate.opf` ("... - Collector's Edition") also adds `text/printer.xhtml` and ten notes.

The end of Chapter II also nudges you here: *"This is not the only printing. Look at what the printer left out of yours."*

## 4. The omitted page

`printer.xhtml` ("Collector's Note") says it was omitted from the standard edition, and contains:

```
I remember the shape of the words more clearly than the words themselves.
PRNUFT2026{CMMHFH_OQ_FIC_DHMAOQ}
```

That looks like a flag, but not a readable one. The "shape of the words" hint also matters: the underscores and repeated letters survive the scrambling. `CMMHFH` has the same repeat pattern as the word `ERRATA`.

## 5. The font lies

`book.css` declares `@font-face { font-family: "Narrator"; src: url(../fonts/narrator.ttf) }`, but `.narrator-clue` is styled `font-family: serif`, so the font never loads in a reader. Inspecting `narrator.ttf` with `fontTools`:

- It is stock **DejaVu Serif v2.37**. I diffed every table against the real font.
- The **only** difference is the `cmap`: the 26 uppercase letters are permuted (for example `A` is drawn as `G`).
- Glyph outlines, widths, kerning and substitutions are untouched.

So the font maps codepoint to a different glyph. If you render `PRNUFT2026{...}` with this font, you see the real flag. (Its Mac Roman `cmap` subtable is left unscrambled, so readers that pick a different subtable disagree about what the "narrator" says.)

## 6. Solve script

```python
import zipfile, re
from io import BytesIO
from fontTools.ttLib import TTFont

epub = zipfile.ZipFile("unreliable-narrator.epub")

# 1. container.xml lists more than one package (rootfile)
container = epub.read("META-INF/container.xml").decode()
print(re.findall(r'full-path="([^"]+)"', container))

# 2. grab the scrambled clue from the omitted page
page = epub.read("EPUB/text/printer.xhtml").decode()
clue = re.search(r'narrator-clue">\s*(\S+)\s*<', page).group(1)
print("raw clue:", clue)

# 3. load the custom font and decode via its cmap (codepoint -> drawn glyph)
font = TTFont(BytesIO(epub.read("EPUB/fonts/narrator.ttf")))
cmap = font.getBestCmap()
print("flag:", "".join(cmap[ord(c)] if len(cmap.get(ord(c), "xx")) == 1 else c for c in clue))
```

Output:

```
['EPUB/main.opf', 'EPUB/alternate.opf']
raw clue: PRNUFT2026{CMMHFH_OQ_FIC_DHMAOQ}
flag: CYBLTZ2026{ERRATA_IN_THE_MARGIN}
```

Requires `pip install fonttools`. You can also render the string with Pillow and the font and read it off the image.

## 7. Flag

```
CYBLTZ2026{ERRATA_IN_THE_MARGIN}
```

---

## Takeaways

- An EPUB is a zip. Check `container.xml` for **multiple rootfiles**, and compare the manifest against the spine and the actual files.
- A custom font can change what text *looks like* without changing the underlying characters. Inspect the `cmap`.
- Compare the font against the stock one it was built from. A diff showed the only change was the `cmap`, so I didn't waste time hunting in outlines or widths.
- The hint about the "shape of the words" suggests a pattern-based solve (monoalphabetic substitution) is also possible without touching the font.

---

## Unsolved: *Unreliable Narrator: The Collector's Edition* (445 pts)

I did not solve the sequel, which reuses the **same EPUB file** (identical MD5 across downloads). What I found and ruled out:

**Structure**
- `alternate.opf`'s spine lists 8 of the 10 notes in a specific order (5, 7, 2, 10, 8, 4, 1, 9), mostly `linear="no"`. Notes 3 and 6 are in the manifest but not the spine.
- Each note holds a 5-letter `flag-fragment`. The colophon reads like an ordering riddle: the *catalogue* (manifest), the *binding* (spine), and "what I walked past" (non-linear items).
- The note files, font, printer page and chapters were regenerated later than the rest of the package (zip timestamps), but the chapters contain no hidden text or typos.

**Tried on the fragments (none produced English)**
- Font decoding applied once, several times, inverted, and every global power (0 to 339) combined with every Caesar shift.
- Caesar, Atbash, rail fence, base32, base-26 numeric readings.
- Vigenère and Beaufort with every word in the book as key, plus the puzzle's own strings as keys.
- Running key from the story text at every offset.
- Per-fragment shifts keyed to note number or spine position.
- Whole-word constraint solving and pattern-based crib dragging.

The spine-ordered fragment string has an index of coincidence of about 0.04, which looks random, so it likely needs a key or rule I never found. I'd love to see how it was meant to be solved.
