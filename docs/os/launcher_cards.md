# Launcher Cards

The Playdate Launcher's home grid shows one **card** per installed game.
Cards can be static or animated, with configurable idle/pressed/select
state transitions.

## Directory convention

Under the pdx bundle root, path controlled by `pdxinfo` `imagepath=`
(default `launcher/`):

```
Foo.pdx/launcher/
├── card.pdi                    # 350×155 (or 380×230) static card
├── card-highlighted/           # bitmap-table dir: animated highlight
│   ├── card-highlighted-table-N-1.png   # source frames (build-time)
│   └── card-highlighted.pdt             # compiled table
├── card-pressed.pdi            # single frame shown briefly on select
├── icon.pdi                    # 32×32 grid-view icon
├── launchImage.pdi             # full-screen 400×240 while loading
├── card-launchImages/          # optional animated launch sequence
│   └── card-launchImages-table-N-1.pdt
└── wrapping-pattern.pdi        # tiled background pattern (behind card)
```

Some games put cards in the bundle root (`imagepath=.`); Catalog paid
games use `imagepath=metadata` (all launcher assets in `metadata/`).

## Card sizes

| Asset | Dimensions | Purpose |
|-------|-----------|---------|
| `card.pdi` | 350×155 | main tile in the "cover flow" scroll |
| Alt: `card.pdi` | 380×230 | full-height layout |
| `card-highlighted` frames | same as `card.pdi` | animation played when card is centered/focused |
| `card-pressed.pdi` | same as `card.pdi` | one-frame "click" flash |
| `icon.pdi` | 32×32 | small icon in dense grid view |
| `launchImage.pdi` | 400×240 | full-screen splash |
| `wrapping-pattern.pdi` | any (tiled) | pattern shown around the card |

## Animation timing

`card-highlighted-table-<N>-1.pdt` is a horizontal strip of `N` frames.
The Launcher plays them at a fixed 20 FPS (50 ms per frame) unless
overridden.

Advertised override mechanisms (community observed):

- File name suffix `-loop` on the table → forces looping.
- A companion `card-highlighted.json` (rare) can specify
  `{ "frameDurations": [30, 30, 100, 30, ...] }` for per-frame timing.

## `launchImage` animation

If `card-launchImages/` exists, it's played in sequence during the load
transition instead of the static `launchImage.pdi`. Duration is capped
by the actual load time — if the game is ready before the animation
finishes, the last frame holds.

## Wrapping pattern

`wrapping-pattern.pdi` (or PNG named per `pdxinfo`'s `wrappingpattern=`)
tiles across the launcher's negative space around the card. Typical
size: 24×24 or 32×32. Two-tone (black/white) with a 1-bit dither for
"paper texture" feel.

## First-launch card

The Launcher caches thumbnails in
`Data/com.panic.launcher/tiles/<bundleID>.pdi` after first successful
render. On uninstall the cache stays until a rebuild is triggered
(currently only via Settings → Reset).

## Panic's own cards

`Games/Purchased/*.pdx/metadata/` structure (verified on bloxzy):

```
metadata/
├── banner.pdi              # 400×208 banner for Catalog detail page
├── card.pdi                # standard card
├── card-highlighted/       # animation
├── icon.pdi                # 32×32
├── icon-highlighted/       # animated icon (grid view)
├── launchImage.pdi
└── wrapping-pattern.pdi
```

Note the extra `icon-highlighted/` — even the tiny grid-view icon can
animate on focus.

## Card animation performance

Card animations run **on the Launcher's Lua VM**, which is itself
another `.pdx` — the Launcher decodes `.pdt` frames and blits them at
its own refresh rate. Games with too many frames or high visual detail
can make the Launcher stutter; the Launcher throttles at 30 fps for
its own draw.

## Fade & scaling

The Launcher applies a **1-bit dither fade** to non-selected cards.
Selected card is drawn at full contrast; adjacent cards get progressive
dither patterns that visually mimic gray.

## Card art guidelines (per Panic)

- 1-bit, dithered from source art.
- High-contrast bold shapes read best at LCD scale.
- Avoid single-pixel details in card animations — they blink on frame
  changes.
- `icon.pdi` should be readable at 32×32 without dither noise.

## Boot flow interaction

`launchSoundPath` in `pdxinfo` plays during the launch animation. Path
is usually `launcher/launchSound.pda`. Panic games use short 0.5-1s
stings.
