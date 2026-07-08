# Framebuffer & LCD driver

## Panel

- Sharp Memory LCD, part LS027B7DH01A (or interchangeable).
- 400 × 240 pixels, 1 bit each.
- Memory-in-pixel: each pixel latches until re-written.
- Requires periodic VCOM inversion (~1 s cadence) or the LCD burns in.
- Reflective; no backlight, no color filter, no polarizer.
- Contrast ~10:1 in daylight; drops in low light.

## Framebuffer memory layout

Two buffers in the STM32F746's internal SRAM (fast, single-cycle):

```
BACK  buffer  — get via graphics->getFrame()
FRONT buffer  — get via graphics->getDisplayFrame()
```

Layout of each buffer:

```
uint8_t framebuffer[240][52];  // 240 rows, 52 bytes each
// row bytes = LCD_ROWSIZE = 52
// visible pixels: 400 (bit-packed MSB-left)
// 52 bytes = 416 bits: 400 visible + 16 pad bits on the right
// bit=1 → white, bit=0 → black
```

Pixel `(x, y)` addressing:

```c
static inline int pd_pixel(uint8_t* fb, int x, int y) {
    return (fb[y*52 + (x>>3)] >> (7 - (x & 7))) & 1;
}
static inline void pd_set_pixel(uint8_t* fb, int x, int y, int white) {
    uint8_t* p = &fb[y*52 + (x>>3)];
    uint8_t mask = 1 << (7 - (x & 7));
    if (white) *p |=  mask;
    else       *p &= ~mask;
}
```

## Dirty-row tracking

The LCD is driven over 8-bit SPI at ~8 MHz. Full-frame transmit is
~30 ms — too slow for 30 fps. To hit refresh, the driver only pushes
**rows that changed** since the last flip.

Games use:

```c
pd->graphics->markUpdatedRows(int start, int end);
```

`start`, `end` are row indexes (inclusive `start`, inclusive `end`).
`updateAndDrawSprites()` calls this internally with the union of all
dirty sprite rects' vertical extents.

At flip time (`graphics->display()`), the driver compares each row's
CRC (or a dirty-bitmap) against the last-sent version and skips
unchanged rows. Rows are transmitted with the panel's `M0` update
command:

```
COMMAND byte    ROW_ADDR    52 data bytes    DUMMY 16 bits
   0x80          y+1           bmp[y]           0x0000
```

The row address is **1-based** in the M0 protocol — row 0 is address 1.

## Software double-buffering

`graphics->display()` performs the "flip":

1. Diff back buffer against front buffer, row-by-row.
2. For each differing row, push its bytes over SPI.
3. Copy back → front for the rows that changed.
4. (Optionally) trigger VCOM inversion signal to the panel.

## Scaling / mosaic / offset

Handled at the LCD-driver level, not in the framebuffer. `setScale(s)`
tells the driver to point-sample the back-buffer into a (400/s)×(240/s)
region and expand at push time. `setMosaic(x, y)` down-samples into
tiles at push time. `setOffset` shifts each row's read origin.

This means `getFrame()` still returns a **native 400×240** buffer even
when `setScale(4)` is active — the visible image is 100×60 tiled up.

## Row size gotcha

`LCD_ROWSIZE == 52` because the driver rounds up 400 bits (50 bytes) to
a 32-bit multiple for the DMA engine. **Rows are 52 bytes, not 50** —
addressing bugs are common. Always use `LCD_ROWSIZE`.

## Alternate context

`pushContext(bmp)` swaps the current framebuffer pointer for an
`LCDBitmap`'s data buffer. All subsequent primitive calls target that
bitmap (which has its own `rowbytes`, not 52). `popContext()` restores.

## VCOM inversion

If the panel sees no updates for a while, the runtime still ticks a
VCOM invert every ~1 s to prevent LCD degradation. Games don't manage
this; it's transparent.
