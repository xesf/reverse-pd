# .pdt — Playdate Image Table

Sprite-sheet: an array of same-shape 1-bit bitmaps. Every cell is one
`LCDBitmap` when accessed via `graphics->getTableBitmap(table, idx)`.

## File shape

```
0x00  12  magic "Playdate IMT"
0x0C   4  flags (u32 LE); bit31 = compressed
0x10   4  decompressed_size (u32 LE)
0x14   4  cell_count (u32 LE)      -- number of cells
0x18   4  cell_width  (u32 LE)     -- px, same for all cells
0x1C   4  cell_height (u32 LE)     -- px
0x20  --  zlib payload
```

Verified on `Setup.pdx/images/progress-indicator.pdt`:

```
0x0C: 00 00 00 80                flags = 0x80000000
0x10: dc 36 00 00                decompressed_size = 14044
0x14: 30 00 00 00                cell_count  = 48
0x18: 30 00 00 00                cell_width  = 48 px
0x1C: 2d 00 00 00                cell_height = 45 px
0x20: 78 da ed 98 3f 6f d3 40 ...  zlib
```

## Inflated payload

```c
struct pdt_inflated {
    uint16_t count;      // == outer cell_count
    uint16_t stride;     // rowbytes per cell
    uint16_t width;      // == outer cell_width
    uint16_t height;     // == outer cell_height
    uint32_t flags;      // bit0 = has_mask (all cells share the flag)

    // Then, per cell:
    //   uint8_t bitmap[stride * height];
    //   uint8_t mask  [stride * height];   // if has_mask
};
```

Total inflated size ≈ `header + count * stride * height * (has_mask ? 2 : 1)`.

For the sample: 48 cells × 6 bytes/row × 45 rows × 2 (image+mask) =
25920 bytes. But `decompressed_size == 14044`, so **this asset stores
image only** (`has_mask == 0`). 48 × 6 × 45 = 12960 + header (few
bytes) — plausible.

## Cell addressing

`graphics->getTableBitmap(table, idx)` returns a stack `LCDBitmap` that
shares the underlying data with the table. **Do not free** the returned
bitmap independently.

`getBitmapTableInfo(table, &count, &width)` (2.5) returns
`cell_count` and `cell_width` from the header.

## Filename convention

Author-side, image tables are named `foo-table-<W>-<H>.png` (single
strip) or `foo-table-<COLS>-<ROWS>.png` (grid). `pdc` splits them at
build time.

## Bitmap encoding

Identical to [.pdi](pdi.md): 1 bpp, MSB-left, `1`=white, `0`=black.
Each cell's mask (when present) sits immediately after its bitmap in
the payload, then the next cell begins — cells are interleaved, not
grouped in image/mask planes.
