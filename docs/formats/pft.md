# .pft — Playdate Font

Bitmap font with kerning tables. Compiled from source `.fnt` + `.png`
pair by `pdc`.

## File shape

```
0x00  12  magic "Playdate FNT"
0x0C   4  flags (u32 LE); bit31=1 -> compressed
0x10   4  decompressed_size (u32 LE)
0x14   4  wide flag (u32 LE)         0=8-bit codepoints, 1=16-bit (CJK)
0x18   4  reserved / metric (u32 LE)
0x1C   4  reserved (u32 LE)
0x20  --  zlib payload
```

Sample: `Setup.pdx/fonts/font-Cuberick-Bold.pft`

```
0x0C: 00 00 00 80          flags = compressed
0x10: 9c 11 00 00          decompressed_size = 4508
0x14: 0c 00 00 00          wide? or line_height  = 12
0x18: 0f 00 00 00          maybe glyph_height   = 15
0x1C: 00 00 00 00
```

The two mid-header u32s were originally `wide` + spare, but observed
values (`12`, `15`) instead look like line metrics — the runtime may
overload the field. Callers should trust the inflated payload rather
than the outer header for glyph geometry.

## Inflated payload (working model)

```c
struct pft_header {
    uint8_t  line_height;       // px, includes leading
    uint8_t  glyph_max_width;
    uint8_t  glyph_max_height;
    uint8_t  wide;              // 1 => 16-bit codepoint pages, else 8-bit
    uint16_t page_count;
    uint16_t default_page;      // page index used for fallback glyph
    uint16_t page_index_table_off;   // relative to payload start
    uint16_t kerning_table_off;
    uint32_t glyph_data_off;
    // ... per-page arrays and glyph bitmaps follow ...
};

struct pft_page {              // 256 glyphs
    uint16_t glyph_count;
    uint16_t reserved;
    struct { uint16_t codepoint; uint16_t glyph_index; } entries[glyph_count];
};

struct pft_glyph {
    int8_t   advance;          // pixels to advance pen X after this glyph
    int8_t   bearing_x;
    int8_t   bearing_y;
    uint8_t  width;
    uint8_t  height;
    uint8_t  stride;
    uint8_t  bitmap[stride * height];
    // (mask may follow when the "with-mask" bit is set on the header)
};

struct pft_kerning_pair {
    uint32_t left_codepoint;   // in wide fonts; otherwise just uint16_t
    uint32_t right_codepoint;
    int8_t   adjust;
};
```

**Status**: the outer container is verified; the inflated layout above
is derived from the SDK API shape (`getFontPage` → `getPageGlyph` →
`getGlyphKerning` maps 1:1 to page → glyph → kerning tables) and
observed data patterns, but individual field widths have not been
byte-verified against every distributed font.

## API round-trip

```c
LCDFont*      f = pd->graphics->loadFont("fonts/font-Cuberick-Bold", &err);
LCDFontPage*  p = pd->graphics->getFontPage(f, 'A');     // page 0
int adv;
LCDBitmap*    b;
LCDFontGlyph* g = pd->graphics->getPageGlyph(p, 'A', &b, &adv);
int k = pd->graphics->getGlyphKerning(g, 'A', 'V');       // negative, e.g. -1
```

`makeFontFromData(data, wide, datalength)` accepts the raw inflated
payload (starting at the `pft_header`) plus a size. The `wide` param
matches the header's `wide` field; passing an inconsistent value
crashes.

## Codepoint encoding

- **Wide=0**: page = codepoint >> 8, glyph = codepoint & 0xFF. UTF-8
  callers upstream decode into 16-bit before lookup.
- **Wide=1**: same page/glyph decomposition applied to a 21-bit
  codepoint (top bits ignored). Used for CJK glyph tables shipped as
  supplementary `.pft` alongside the ASCII font.
