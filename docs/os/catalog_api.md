# Catalog API

Reverse-engineered from cached responses under
`/Volumes/PLAYDATE/Data/com.panic.catalog/Cache/`.

Two consumers of the same backend:

- **Device**: `Games/Catalog.pdx` (bundleID `com.panic.catalog`) — the
  on-Playdate store client.
- **Web**: `https://play.date/` — the browser storefront.

Both talk to `https://play.date/api/v2/`. Media CDN is
`https://media-cdn.play.date/media/`. All observed responses are JSON,
public GET (no auth token in cached payloads — presumably session
cookie or device certificate binds the identity server-side).

## Roots

| URL | Purpose |
|-----|---------|
| `https://play.date/api/v2/` | JSON API root |
| `https://media-cdn.play.date/media/` | image/video CDN (.pdi, .pdv) |
| `https://play.date/g/<slug>/` | short-link redirect to game page |
| `https://play.date/games/<slug>/` | web storefront game page (HTML) |

## Endpoints observed

Endpoint patterns extracted from response bodies (bundleIDs and slugs
substituted with placeholders):

```
GET /api/v2/games/<slug>/                          # game detail
GET /api/v2/games/<slug>/purchase/                 # POST target for buy
GET /api/v2/games/<bundleID>/purchase/             # alt purchase URL, using bundleID

GET /api/v2/games/tags/<tag>/                      # paginated list by tag
GET /api/v2/games/groups/<group-slug>/             # paginated list — editorial groups
GET /api/v2/games/schedules/<schedule-slug>/       # paginated list — release schedules
GET /api/v2/games/schedules/<schedule-slug>/purchase/  # season purchase (Season 3 = "three")
```

### Tag slugs seen

`strategy`, `puzzle`, `arcade`, `action`, `adventure` (and others — the
`tags` field of each game is a UI-facing string like `"Puzzle"`,
`"Strategy"`).

### Group slugs seen

- `collections` — top-level catalog
- `pass-n-play` — party/two-player
- `staff-picks-oct2024` — dated staff pick list

### Schedule slugs

- `two`  — Season 2 subscription
- `three` — Season 3 subscription (upcoming as of dump)

## Pagination

List endpoints return the DRF (Django Rest Framework) standard shape:

```json
{
  "count": 67,
  "next": "https://play.date/api/v2/games/tags/strategy/?page=3&page_size=30&size=small",
  "previous": "https://play.date/api/v2/games/tags/strategy/?page_size=30&size=small",
  "current": 2,
  "results": [ /* summary objects */ ]
}
```

Query params:

| Param       | Values seen | Notes |
|-------------|-------------|-------|
| `page`      | 1..N        | 1-indexed |
| `page_size` | `30`        | default; other values likely accepted |
| `size`      | `small`, `wide` | selects `list_image_size` variant to include |

Response items in `results[]` are the **summary** flavor (see below),
not the full game payload.

## Game summary object (in list responses)

Shape observed across 6+ list endpoints:

```json
{
  "name": "10-Day Champion",
  "detail_url": "https://play.date/api/v2/games/10-day-champion/",
  "bundle_id": "com.nebukadstudio.10daychampion",
  "header_image": "https://media-cdn.play.date/media/games/318650/Catalog-Header.pdi",
  "studio_name": "Nebukad Studio",
  "price": 6.0,
  "authorized": false,
  "purchasable": true,
  "purchase_url": "https://play.date/api/v2/games/com.nebukadstudio.10daychampion/purchase/",
  "icon": "https://media-cdn.play.date/media/games/com.nebukadstudio.10daychampion/icon_eF6HxuQ.pdi",
  "list_image": "https://media-cdn.play.date/media/games/318650/Catalog-Small.pdi",
  "list_image_size": "small",
  "ui_state": 0
}
```

Some group listings substitute:

```json
{
  "type": "gamegroup",                      // "game" | "gamegroup"
  "animation_frame_timing": null,
  ...
}
```

Field types:

| Field | Type | Notes |
|-------|------|-------|
| `name` | string | display name |
| `detail_url` | url | full detail endpoint |
| `bundle_id` | string | reverse-DNS |
| `header_image` | url | 400×208-ish `.pdi` for large tile |
| `studio_name` | string | developer/studio |
| `price` | float | USD; `0.0` = free |
| `authorized` | bool | current device has license (i.e. purchased or free-owned) |
| `purchasable` | bool | can be bought now (may be false for pre-release) |
| `purchase_url` | url | POST target |
| `icon` | url | small `.pdi` |
| `list_image` | url | list-tile art |
| `list_image_size` | enum | `"small"` or `"wide"` |
| `ui_state` | int | see below |
| `type` | enum | `"game"` (default) or `"gamegroup"` |
| `animation_frame_timing` | array or null | frame-hold durations for animated tiles |

### `ui_state` distribution (n=222 observed)

| Value | Count | Meaning (inferred) |
|-------|-------|---------------------|
| 0  | 93   | default / plain tile |
| 1  |  2   | ??? |
| 4  | 111  | most common — likely "purchasable" |
| 6  |  4   | ??? |
| 16 |  7   | ??? |
| 20 |  5   | ??? |

`ui_state` is a bitmask driving UI badges (NEW, SALE, HOT, OWNED, etc.).
Not fully decoded.

## Game detail object

Returned by `/api/v2/games/<slug>/`. Superset of summary:

```json
{
  "name": "Hausbau",
  "developer_name": null,
  "staff_note": "",
  "bundle_id": "dev.pemu.hausbau",
  "studio_name": "Peter Mühleder",
  "short_description": "…",
  "description": "Hausbau is a chill, minimalist puzzle game where …",
  "detail_url": "https://play.date/api/v2/games/hausbau/",
  "web_url": "https://play.date/games/hausbau/",
  "price": 2.0,
  "base_price": 5.0,                    // only on discounted items
  "header_image": "https://media-cdn.play.date/media/games/433137/catalog-header.pdi",
  "icon": "https://media-cdn.play.date/…/icon_….pdi",
  "accessibility": "This game uses the A and B buttons, and the crank.",
  "rating": "We think this game is appropriate for everyone.",
  "screenshots": [
    {
      "url": "https://media-cdn.play.date/media/games/com.pemudev.package/hausbau_zen2.pdv",
      "frame_timing": [120, 1, 160, 1, /* … */]
    }
  ],
  "tags": ["Puzzle"],
  "build_size": "3.2 MB",
  "published_date": "10/14/2025",
  "updated_date": null,
  "authorized": false,
  "purchasable": true,
  "purchase_url": "https://play.date/api/v2/games/dev.pemu.hausbau/purchase/",
  "ui_state": 4
}
```

### `screenshots[].frame_timing`

Flat array of alternating `[hold_ms, frame_delta, hold_ms, frame_delta, ...]`.

Example: `[120, 1, 160, 1, 200, 1]` means:
- hold current frame 120 ms, advance by 1 frame
- hold 160 ms, advance by 1
- hold 200 ms, advance by 1

Used to loop a `.pdv` screenshot with variable-rate playback rather than
fixed 30 fps. On the client this maps to `LCDVideoPlayer` with a manual
tick-based frame counter.

### `accessibility` and `rating`

Free-text UX blurbs, not machine-parseable. Panic surfaces them under
the game's info screen.

## Media URLs

Three URL shapes observed on `media-cdn.play.date`:

```
/media/games/<numeric-id>/<slug>.pdi        # legacy, per-numeric-asset-id
/media/games/<bundle-id>/<name>.pdi         # newer, keyed by bundle
/media/gamegroups/<numeric-id>/<name>.pdi   # editorial group art
```

The `<numeric-id>` (e.g. `433137`) is Panic's internal DB primary key
for the game. Only used in URLs — never surfaced to the user.

## Purchase flow (inferred)

Not directly captured (would require live traffic), but from
`purchase_url` shape + Panic ecosystem hints:

1. Client (device or web) POSTs to `/api/v2/games/<bundleID>/purchase/`
   with session cookie (web) or device certificate (device).
2. Server returns download token + `_protected.zip` URL on media CDN.
3. Client downloads to `Data/com.panic.catalog/tmp/<name>_protected.zip`
   (device) or triggers browser download.
4. Client (device only) unpacks zip into `Games/Purchased/<name>.pdx/`,
   verifies encrypted `main.pdz` + `pdex.bin` against `hash=` in
   `pdxinfo`, and registers with launcher.

Panic operates its own payment processor (`play.date` domain includes
account/subscription management). Payment happens on web only —
device-side purchase links redirect the user to log in via a paired
account.

## Catalog client identifier

Device client bundleID: `com.panic.catalog`. Persistent state under
`Data/com.panic.catalog/`. Only `Cache/` observed here — presumably
authentication tokens live elsewhere in the OS partition (not exposed
under `datadisk`).

## Web-only vs. device-only

Both clients hit the same API. Differences:

| Feature | Web | Device |
|---------|-----|--------|
| Browse | ✅ | ✅ |
| Login / account | ✅ (email/password + 2FA) | via desktop pairing |
| Purchase confirmation | ✅ | UI defers to web |
| Download to install | download zip file | streams into `Games/Purchased/` |
| Screenshot playback | `.pdv` decoded in JS | native `LCDVideoPlayer` |
| Localization | Accept-Language header | firmware `PDLanguage` setting |

## Rate limits / errors

Not captured in the cache (only successful GETs preserved). Django REST
Framework typical: `429 Too Many Requests`, `401` for missing auth on
purchase endpoint, `404` for unknown slug.

## Short-link `/g/<slug>/`

Purely a **redirect service** — hits `play.date/g/scope/` and 302s to
`play.date/games/net.palancastudios.scope/`. Used in printed / QR-code
promotional material.

## Observed short-link slugs

`AoG`, `castle`, `cribbage`, `elena-temple`, `ff`, `haus`, `oasis`,
`off-planet`, `picro`, `scope`, `tmgcn` — mnemonic keys, not slugs.

## Non-content endpoints not observed

Not present in this cache, but exist based on ecosystem clues:

- `POST /api/v2/auth/session/` — desktop pairing token exchange
- `GET  /api/v2/user/library/` — list of user's purchased games (drives
  the "Purchased" section)
- `GET  /api/v2/user/subscriptions/` — Season subscription status
- `POST /api/v2/games/<bundleID>/report/` — Bug/crash report submission
- `GET  /api/v2/system/updates/` — firmware update check
- `POST /api/v2/system/telemetry/` — anonymized play stats

These are inferred from the launcher/setup UI flows; not verified.
