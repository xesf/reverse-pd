# Playdate Container Header

All Playdate binary asset formats begin with the same 16-byte
container header, then a payload whose interpretation depends on the
`type` in the magic string.

## The header (16 bytes)

```c
struct pd_container_header {
    char     magic[12]; // "Playdate XXX", space padded, no NUL terminator
    uint32_t flags;     // LE; only bit 31 (0x80000000) is documented: payload is zlib-compressed
};
```

Verified magics:

| Bytes                 | Ext       | Payload type |
|-----------------------|-----------|--------------|
| `Playdate PDZ` (0x50 0x6C 0x61 0x79 0x64 0x61 0x74 0x65 0x20 0x50 0x44 0x5A) | `.pdz` | archive |
| `Playdate IMG` | `.pdi` | bitmap |
| `Playdate IMT` | `.pdt` | bitmap table |
| `Playdate FNT` | `.pft` | font |
| `Playdate AUD` | `.pda` | audio (ADPCM) |
| `Playdate VID` | `.pdv` | video |
| `Playdate STR` | `.pds` | localized strings |

## Flag bits (offset 0x0C, u32 LE)

| Bit | Mask         | Meaning |
|-----|--------------|---------|
| 31  | `0x80000000` | Payload zlib-compressed (starts with `78 DA` stream) |
| 30  | `0x40000000` | Payload **encrypted** (Catalog DRM; see [../os/drm.md](../os/drm.md)) |
| others | reserved  | observed always 0 |

Both bits can be combined in principle; observed Catalog files set only
bit 30 (the encrypted stream is not additionally zlib-wrapped — likely
because zlib's dictionary compression is ineffective on already-random
ciphertext).

## Related: `Playdate PDX` magic

`pdex.bin` (the ARM ELF wrapper) uses a **different** magic:
`Playdate PDX` instead of `Playdate PDZ`. Layout is otherwise
identical:

```
0x00  "Playdate PDX"                     magic
0x0C  flags (u32 LE)
0x10  ARM ELF or encrypted payload
```

Sideload user pdex.bin has flags = 0x00000000; Catalog-purchased
pdex.bin has flags = 0x40000000.

## Compression flag

If `flags & 0x80000000`, the payload begins with an uncompressed
**payload prelude** and then a zlib stream. The prelude carries the
size and shape data needed **before** decompression — the runtime uses
it to pick an inflate destination buffer.

Observed prelude for image/bitmap-family payloads:

```c
struct pd_asset_prelude {
    uint32_t decompressed_size;   // total bytes after inflate
    uint32_t p0;                  // format-specific (usually width)
    uint32_t p1;                  // format-specific (usually height)
    uint32_t p2;                  // format-specific (reserved or flags)
    // followed by zlib bitstream (starts with 0x78 0xDA for level-9)
};
```

Sample: `Setup.pdx/images/check.pdi` (98 bytes on disk)

```
0x00: 50 6c 61 79 64 61 74 65  20 49 4d 47              "Playdate IMG"
0x0C: 00 00 00 80                                       flags = 0x80000000  (compressed)
0x10: 89 00 00 00                                       decompressed_size = 137
0x14: 13 00 00 00                                       width  = 19
0x18: 14 00 00 00                                       height = 20
0x1C: 00 00 00 00                                       reserved = 0
0x20: 78 da 13 66 10 61 60 66  80 01 76 06 ca c0 01 06  zlib stream...
```

The three u32s after `decompressed_size` are format-specific; empty
image formats (`STR`, `FNT`) leave them zero-ish and put their real
counters inside the inflated payload.

For **uncompressed** payloads (`flags == 0`) the format-specific header
starts immediately at offset 0x10.
