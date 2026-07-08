# Achievements — `Shared/Achievements/<bundleID>/`

Firmware-managed cross-game achievement store, added in firmware ~2.x.
Each game gets a directory keyed by `bundleID` under `/Shared/Achievements/`.

## Layout

```
/Shared/Achievements/<bundleID>/
├── Achievements.json    # manifest + progress (grantedAt timestamps)
├── icon.pdi             # game icon (32×32) — reused from launcher
└── catalog-wide.pdi     # catalog tile (400×208) — for the OS achievements grid
```

Verified sample: `/Shared/Achievements/net.palancastudios.scope/`.

## `Achievements.json`

UTF-8 JSON, tab-indented (game-authored). Structure:

```json
{
    "specVersion": "1.0.0",
    "gameID": "net.palancastudios.scope",
    "name": "scope",
    "author": "Alexandre Fontoura",
    "description": "A logic puzzle game for Playdate Console.",
    "version": "1.0.0",
    "iconPath": "icon.pdi",
    "cardPath": "catalog-wide.pdi",
    "achievements": [
        {
            "id": "first_steps",
            "name": "First Steps",
            "description": "Complete your first level",
            "grantedAt": 826874022
        },
        {
            "id": "scope_gold",
            "name": "Scope Gold",
            "description": "Gold medals on all Scope levels",
            "grantedAt": 826897348
        }
    ]
}
```

### Field catalog

| Field | Type | Notes |
|-------|------|-------|
| `specVersion` | string | schema version, currently `"1.0.0"` |
| `gameID` | string | must match owning `bundleID` |
| `name`, `author`, `description`, `version` | string | mirrors pdxinfo |
| `iconPath` | path | relative to this dir, small icon |
| `cardPath` | path | relative, catalog-tile size |
| `achievements[]` | array | per-achievement records |
| `achievements[].id` | string | unique key within game |
| `achievements[].name` | string | display title |
| `achievements[].description` | string | display body |
| `achievements[].grantedAt` | uint32 | UNIX seconds; **0 or missing = locked** |

Optional per-achievement fields observed in the wider ecosystem (not in
this specimen but supported):

- `hidden`: bool — don't show in list until unlocked.
- `iconPath`: string — override per-achievement icon.
- `iconLockedPath`: string — icon when locked.
- `progressMax`: int — for progress-tracked achievements.
- `progress`: int — current value.
- `scoreValue`: int — gamerscore-like weight.

## Write access

Games write to their own `Shared/Achievements/<bundleID>/` via the
standard file API. The OS notifies its achievements UI when
`Achievements.json` is modified — a small pop-up appears if any
`grantedAt` transitions from 0 to non-zero.

There is no C API for achievements yet; games use the file API directly:

```lua
local a = json.decodeFile("/Shared/Achievements/"..bundleID.."/Achievements.json")
a.achievements[1].grantedAt = playdate.getSecondsSinceEpoch()
playdate.file.mkdir("/Shared/Achievements/"..bundleID)
local f = playdate.file.open("/Shared/Achievements/"..bundleID.."/Achievements.json", playdate.file.kFileWrite)
f:write(json.encode(a, true))
f:close()
```

A third-party helper library (`achievements.lua`, community-authored)
wraps this pattern.

## Firmware-level integration

The Launcher's "Achievements" section reads every subdirectory of
`Shared/Achievements/` at boot, builds a global index, and shows per-game
progress. Games without a directory don't appear.

Icon and card images are copied from the bundle at first-unlock, so the
Launcher can show them even when the game isn't installed anymore
(uninstall preserves achievements).

## Community "spec" games observed

Only 5 games have populated Achievements on this device's disk:

```
com.grapefruitopia.triston
com.pawprints.ottosgalacticgroove
net.palancastudios.juggling-jolt
net.palancastudios.scope
pulp.quanghoang.solardescent
```

All use `specVersion: "1.0.0"`. Adoption is opt-in per developer.
