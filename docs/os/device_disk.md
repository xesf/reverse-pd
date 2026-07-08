# Device Disk Layout (Data Partition)

Exposed via `pdutil <port> datadisk`, then mounted as `PLAYDATE` on the
host. This is the **data partition** — the OS partition (`bootdisk`)
and recovery partition (`recoverydisk`) are separate and not covered
here.

## Root

```
/Volumes/PLAYDATE/
├── crashlog.txt          # last N hard-fault dumps (append-only, ~20 KB cap)
├── errorlog.txt          # Lua runtime errors + missing-file messages
├── Data/                 # per-game persistent storage (bundleID-keyed)
├── Games/                # installed games
├── Scores/               # per-game leaderboard cache (mostly empty)
├── Screenshots/          # user-triggered screenshots (Menu + Lock combo)
├── Shared/               # cross-game shared state (see below)
├── Staging/              # firmware upload staging (usually empty)
├── System/               # empty on Rev D unit; likely populated on recovery
└── tmp/                  # in-progress Catalog downloads
```

## `Games/`

Four launcher sections + one top-level bucket:

```
Games/
├── <Foo>.pdx                            # sideloads (user-installed via pdutil)
├── User/                                # Catalog user-submissions (free)
│   └── user.<accountID>.<bundleID>.pdx  # prefixed with uploader account ID
├── Purchased/                           # Catalog paid store
│   └── <bundleID>.pdx                   # DRM: bit-30 encrypted main.pdz/pdex.bin
└── Seasons/                             # Bundled subscription seasons
    ├── Season-001/*.pdx                 # Season 1 (24 games)
    └── Season-002/*.pdx                 # Season 2 (12 games so far)
```

Sideloads and User/ pdx have no DRM. Purchased/ and Seasons/ carry
`hash=` in `pdxinfo` and encrypted top-level executable payloads.
See [drm.md](drm.md).

## `Data/<bundleID>/`

Writable per-game scratch space, up to a soft cap enforced by the
firmware. Games open paths under this root via `playdate_file->open`.

Observed contents (typical):
- `settings.json`, `saveslot1.json` — game state
- `highscore.json` — local best scores
- `*.pdi`, `*.pdv` — cached download assets

Notable Panic-owned dirs:

- `Data/com.panic.launcher/` — Launcher's own settings + tile cache.
- `Data/com.panic.catalog/` — the Catalog client's cache; see below.
- `Data/com.panic.setup/` — persistent Setup wizard state.

## `Data/com.panic.catalog/Cache/`

Local mirror of the Catalog metadata + tile art:

```
Cache/
├── _access.log       # JSON dict {"path": mtime_epoch, ...}
├── <hash>            # JSON blob describing a game (no ext)
├── <hash>.pdi        # tile image, product art
└── <hash>.pdv        # trailer video for the game
```

The bare-number filenames are used as content-addressed cache keys.
The `_access.log` records when each was last touched — used for LRU
eviction.

Sample manifest content (JSON):

```json
[{"name":"Merry Andrew","studio_name":"Tragic Planet",
  "detail_url":"https://play.date/api/v2/games/merry-andrew/",
  "price":5.0,
  "header_image":"https://media-cdn.play.date/media/games/488271/andrew_400x208.pdi",
  "list_image_size":"wide",
  ...}]
```

Confirms the Catalog HTTPS API root: `https://play.date/api/v2/`.
Media CDN: `https://media-cdn.play.date/media/`.

## `Shared/`

Cross-game state. Currently:

```
Shared/
└── Achievements/
    ├── com.grapefruitopia.triston/
    ├── com.pawprints.ottosgalacticgroove/
    ├── net.palancastudios.juggling-jolt/
    ├── net.palancastudios.scope/
    └── pulp.quanghoang.solardescent/
```

Achievements are a firmware-managed feature (added in 2.x) that lets
games register unlocks readable across the OS UI. Directory per
`bundleID`; contents typically small JSON.

## `Screenshots/`

Triggered by user (Menu + Lock chord). PNGs at 400×240, 1-bit.

## `Staging/`

Used during OS + firmware update. Firmware unpacks pdx files pushed
via `pdutil install` here first, verifies, then moves into `Games/`
atomically. Normally empty between installs.

## `tmp/`

Scratch space. Currently observed:

```
media_W_C_WCACRUJGCUJXMPOYHBYPJonas_Abildgaard_Bruun_-_What_time_is_it.pdx_protected.zip
```

A partially-downloaded Catalog game. The file name is
`media_<random>_<title>.pdx_protected.zip`. The `_protected.zip` suffix
is Catalog convention — it's a plain zip (verified) whose *contents*
are the DRM-flagged pdx bundle. See [drm.md](drm.md).

## `crashlog.txt` (analyzed)

Append-only, most-recent-first, ~4 KB blocks per crash. Format:

```
--- crash at YYYY/MM/DD hh:mm:ss ---
build:<git-hash>-<version>-release.<build#>-gitlab-runner
   r0:HHHHHHHH    r1:HHHHHHHH     r2:HHHHHHHH    r3: HHHHHHHH
  r12:HHHHHHHH    lr:HHHHHHHH     pc:HHHHHHHH   psr: HHHHHHHH
 cfsr:HHHHHHHH  hfsr:HHHHHHHH  mmfar:HHHHHHHH  bfar: HHHHHHHH
rcccsr:HHHHHHHH
heap allocated: <bytes>
Lua totalbytes=<n> GCdebt=<n> GCestimate=<n> stacksize=<n>

```

Every stanza ends with a blank line — matches
`firmware_symbolizer.py`'s `\n\n` split regex. Fields:

- `r0..r3, r12, lr, pc, psr` — standard Cortex-M exception frame.
- `cfsr` — Configurable Fault Status Register (SCB.CFSR at 0xE000ED28).
- `hfsr` — Hard Fault Status Register (SCB.HFSR at 0xE000ED2C).
- `mmfar` — MemManage Fault Address Register.
- `bfar` — BusFault Address Register.
- `rcccsr` — Reset & Clock Control CSR (STM32-specific; source of reset).
- `heap allocated` — bytes owned by system allocator at crash time.
- `Lua totalbytes/GCdebt/…` — Lua VM state.

## `errorlog.txt` (analyzed)

Runtime errors that don't hard-fault. Each entry:

```
YYYY/MM/DD hh:mm:ss
/Games/.../foo.pdx
<message>
<optional stack trace>

```

Observed messages:
- `Couldn't find pdz file main.pdz` — bundle install truncated.
- `<game_incompatible>` — pdxversion above firmware capability.
- `File not found.`
- Full Lua stack traces: `core/Animations.lua:146: attempt to index a nil value (field '?')`.

## Firmware build identifier

From crashlog `build:` line:

```
build:83737252-3.0.3-release.198853-gitlab-runner   # 3.0.3
build:8ef6724f-3.0.4-release.200238-gitlab-runner   # 3.0.4
```

- First token = git commit short-hash.
- Second = user-visible version (matches `system->getSystemInfo()->osversion`).
- Third = internal build number.
- Fourth = CI runner tag (Panic uses self-hosted GitLab).

## Recovery/boot partitions

Not exposed under `datadisk`. Use:

- `pdutil <port> bootdisk` — main OS partition. Read-only unless
  updating firmware.
- `pdutil <port> recoverydisk` — golden-image partition for factory
  reset.

## Access rules

The data partition is user-writable in host FS terms, but the
firmware controls what gets **launched**. Editing `pdxinfo` `hash=`
value, corrupting an encrypted `main.pdz`, or renaming `.pdx` dirs
may confuse the launcher — worst case triggers the Catalog client to
re-download.
