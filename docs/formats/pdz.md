# .pdz — Playdate Archive

The `.pdz` container is a sequence of typed, optionally-zlib-compressed
records. It's the payload of `main.pdz` inside every `.pdx` bundle, and
is also used standalone for shared code/asset packages.

## File shape

```
+-----------------------------+
| 16B container header        |   magic="Playdate PDZ", flags (usually 0)
+-----------------------------+
| entry 0                     |
| entry 1                     |
| ...                         |
| entry N-1                   |
+-----------------------------+
```

The outer `flags` is normally `0` (PDZ never wraps itself in an outer
zlib; each entry compresses independently).

## Entry format

```c
struct pdz_entry {
    uint32_t type_and_hdr;   // low 8 bits: type/flags (see below)
                             // upper 24 bits: reserved / entry index
    char     name[];         // NUL-terminated path (typically "CoreLibs/foo")
    uint32_t size;           // bytes of payload
    uint8_t  data[size];     // payload; may be zlib-compressed
    // 4-byte alignment padding follows to the next entry
};
```

### Type/flags byte

Low 8 bits of `type_and_hdr`:

| Bits | Meaning |
|------|---------|
| `0x80` | Payload is zlib-compressed (bitstream starts with `78 da`) |
| `0x0F` | Content type nibble |

Content-type nibble (inferred; not all values enumerated):

| Value | Extension | Content |
|-------|-----------|---------|
| `0x01` | `.lua` compiled | Lua 5.4 bytecode |
| `0x02` | `.pdz` | nested archive |
| `0x03` | `.pds` | string table |
| `0x04` | `.pft` | font |
| `0x05` | `.pda` | audio |
| `0x06` | `.pdi` | image |
| `0x07` | `.pdt` | image table |
| `0x08` | `.pdv` | video |

*Note:* the exact type→ext mapping was derived from field observations
in `Setup.pdx/main.pdz`; entries whose top type-byte flag is `0x81` are
compressed Lua bytecode (matches `CoreLibs/easing`).

### Upper 24 bits

The upper 24 bits of `type_and_hdr` are non-zero in the wild. Observed
value in `Setup.pdx/main.pdz` first entry: `0x0d69`. This is believed to
be a **compact directory index** (offset into a lookup table maintained
in RAM after mount) but has **not** been independently verified. Older
community RE (jaames/pd-emu) treats it as an opaque per-entry token.

## Sample walk (`Setup.pdx/main.pdz`)

```
0x0000: "Playdate PDZ" 00 00 00 00           container, flags=0
0x0010: 81 69 0d 00                          type=0x01 (Lua), zlib flag; hi=0x0d69
0x0014: "CoreLibs/easing\0"                  name (16B including NUL)
0x0024: 44 28 00 00                          payload size = 10308
0x0028: 78 da cd 5a 6b 6c 1c d5 ...          zlib-compressed Lua bytecode
0x286C: <next entry starts here>
```

## Reader pseudocode

```c
int pdz_walk(const uint8_t* buf, size_t len,
             void (*visit)(const char* name, uint8_t type,
                           const uint8_t* data, uint32_t sz, int compressed))
{
    if (len < 16 || memcmp(buf, "Playdate PDZ", 12) != 0) return -1;
    const uint8_t* p = buf + 16;
    const uint8_t* end = buf + len;

    while (p + 8 < end) {
        uint32_t hdr = read_u32_le(p);
        uint8_t  type = hdr & 0x7F;
        int      compressed = (hdr >> 7) & 1;
        p += 4;

        const char* name = (const char*)p;
        size_t nlen = strnlen(name, end - p);
        if (nlen == (size_t)(end - p)) return -1; // unterminated
        p += nlen + 1;

        if (p + 4 > end) return -1;
        uint32_t sz = read_u32_le(p);
        p += 4;

        if (p + sz > end) return -1;
        visit(name, type, p, sz, compressed);
        p += sz;

        // 4-byte alignment; align next entry start
        while (((uintptr_t)p & 3) && p < end) p++;
    }
    return 0;
}
```

Decompression of a Lua entry:

```c
uint8_t out[BIG];
uLongf out_len = sizeof(out);
uncompress(out, &out_len, entry.data, entry.sz);   // zlib.h
// out[0..out_len] = Lua 5.4 bytecode chunk, starts with 1B 4C 75 61 (\x1BLua)
```

## Lua bytecode

Playdate ships **Lua 5.4** with a custom header. The bytecode chunk
header is standard Lua 5.4:

```
1B 4C 75 61                 magic "\x1BLua"
54                          version 5.4
00                          format (official)
19 93 0D 0A 1A 0A           conv/check
04                          sizeof(instruction)
04                          sizeof(lua_Integer)  = 4  (Playdate uses 32-bit)
04                          sizeof(lua_Number)   = 4  (float, not double)
78 56 00 00                 endian check integer
00 00 20 40                 endian check number (float 2.5)
...                          top-level function...
```

The important divergence from stock Lua 5.4 is `lua_Number = float`. The
compiled bytecode is **not** portable to a stock Lua desktop VM without
patching.

## Nested PDZ

Some entries are themselves `.pdz` files (type nibble = 2, magic
`Playdate PDZ`). This is how `CoreLibs.pdz` gets embedded in a game —
the runtime mounts nested archives transparently.
