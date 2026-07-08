# playdate_sprite — Retained-mode scene + collisions

Source: `C_API/pd_api/pd_api_sprite.h`.

The sprite subsystem is a **retained-mode 2D scene graph** with
dirty-rect redraw and a spatial-hash collision world. Games either
retain everything (call `updateAndDrawSprites()` once per frame) or
mix immediate-mode gfx and manual sprite draws.

## PDRect (float, inclusive-size)

```c
typedef struct {
    float x, y, width, height;
} PDRect;

static inline PDRect PDRectMake(float x,float y,float w,float h) {
    return (PDRect){ .x=x, .y=y, .width=w, .height=h };
}
```

Distinct from `LCDRect`:
- `PDRect` is float, x/y/w/h.
- `LCDRect` is int, left/right/top/bottom (half-open).

## Collision types

```c
typedef enum {
    kCollisionTypeSlide,
    kCollisionTypeFreeze,
    kCollisionTypeOverlap,
    kCollisionTypeBounce
} SpriteCollisionResponseType;

typedef struct { float x, y; } CollisionPoint;
typedef struct { int   x, y; } CollisionVector;

struct SpriteCollisionInfo {
    LCDSprite *sprite;    // being moved
    LCDSprite *other;     // colliding into
    SpriteCollisionResponseType responseType;
    uint8_t overlaps;     // 1 = started overlapping; 0 = tunneled
    float ti;             // 0..1, distance along path when hit
    CollisionPoint move;  // actual delta
    CollisionVector normal;
    CollisionPoint touch; // contact point
    PDRect spriteRect;    // sprite AABB at touch
    PDRect otherRect;     // other AABB at touch
};

struct SpriteQueryInfo {
    LCDSprite *sprite;
    float ti1, ti2;               // entry/exit 0..1 along query segment
    CollisionPoint entryPoint;
    CollisionPoint exitPoint;
};
```

## Callback typedefs

```c
typedef void LCDSpriteDrawFunction(LCDSprite* s, PDRect bounds, PDRect drawrect);
typedef void LCDSpriteUpdateFunction(LCDSprite* s);
typedef SpriteCollisionResponseType
    LCDSpriteCollisionFilterProc(LCDSprite* self, LCDSprite* other);
```

## The vtable (abridged, full 60+ fns)

Grouped by concern.

### Global scene

```c
void  (*setAlwaysRedraw)(int flag);          // 1 = ignore dirty rects
void  (*addDirtyRect)(LCDRect);              // manual invalidate
void  (*drawSprites)(void);                  // draw only
void  (*updateAndDrawSprites)(void);         // per-sprite update + draw
int   (*getSpriteCount)(void);
void  (*removeAllSprites)(void);
```

### CRUD

```c
LCDSprite* (*newSprite)(void);
LCDSprite* (*copy)(LCDSprite*);
void       (*freeSprite)(LCDSprite*);
void       (*addSprite)(LCDSprite*);
void       (*removeSprite)(LCDSprite*);
void       (*removeSprites)(LCDSprite** arr, int count);
```

### Placement / transform

```c
void   (*setBounds)   (LCDSprite*, PDRect);
PDRect (*getBounds)   (LCDSprite*);
void   (*moveTo)      (LCDSprite*, float x, float y);
void   (*moveBy)      (LCDSprite*, float dx, float dy);
void   (*getPosition) (LCDSprite*, float* x, float* y);
void   (*setSize)     (LCDSprite*, float w, float h);
void   (*setCenter)   (LCDSprite*, float x, float y);  // 2.1, anchor 0..1
void   (*getCenter)   (LCDSprite*, float* x, float* y);
```

`setBounds` sets rect in world space; `moveTo` shifts the anchor
(default 0,0 = top-left) to (x,y). `setCenter(0.5, 0.5)` makes moveTo
address the sprite's center.

### Appearance

```c
void          (*setImage)     (LCDSprite*, LCDBitmap*, LCDBitmapFlip);
LCDBitmap*    (*getImage)     (LCDSprite*);
void          (*setImageFlip) (LCDSprite*, LCDBitmapFlip);
LCDBitmapFlip (*getImageFlip) (LCDSprite*);
void          (*setDrawMode)  (LCDSprite*, LCDBitmapDrawMode);
void          (*setZIndex)    (LCDSprite*, int16_t);
int16_t       (*getZIndex)    (LCDSprite*);
void          (*setVisible)   (LCDSprite*, int);
int           (*isVisible)    (LCDSprite*);
void          (*setOpaque)    (LCDSprite*, int flag);  // hint: drop-tests below
void          (*setTag)       (LCDSprite*, uint8_t);
uint8_t       (*getTag)       (LCDSprite*);
void          (*setStencil)         (LCDSprite*, LCDBitmap*);   // deprecated
void          (*setStencilImage)    (LCDSprite*, LCDBitmap*, int tile);  // 1.10
void          (*setStencilPattern)  (LCDSprite*, uint8_t p[8]); // 1.7
void          (*clearStencil)       (LCDSprite*);
void          (*setUserdata)        (LCDSprite*, void*);
void*         (*getUserdata)        (LCDSprite*);
void          (*setTilemap)         (LCDSprite*, LCDTileMap*);  // 2.7
LCDTileMap*   (*getTilemap)         (LCDSprite*);
```

`setOpaque(1)` tells the redraw engine that this sprite's bounding rect
completely covers what's below — sprites at lower z within the same
rect are skipped.

### Redraw control

```c
void (*markDirty)         (LCDSprite*);
void (*markDirtyRect)     (LCDSprite*, PDRect); // 3.1
void (*setClipRect)       (LCDSprite*, LCDRect);
void (*clearClipRect)     (LCDSprite*);
void (*setClipRectsInRange)(LCDRect, int startZ, int endZ);
void (*clearClipRectsInRange)(int startZ, int endZ);
void (*setIgnoresDrawOffset)(LCDSprite*, int);   // draw in screen space
```

### Custom draw / update

```c
void (*setUpdateFunction)(LCDSprite*, LCDSpriteUpdateFunction*);
void (*setDrawFunction) (LCDSprite*, LCDSpriteDrawFunction*);
void (*setUpdatesEnabled)(LCDSprite*, int);
int  (*updatesEnabled)   (LCDSprite*);
```

When both `setImage` and `setDrawFunction` are set, the custom draw
function wins.

## Collision world

The engine maintains a **spatial hash grid** (~50 px cell, tunable in
firmware) indexed by AABB.

```c
void (*resetCollisionWorld)(void);
void (*setCollisionsEnabled)(LCDSprite*, int);
int  (*collisionsEnabled)   (LCDSprite*);
void (*setCollideRect)      (LCDSprite*, PDRect);   // in local coords
PDRect (*getCollideRect)    (LCDSprite*);
void (*clearCollideRect)    (LCDSprite*);
void (*setCollisionResponseFunction)(LCDSprite*, LCDSpriteCollisionFilterProc*);
```

### Simulate movement with collision resolution

```c
SpriteCollisionInfo* (*checkCollisions)(LCDSprite* s,
                                        float goalX, float goalY,
                                        float* actualX, float* actualY,
                                        int* outCount);
SpriteCollisionInfo* (*moveWithCollisions)(LCDSprite* s,
                                           float goalX, float goalY,
                                           float* actualX, float* actualY,
                                           int* outCount);
```

- `checkCollisions` — simulate, don't apply.
- `moveWithCollisions` — simulate + apply (calls `moveTo(actualX, actualY)`).

The response type from the filter function determines how the sprite
resolves each contact:
- **Slide** — continue along the tangent.
- **Freeze** — stop at contact.
- **Overlap** — pass through, still report.
- **Bounce** — reflect the remaining move around the collision normal.

Caller must `realloc(0)` the returned arrays via `system->realloc`.

### Queries

```c
LCDSprite**       (*querySpritesAtPoint)   (float x,float y, int* n);
LCDSprite**       (*querySpritesInRect)    (float x,float y,float w,float h, int* n);
LCDSprite**       (*querySpritesAlongLine) (float x1,float y1,float x2,float y2, int* n);
SpriteQueryInfo*  (*querySpriteInfoAlongLine)(float x1,float y1,float x2,float y2, int* n);
LCDSprite**       (*overlappingSprites)    (LCDSprite*, int* n);
LCDSprite**       (*allOverlappingSprites) (int* n);
```

The `SpriteQueryInfo` variant gives entry/exit points for raycast-style
usage.

## Draw pipeline (per `updateAndDrawSprites`)

1. Call each sprite's update function (in insertion order, not z-order).
2. Compute dirty rects: prior positions ∪ current positions of moved/marked.
3. Iterate sprites back-to-front (low z first). For each:
   - Skip if not visible or clipped out.
   - Skip if fully occluded by a higher-z opaque sprite.
   - Set image/stencil/draw-mode state.
   - Call custom draw or blit image.
4. `markUpdatedRows(dirty.top, dirty.bottom)` for the LCD driver.

The dirty-rect optimization is the biggest performance win — a
mostly-static scene refreshes only the few rows that changed, dropping
SPI traffic to the panel and CPU spent on the blit.
