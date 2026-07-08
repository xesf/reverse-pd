# Font Sources — `.fnt` + `.png`

Author-side font input to `pdc`. The compiler pairs one `.fnt` metrics
file with a companion `.png` glyph atlas, and emits a compact `.pft`
(see [pft.md](pft.md)).

## `.fnt` — glyph metrics

Plain-text UTF-8. Two syntactic families are supported:

### Simple form (most common)

One glyph per line, character + advance width:

```
space	4
!		4
"		6
#		7
$		7
%		7
0		7
1		7
A		9
B		9
```

Verified from `PlaydateSDK/Resources/Fonts/Quickboot/Quickboot-7-Medium.fnt`.

Notes on the format:

- **Field separator**: tab. Multiple tabs = padding for alignment; only
  the first-non-tab field is the value.
- **Character**: either a literal glyph (`A`, `!`, `π`) or one of the
  special keywords `space`, `tab`, `newline`, or a hex codepoint
  `0x21` / decimal `33`.
- **Value**: integer, advance in pixels (post-glyph pen-X delta).

### Extended form (with kerning and metrics)

```
tracking=1
height=15
leading=3

space	4
A	9
B	9
A-V	-1                  # kerning pair
A-W	-1
```

Header block at top (blank-line separated from glyphs):

- `tracking` — default spacing added between glyphs.
- `height`   — line height, px.
- `leading`  — additional gap between lines.

Kerning entries: `<left>-<right>` followed by tab + signed integer
adjustment (usually negative to bring pairs closer, e.g. `A` before
`V`).

## `.png` — glyph atlas

Companion image next to the `.fnt`. Naming convention:

```
FooFont.fnt
FooFont-table-<W>-<H>.png   # W = glyph cell width, H = cell height
```

`W` and `H` in the filename are the **fixed cell size** in the atlas.
`pdc` slices the PNG into equal-size cells, top-to-bottom, left-to-right,
and matches them to the `.fnt` in order. Blank cells = missing glyph.

For the shipped `Roobert-24-Keyboard-Medium-table-36-36.png`:

- Cell size: 36×36 px.
- Glyphs stored in codepoint order matching `.fnt` order.
- 1-bit PNG (b/w) — `pdc` converts to the packed `.pft` bitmap format.

## Wide-glyph fonts (CJK)

For Japanese fonts, a **second** `.png` and `.fnt` pair is provided,
using UCS-2 codepoints in the `.fnt`:

```
0x3042	16      # あ
0x3044	16      # い
0x3046	16      # う
```

`pdc` builds a wide-page `.pft` (see [pft.md](pft.md) — `wide=1`).

## Glyph bitmap conventions

- **1 bit per pixel**, MSB-left, `1` = white/on.
- Baseline: implicit at row `height - descent` (the `.fnt` metrics
  imply this).
- Anti-aliasing: not supported. Fonts must be crafted for hard 1-bit
  rendering.

## Author toolchain

Panic distributes a font editor (`Playdate Font Tools`?) that produces
paired `.fnt` + `.png`. Many third-party developers use:

- **BMFont** (AngelCode) with a custom exporter (community-made).
- **FontForge** with a script that dumps 1-bit glyphs.
- **Aseprite** for pixel-precise hand-authored fonts.

## `pdc` conversion pipeline

For each font:

1. Read `.fnt` header metrics.
2. Load matching `.png` via libpng (`libpng 1.6.37`, per strings dump).
3. Convert to 1-bit if needed (threshold or dither depending on flags).
4. For each glyph in `.fnt` order:
   - Extract cell from atlas.
   - Trim to actual glyph bounding box (record bearing).
   - Pack into `.pft` glyph record.
5. Build page index for fast codepoint lookup.
6. Build kerning table.
7. zlib-deflate the whole payload.
8. Emit `Foo.pft` with `Playdate FNT` container header.

## Round-trip example

Author files:

```
project/
├── fonts/
│   ├── MyFont.fnt
│   └── MyFont-table-8-8.png
```

`pdc` output (inside main.pdz OR loose in bundle):

```
MyFont.pft
```

Runtime access:

```c
LCDFont* f = pd->graphics->loadFont("fonts/MyFont", &err);
```

or Lua:

```lua
local f = playdate.graphics.font.new("fonts/MyFont")
playdate.graphics.setFont(f)
```

## Included SDK fonts

Under `PlaydateSDK/Resources/Fonts/`:

- `Asheville-*`  — main system UI font (sans-serif geometric)
- `Roobert-*`    — used in keyboard widget
- `Cuberick-*`   — big display font
- `Namco-1994`   — retro monospace
- `Newsleak Serif` — for readable body text
- `Mikodacs-Clock` — clock display
- `Quickboot`    — small boot/diagnostic font
- `Nontendo`     — pixel display

Each ships as `.fnt` + `.png` pair for you to compile, or already
compiled `.pft` under the SDK's `Resources/Fonts/` for direct use in
your bundle.
