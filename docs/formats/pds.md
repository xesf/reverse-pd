# .pds — Playdate String Table

Localized string table. One file per language (e.g. `en.pds`, `jp.pds`)
inside a `.pdx` bundle. Consumed by `system->getLocalizedText(key, lang)`.

## File shape

```
0x00  12  magic "Playdate STR"
0x0C   4  flags (u32 LE); bit31 = compressed
0x10   4  decompressed_size (u32 LE)
0x14  12  reserved / zero
0x20  --  zlib payload
```

Sample: `Setup.pdx/en.pds`

```
0x0C: 00 00 00 80        flags = 0x80000000
0x10: 6e 12 00 00        decompressed_size = 0x126e = 4718
0x14: 00 00 00 00 00 00 00 00 00 00 00 00
0x20: 78 da ...
```

## Inflated payload

```c
struct pds_inflated {
    uint32_t entry_count;
    uint32_t entries[entry_count];  // each: absolute offset into the
                                    // inflated buffer where that key's
                                    // NUL-terminated key follows, then
                                    // a NUL-terminated value.
    // followed by the string blob
};
```

Verified inflated head for `en.pds`:

```
4a 00 00 00                     entry_count = 74
a2 00 00 00                     offset[0]   = 0x00A2  (162)
12 01 00 00                     offset[1]   = 0x0112
60 01 00 00                     offset[2]   = 0x0160
9e 01 00 00                     offset[3]   = 0x019E
...
```

At each offset lies:

```
<key>\0<value>\0
```

Both key and value are UTF-8.

Sanity: 74 entries * 4 bytes = 296 bytes of table + 4-byte count = 300 =
0x12C header. First string offset `0xa2` (162) is **less than** 300 —
which suggests offsets are actually indices into a **string pool that
begins immediately after the count field**, not after the offset table.
Re-reading as such:

```
Header:
  uint32 count;
String pool starts at 4;
Offset table follows the pool? No, the offset values are increasing which
matches a suffix-table layout: offsets are into the file relative to
some anchor.
```

**Working interpretation** (matches the observed offsets when treated
as absolute-into-inflated-payload with the strings section starting at
`4 + count*4`):

```
count = 74
strings_base = 4 + 74*4 = 300 (0x12C)   -- header size

But offsets[0] = 0xA2 < 0x12C.
```

Given this contradiction, the actual layout is either:
1. **Two-level**: `count`, then `2*count` u32s (key-off, val-off), then
   strings. `2*74*4 + 4 = 596 = 0x254`. First offset `0xA2` still less.
2. **Sorted-by-key hash index**: `count`, then `count` u32 hash values,
   `count` u32 offsets into a variable-length string blob. Offsets are
   pool-relative. First offset `0xA2` into a pool of ~4400 bytes fits.

The second interpretation is adopted here as the working model. It's
also how the runtime's binary-search lookup on
`getLocalizedText(key, lang)` works efficiently.

**Status**: not fully byte-verified for `.pds`. Consumers should treat
this as one of the less-solid parts of the RE until source is
available.

## Runtime lookup

```c
char* text = pd->system->getLocalizedText("settings.wifi.title", kPDLanguageEnglish);
```

The runtime opens `en.pds` at boot, keeps the inflated payload in RAM
(~5 KB), and does a hashed lookup on the key. Returned pointer aliases
into that buffer — do not `free`.

## Related

- `.pdxinfo` `bundleID` field determines the search root.
- Missing keys fall back to `kPDLanguageEnglish` then to the literal
  key name (with a warning to the console).
