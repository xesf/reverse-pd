# Pulp Runtime

**Pulp** is Panic's browser-based visual game maker for Playdate.
Games authored at [play.date/pulp](https://play.date/pulp) export as
standard `.pdx` bundles. On disk, Pulp games are visually
indistinguishable from Lua games — they use the same container and file
tree — but the runtime is different.

## Identification

Pulp game bundleIDs follow the convention `pulp.<username>.<slug>`
(e.g. `pulp.simongander.coda`, `pulp.snstudios.delved`,
`pulp.quanghoang.solardescent`).

`pdxinfo` for Pulp games has `pdxversion=11000` (SDK 1.1.0-compatible
runtime baseline). Verified specimens:

- `Games/Purchased/CODA.pdx` (`bundleid=pulp.simongander.coda`)
- `Games/Purchased/Delved.pdx` (`bundleid=pulp.snstudios.delved`)

## Bundle layout

```
CODA.pdx/
├── main.pdz              # engine + game data (~25 KB)
├── data.pdz              # nested archive: game rooms/tiles
├── card.pdi              # launcher card
├── chars.pdt             # character tiles (bitmap table)
├── frames.pdt            # frame tiles
├── pipe.pdt              # pipe/border tiles
├── wrapping-pattern.pdi  # launcher wrap pattern
├── icon.pdi              # launcher icon
├── data.json.zip         # ??? — see notes below
└── pdxinfo
```

**Same 25753-byte `main.pdz`** appears across all Pulp games — verified
by comparing CODA and Delved. This confirms `main.pdz` = the **Pulp
runtime engine** (compiled Lua interpreter for Pulp's scripting DSL),
identical bytecode across every Pulp export.

Game-specific data lives in `data.pdz` + the `.pdt` sprite tables +
`data.json.zip`.

## `data.pdz` — game data archive

`Playdate PDZ` container, uncompressed flag. Contains:

- Room graphs (Pulp's "world" is a grid of rooms).
- Tile definitions (which sprites to draw in each cell).
- Script blocks per-tile (Pulp DSL bytecode).
- Sprite frames.

Byte-level structure not fully reversed — needs one plain (non-Catalog)
Pulp game to inflate. Purchased ones have their **top-level** `main.pdz`
encrypted but `data.pdz` **plain** (bit-30 flag not set on nested pdz).
This makes `data.pdz` a good candidate for further RE.

## `data.json.zip`

A **PKZip** wrapper (per file magic) containing the game's JSON export
from the Pulp editor. Probably kept for debugging or re-import into the
editor. Not required at runtime (the engine reads `data.pdz`).

## Pulp DSL

Author-side language, English-ish syntactic:

```
on love do
    tell event.player to
        say "Hi!"
    end
end

on collect do
    tell event.player to
        add 1 to gold
    end
    play "coin"
end
```

Compiled by Pulp's web editor into a bytecode blob inside `data.pdz`.
The runtime engine (`main.pdz`) implements the DSL interpreter in Lua.

## Runtime engine (`main.pdz`)

Encrypted on Catalog exports; plain on user-submitted / free exports.
Reversing needs one free Pulp game.

Educated guess at what's inside based on Pulp's public docs + Lua
API:

- Load `data.pdz` into memory tables.
- Render 16×16 tile grid using `chars.pdt` + `frames.pdt`.
- Dispatch DSL events (`on love`, `on interact`, `on tick`, etc.).
- Implement `say`, `ask`, `menu`, `sound`, `music` intrinsics.
- Maintain player state, room state, global variables.
- Handle input via `playdate.buttonIsPressed` mappings.

## Distinguishing Pulp from Lua at load

The Launcher shows Pulp and Lua games identically. Runtime detection
by the OS: not needed — both go through the same Lua VM boot path
(`kEventInitLua`). Pulp is just a normal Lua game whose `main.lua`
happens to be the Pulp interpreter loop.

## Sound

Pulp games use `.pda` files under `sfx/` (same as Lua games). Music
tracks stored as `.mid` (SMF) — Pulp editor exports MIDI, engine
plays via `playdate.sound.sequence`.

## Screen tile grid

Pulp fixes rendering to a 25×15 grid of 16×16 tiles = 400×240 pixels
= exactly the LCD. Tile atlas from `frames.pdt` and `chars.pdt`.

Coordinate system: `(x, y)` in tile units, top-left origin. Movement is
discrete (one tile at a time), animated by the engine with easing.

## Save format

Pulp games save via `playdate.datastore` — plain JSON under
`Data/<bundleID>/data.json`. See [savefile.md](savefile.md).

## Open questions

- Full opcode table for Pulp DSL bytecode.
- Runtime `main.pdz` decompilation (need free Pulp game).
- `data.pdz` internal structure — top-level directory pointer, tile
  atlas format, script blob layout.
- Whether Pulp encrypts differently from Lua (no — same bit-30 flag
  observed on Catalog Pulp games).
