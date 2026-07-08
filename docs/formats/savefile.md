# Save File Conventions — `Data/<bundleID>/*`

No enforced format. Games write whatever they like under their bundle
data dir. Empirical convention: **JSON** files, tab-indented, one file
per save slot.

## Location

Root: `Data/<bundleID>/`. See [../os/device_disk.md](../os/device_disk.md#databundleid).

## Common file names (observed)

| Filename | Meaning |
|----------|---------|
| `data.json` | Default from `playdate.datastore.write()` |
| `settings.json` | Per-user options (audio, difficulty, etc.) |
| `save.json`, `saveslot1.json`, `saveslot2.json` | Progress |
| `file1.json` | Older SDK games (Casual Birder pattern) |
| `highscore.json` | Local best times/scores |
| `data.pdz` | Compressed structured data (rare) |
| `data.json.zip` | Zip-wrapped JSON (CODA pattern) |

## Encoding

- UTF-8 without BOM.
- Newlines: LF.
- Indentation: tab (default of `playdate.datastore` when `pretty=true`).
- Lua tables → JSON objects. `nil` values dropped (JSON has no null on
  round-trip through `playdate.datastore.read`; explicit `false`
  survives).
- Numbers: JSON stores as decimal string; `playdate.datastore` reads
  back as float (loses Lua integer type distinction).

## Verified specimens

### Casual Birder — NPC state map

```json
{
    "NPCStates": {
        "Alison":1,
        "Bandy":1,
        "Cafe Bro":1,
        "Donklas":3,
        "Modo the Mover":5,
        ...
    }
}
```

Flat int-per-key; probably a state-machine step.

### Blippo+ — UI settings

```json
{
    "captions enabled":true,
    "fps enabled":false,
    "is scrambled":false,
    "last unscrambled":2,
    "last viewing":6,
    "read messages indexes": [
        null,
        true
    ],
    "watched credits":false
}
```

Note the `null` in an array — legal JSON but unusual for round-tripped
Lua tables. Author wrote raw JSON, not via `datastore`.

### Wheelsprung — per-level progress with signatures

```json
{
    "progress": [
        {
            "levelId":"levels/tutorial_turn_around.flatty",
            "bestTime":13420,
            "hasCollectedStar":true,
            "signature":"6B91A6BDD98CC474ED63C43CEE5DD238"
        }
    ]
}
```

`signature` is a 32-char hex string — 128 bits, likely MD5 of level
file. Used to invalidate saves when level design changes (game-level
integrity, not DRM).

## Anti-tamper patterns

None of the saves observed are encrypted. Some use CRC or MD5-style
signatures per-entry (Wheelsprung) so the game can detect
hand-edited files. Signature scheme is fully game-specific — no OS
support.

## `playdate.datastore` behavior

```lua
playdate.datastore.write(t, "save", true)    -- writes Data/<bundleID>/save.json (pretty)
local t = playdate.datastore.read("save")    -- reads Data/<bundleID>/save.json
```

- Default file: `"data"` (`.json` appended).
- `pretty=true` inserts tabs + newlines (larger file, human-readable).
- `pretty=false` (default) is minified.
- Absent file: `read` returns `nil`, no error.
- Bad JSON: `read` prints to console + returns `nil`.

## Quota

Data partition is shared across all games. No per-game quota advertised
in the API, but total data size is bounded by the flash's user
partition (a few dozen MB). Games writing multi-MB saves will
eventually run out.

## Cross-device sync

None. Playdate saves stay on the physical device. Panic has hinted at
cloud sync but no API surface exists yet in `datastore` or `network`.
