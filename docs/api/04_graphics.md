# playdate_graphics — 2D drawing

Source: `C_API/pd_api/pd_api_gfx.h` (302 lines, biggest single header).

The graphics subsystem is a **1-bpp bitmap engine** with an optional mask
plane for transparency. All operations render into the current *context*
— either the framebuffer or a user-supplied `LCDBitmap`.

## Constants

```c
#define LCD_COLUMNS 400
#define LCD_ROWS    240
#define LCD_ROWSIZE 52    // bytes per row, 400/8 + pad = 50 + 2
```

## Core types

### LCDRect (integer half-open)

```c
typedef struct {
    int left;
    int right;   // exclusive
    int top;
    int bottom;  // exclusive
} LCDRect;

static inline LCDRect LCDMakeRect(int x, int y, int w, int h) {
    return (LCDRect){ .left=x, .right=x+w, .top=y, .bottom=y+h };
}
```

### LCDColor and patterns

```c
typedef enum {
    kColorBlack,
    kColorWhite,
    kColorClear,   // transparent
    kColorXOR      // invert dest
} LCDSolidColor;

typedef uint8_t   LCDPattern[16];  // rows 0-7 image, rows 8-15 mask
typedef uintptr_t LCDColor;        // LCDSolidColor value OR (uintptr_t)&LCDPattern[0]
```

The pattern is a **tiled 8×8 stipple**. The engine distinguishes color
vs. pointer by numeric magnitude: values `< 16` are treated as
`LCDSolidColor`, everything else as a pattern pointer. Users almost
always store patterns in static memory to keep the pointer valid.

`LCDOpaquePattern` macro sets the mask to all-ones (fully opaque):

```c
#define LCDOpaquePattern(r0,r1,r2,r3,r4,r5,r6,r7) \
    {(r0),(r1),(r2),(r3),(r4),(r5),(r6),(r7), \
     0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff}
```

### Draw modes

```c
typedef enum {
    kDrawModeCopy,             // src overwrites dst
    kDrawModeWhiteTransparent, // white pixels skipped
    kDrawModeBlackTransparent,
    kDrawModeFillWhite,        // any non-transparent src -> white
    kDrawModeFillBlack,
    kDrawModeXOR,
    kDrawModeNXOR,             // NOT(src XOR dst)
    kDrawModeInverted          // src inverted, then copy
} LCDBitmapDrawMode;

typedef enum {
    kBitmapUnflipped,
    kBitmapFlippedX,
    kBitmapFlippedY,
    kBitmapFlippedXY
} LCDBitmapFlip;
```

### Text

```c
typedef enum {
    kASCIIEncoding,
    kUTF8Encoding,
    k16BitLEEncoding   // UCS-2 little-endian (for CJK glyphs)
} PDStringEncoding;

typedef enum { kWrapClip, kWrapCharacter, kWrapWord } PDTextWrappingMode;
typedef enum { kAlignTextLeft, kAlignTextCenter, kAlignTextRight } PDTextAlignment;
typedef enum { kLineCapStyleButt, kLineCapStyleSquare, kLineCapStyleRound } LCDLineCapStyle;
typedef enum { kPolygonFillNonZero, kPolygonFillEvenOdd } LCDPolygonFillRule;
```

### Opaque handles

```c
typedef struct LCDBitmap      LCDBitmap;      // W×H 1-bit image + mask
typedef struct LCDBitmapTable LCDBitmapTable; // array of same-size bitmaps
typedef struct LCDFont        LCDFont;        // .pft loaded
typedef struct LCDFontData    LCDFontData;    // raw font memory blob
typedef struct LCDFontPage    LCDFontPage;    // 256 glyphs
typedef struct LCDFontGlyph   LCDFontGlyph;
typedef struct LCDTileMap     LCDTileMap;
typedef struct LCDVideoPlayer LCDVideoPlayer;
typedef struct LCDStreamPlayer LCDStreamPlayer;
```

## The vtable

```c
struct playdate_graphics {
    const struct playdate_video* video;      // .pdv playback

    // State
    void (*clear)(LCDColor color);
    void (*setBackgroundColor)(LCDSolidColor);
    void (*setStencil)(LCDBitmap*);          // deprecated
    LCDBitmapDrawMode (*setDrawMode)(LCDBitmapDrawMode);
    void (*setDrawOffset)(int dx, int dy);
    void (*setClipRect)(int x, int y, int w, int h);
    void (*clearClipRect)(void);
    void (*setLineCapStyle)(LCDLineCapStyle);
    void (*setFont)(LCDFont*);
    void (*setTextTracking)(int px);        // additional spacing between glyphs
    void (*pushContext)(LCDBitmap* target); // NULL = framebuffer
    void (*popContext)(void);

    // Primitives
    void (*drawBitmap)      (LCDBitmap*, int x, int y, LCDBitmapFlip);
    void (*tileBitmap)      (LCDBitmap*, int x, int y, int w, int h, LCDBitmapFlip);
    void (*drawLine)        (int x1,int y1,int x2,int y2, int width, LCDColor);
    void (*fillTriangle)    (int x1,int y1,int x2,int y2,int x3,int y3, LCDColor);
    void (*drawRect)        (int x,int y,int w,int h, LCDColor);
    void (*fillRect)        (int x,int y,int w,int h, LCDColor);
    void (*drawEllipse)     (int x,int y,int w,int h,int lineW,
                             float startDeg, float endDeg, LCDColor);
    void (*fillEllipse)     (int x,int y,int w,int h,
                             float startDeg, float endDeg, LCDColor);
    void (*drawScaledBitmap)(LCDBitmap*, int x, int y, float sx, float sy);
    int  (*drawText)        (const void* text, size_t len,
                             PDStringEncoding, int x, int y);

    // LCDBitmap CRUD
    LCDBitmap* (*newBitmap) (int w, int h, LCDColor bg);
    void       (*freeBitmap)(LCDBitmap*);
    LCDBitmap* (*loadBitmap)(const char* path, const char** outerr);
    LCDBitmap* (*copyBitmap)(LCDBitmap*);
    void       (*loadIntoBitmap)(const char* path, LCDBitmap*, const char** outerr);
    void       (*getBitmapData)(LCDBitmap*, int* w, int* h, int* rowbytes,
                                uint8_t** mask, uint8_t** data);
    void       (*clearBitmap)(LCDBitmap*, LCDColor bg);
    LCDBitmap* (*rotatedBitmap)(LCDBitmap*, float degrees, float sx, float sy,
                                int* allocedSize);

    // LCDBitmapTable
    LCDBitmapTable* (*newBitmapTable)(int count, int w, int h);
    void            (*freeBitmapTable)(LCDBitmapTable*);
    LCDBitmapTable* (*loadBitmapTable)(const char* path, const char** outerr);
    void            (*loadIntoBitmapTable)(const char* path, LCDBitmapTable*, const char** outerr);
    LCDBitmap*      (*getTableBitmap)(LCDBitmapTable*, int idx);

    // LCDFont
    LCDFont*      (*loadFont)(const char* path, const char** outerr);
    LCDFontPage*  (*getFontPage)(LCDFont*, uint32_t codepoint);
    LCDFontGlyph* (*getPageGlyph)(LCDFontPage*, uint32_t cp, LCDBitmap** bmpOut, int* advanceOut);
    int           (*getGlyphKerning)(LCDFontGlyph*, uint32_t glyph, uint32_t next);
    int           (*getTextWidth)(LCDFont*, const void* text, size_t len,
                                  PDStringEncoding, int tracking);

    // Framebuffer access
    uint8_t*   (*getFrame)(void);            // "back buffer", stride LCD_ROWSIZE
    uint8_t*   (*getDisplayFrame)(void);     // "front buffer"
    LCDBitmap* (*getDebugBitmap)(void);      // Simulator only
    LCDBitmap* (*copyFrameBufferBitmap)(void);
    void       (*markUpdatedRows)(int start, int end);   // for partial updates
    void       (*display)(void);             // flip

    // Misc
    void (*setColorToPattern)(LCDColor* out, LCDBitmap* src, int x, int y);
    int  (*checkMaskCollision)(LCDBitmap*, int x1,int y1, LCDBitmapFlip,
                               LCDBitmap*, int x2,int y2, LCDBitmapFlip,
                               LCDRect);

    // 1.1
    void (*setScreenClipRect)(int x, int y, int w, int h);
    // 1.1.1
    void    (*fillPolygon)(int n, int* coords, LCDColor, LCDPolygonFillRule);
    uint8_t (*getFontHeight)(LCDFont*);
    // 1.7
    LCDBitmap* (*getDisplayBufferBitmap)(void);
    void       (*drawRotatedBitmap)(LCDBitmap*, int x, int y, float deg,
                                    float cx, float cy, float sx, float sy);
    void       (*setTextLeading)(int extraPx);
    // 1.8
    int        (*setBitmapMask)(LCDBitmap*, LCDBitmap* mask);
    LCDBitmap* (*getBitmapMask)(LCDBitmap*);
    // 1.10
    void       (*setStencilImage)(LCDBitmap* stencil, int tile);
    // 1.12
    LCDFont*   (*makeFontFromData_deprecated)(LCDFontData*, int wide);
    // 2.1
    int        (*getTextTracking)(void);
    // 2.5
    void          (*setPixel)(int x, int y, LCDColor);
    LCDSolidColor (*getBitmapPixel)(LCDBitmap*, int x, int y);
    void          (*getBitmapTableInfo)(LCDBitmapTable*, int* count, int* width);
    // 2.6
    void (*drawTextInRect)(const void* text, size_t len, PDStringEncoding,
                           int x, int y, int w, int h,
                           PDTextWrappingMode, PDTextAlignment);
    // 2.7
    int  (*getTextHeightForMaxWidth)(LCDFont*, const void*, size_t, int maxW,
                                     PDStringEncoding, PDTextWrappingMode,
                                     int tracking, int extraLeading);
    void (*drawRoundRect)(int x,int y,int w,int h, int radius, int lineW, LCDColor);
    void (*fillRoundRect)(int x,int y,int w,int h, int radius, LCDColor);

    // 3.0
    const struct playdate_tilemap*     tilemap;
    const struct playdate_videostream* videostream;

    // 3.1
    LCDFontGlyph* (*getFontGlyph)(LCDFont*, uint32_t cp, LCDBitmap** bmpOut, int* advanceOut);
    LCDFont*      (*makeFontFromData)(LCDFontData*, int wide, int datalength);
};
```

## Contexts

`pushContext(bmp)` redirects all subsequent drawing to `bmp`. `NULL`
pushes the display framebuffer explicitly. Stack is 8 deep (undocumented
but empirically stable). `popContext` restores previous target.

## Clip rectangles

Two clips: `setClipRect` clips in world (post-offset) coordinates,
`setScreenClipRect` clips in raw framebuffer coords. Both applied
intersectively. `clearClipRect` resets to full 400×240.

## Stencil

The stencil is a per-pixel opacity mask applied to all primitives. Modern
API is `setStencilImage(bmp, tile)` — if `tile != 0`, the bitmap tiles to
fill the destination; otherwise anchored at top-left.

## Font model

- `.pft` files decompose into **pages** of 256 codepoints each.
- Each page holds up to 256 `LCDFontGlyph` records.
- `getFontPage` picks the page for a codepoint by dividing by 256.
- Kerning table is per-glyph, right-side lookup by next codepoint.

`makeFontFromData(data, wide, datalength)` builds a font from an
in-memory blob (e.g. embedded resource). `wide=1` for the CJK 2-byte
variant. The pre-3.1 form without `datalength` is deprecated because it
can't safely bound-check.

## Framebuffer

- `getFrame()` — the *back* buffer that primitives draw into.
- `getDisplayFrame()` — the buffer currently on the LCD.
- `getDisplayBufferBitmap()` — returns the *display* buffer as an
  `LCDBitmap` for convenient composition.
- `markUpdatedRows(start, end)` — flag rows dirty; the LCD driver only
  transmits changed rows via SPI, saving power.
- `display()` — flip. If `setRefreshRate` triggers auto-flip, calling
  this manually is harmless (it just fast-paths).

## Videos and tilemaps (3.0)

See:
- [`playdate_video`](../formats/pdv.md) — `.pdv` playback.
- [`playdate_videostream`](../formats/pdv.md#streaming) — network video.
- [`playdate_tilemap`](07_tilemap.md) — index-based sprite sheets.

## The tilemap subsystem

```c
struct playdate_tilemap {
    LCDTileMap*     (*newTilemap)(void);
    void            (*freeTilemap)(LCDTileMap*);
    void            (*setImageTable)(LCDTileMap*, LCDBitmapTable*);
    LCDBitmapTable* (*getImageTable)(LCDTileMap*);
    void            (*setSize)(LCDTileMap*, int wTiles, int hTiles);
    void            (*getSize)(LCDTileMap*, int* wTiles, int* hTiles);
    void            (*getPixelSize)(LCDTileMap*, uint32_t* wPx, uint32_t* hPx);
    void            (*setTiles)(LCDTileMap*, uint16_t* indexes, int count, int rowwidth);
    void            (*setTileAtPosition)(LCDTileMap*, int x, int y, uint16_t idx);
    int             (*getTileAtPosition)(LCDTileMap*, int x, int y);
    void            (*drawAtPoint)(LCDTileMap*, float x, float y);
};
```

Tile indexes are `uint16_t` → up to 65536 tiles per bitmap table. Index 0
is "empty" by convention (nothing drawn).
