# .pdi — Playdate Image

1-bit-per-pixel bitmap with an optional per-pixel opacity mask.

## File shape

```
0x00  12   magic "Playdate IMG"
0x0C   4   flags (u32 LE)                bit31=1 -> compressed
0x10   4   decompressed_size (u32 LE)    only when compressed
0x14   4   width  (u32 LE)
0x18   4   height (u32 LE)
0x1C   4   reserved (u32 LE, observed 0)
0x20  --   zlib bitstream (deflates to `decompressed_size` bytes)
```

For `flags == 0` (uncompressed) the inflated-payload header layout at
0x10 is identical to the inflated payload described below.

## Inflated payload

```c
struct pdi_inflated_hdr {
    uint16_t width;
    uint16_t height;
    uint16_t stride;      // rowbytes = ceil(width / 8)
    uint16_t flags;       // bit0 = has_mask, bit1 = ??? (see below)
    int32_t  clip_left;   // pixel-crop rectangle (may all be zero if uncropped)
    int32_t  clip_right;
    int32_t  clip_top;
    int32_t  clip_bottom;
    uint8_t  bitmap[stride * height];
    uint8_t  mask  [stride * height];   // present only if flags & 1
};
```

Verified on `Setup.pdx/images/check.pdi`:

```
inflated head: 13 00 14 00 03 00 00 00  00 00 00 00 00 00 07 00 ...
              |-w--|-h--|-str|-flg|--clip_l---|--clip_t---|
w      = 0x0013 = 19  px
h      = 0x0014 = 20  rows
stride = 0x0003 = 3   bytes  (correct for 19 bits → 3 bytes)
flags  = 0x0000       (no mask on this asset)
```

Total inflated size: 137 = 16-byte header + 3×20 (image) + 3×20 (mask?
or padding). The 60 extra bytes suggest **the mask is always present**
in the payload but zeroed out when unused — decoders should trust
`flags`, not compute mask presence from size.

## Bitmap encoding

- **1 bit per pixel**, MSB is leftmost pixel.
- `1` = white, `0` = black.
- Rows are packed left-to-right, top-to-bottom.
- The final byte of each row is padded on the right with the last valid
  bit's color (undefined bits, but typically 0).

## Mask encoding

Same layout as bitmap. `1` = opaque, `0` = transparent (source pixel
skipped in blit). Blit combining: `dst = mask ? bitmap : dst`.

## Clip rectangle

The `clip_*` fields describe a **source crop** applied at draw time — a
non-zero clip lets `pdc` store only the visible bounding box of a
sparse image while preserving its logical (uncropped) placement.

If all four are zero, the whole `width×height` region is drawn.
Otherwise, only the sub-rect `[clip_left, clip_top .. clip_right,
clip_bottom]` is drawn (still packed into `bitmap[]`, not padded).

## Round-trip against SDK

`graphics->loadBitmap("images/check", &err)` yields an `LCDBitmap*` with
`getBitmapData(bmp, &w, &h, &rowbytes, &mask, &data)`:
- `w`, `h`, `rowbytes` match `width`, `height`, `stride` above.
- `data` and `mask` are the two flat byte arrays above; `mask == NULL`
  when the file's `flags & 1 == 0`.

## Uncompressed variant

Small assets (< ~200 B) are sometimes stored uncompressed to skip the
zlib overhead. The layout is the same, just without the outer
`decompressed_size`+`w`+`h`+`reserved` prelude — the inflated header
sits at 0x10 immediately.
