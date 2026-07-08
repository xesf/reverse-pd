# playdate_display — LCD control

Source: `C_API/pd_api/pd_api_display.h`. Thin wrapper over the Sharp
Memory LCD panel driver.

## Panel

- Sharp LS027B7DH01 (or comparable) — 400×240, 1-bit, memory-in-pixel.
- Native refresh capped at ~50 Hz. The runtime default is 30 Hz.
- No backlight; needs external light. VCOM inversion required every ~1 s
  or LCD degrades; firmware handles this transparently.
- Framebuffer stride = **52 bytes/row** (`LCD_ROWSIZE`). That's 416 bits
  per row: 400 visible + 16 pad bits at right. MSB of each byte is
  leftmost pixel; `1` = white, `0` = black.

## API

```c
struct playdate_display {
    int   (*getWidth)(void);          // 400
    int   (*getHeight)(void);         // 240

    void  (*setRefreshRate)(float rate);   // 0..50 Hz. 0 disables auto-flip.
    void  (*setInverted)(int flag);        // 1 = negate all pixels post-render
    void  (*setScale)(unsigned int s);     // 1,2,4,8 — pixel doubling
    void  (*setMosaic)(unsigned int x, unsigned int y);  // 0..3 tile smear
    void  (*setFlipped)(int x, int y);     // 0/1 each axis (user-flip screen)
    void  (*setOffset)(int x, int y);      // scroll whole display by dx,dy

    // 2.7
    float (*getRefreshRate)(void);
    float (*getFPS)(void);            // measured, not target
};
```

## Scale

`setScale(s)` renders the game into a (400/s)×(240/s) logical area and
point-samples up. Values: 1, 2, 4, 8. `getWidth`/`getHeight` still return
400/240 — the scaling is applied at flip time by the LCD driver, not by
the graphics API. Useful for retro-look games (`setScale(2)` → 200×120).

## Mosaic

`setMosaic(x, y)` fills every pixel in an (x+1)×(y+1) tile with the
top-left source pixel, giving a "chunky" look. Range 0..3 per axis.

## Refresh rate quirks

- `setRefreshRate(0)` disables auto-flip; use `graphics->display()`
  manually. Good for menu screens.
- Values above 50 Hz clamp silently; the panel simply can't switch faster.
- Non-integer rates OK; the runtime uses a fractional accumulator.

## Flip vs. offset

`setFlipped(1,1)` gives the user's "upside-down orientation" (crank on
left). `setOffset(dx,dy)` is per-frame scroll for screen-shake style
effects; the offset wraps at the edges.
