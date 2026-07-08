# Catalog Auth Flow (Inferred)

Companion to [catalog_api.md](catalog_api.md). Documents the pairing
and session mechanism from what's visible on-device.

Nothing here is captured from live network traffic — pure inference
from filesystem + Setup.pdx UI + Panic ecosystem hints. Marked as such.

## Player identity

Every Panic user has:

- **Panic account ID** — numeric, seen in bundleIDs like
  `user.289230.com.foo.bar` (289230 = uploader account).
- **Account email** — used for web login.
- **Optional 2FA**.

Web login: username/password + optional TOTP at `https://play.date/`.

## Device pairing

Setup.pdx flow (`Setup.pdx/Server/` directory implies a small local HTTP
server on the device during pairing):

1. User opens Panic's paired-device page on their laptop/phone
   (browser-based).
2. Playdate shows a QR code or numeric pairing code.
3. Web tab sends a POST to `https://play.date/api/v2/devices/pair/`
   (inferred endpoint) with the code + browser session cookie.
4. Backend generates a **device auth token**.
5. Web sends token to Playdate over USB CDC serial (via the Panic
   Setup helper app) OR over Wi-Fi (later firmware, using the
   local server at `Setup.pdx/Server/`).
6. Playdate stores token on the boot/data partition.

The `Setup.pdx/Server/` directory content isn't visible here (we mounted
Data/ only, not Boot/). Structure inferred from the fact that Setup.pdx
is the only bundle that needs to accept credentials.

## Session/auth

For every Catalog API call after pairing:

```
Authorization: Bearer <device-token>
```

or possibly:

```
Cookie: pd_session=<opaque>
```

- **Web client**: cookie-based (standard browser session).
- **Device client**: bearer token from pairing, stored on Boot
  partition, presented on every API call.

Backend maps the token to a Panic account, filters `authorized: true`
on games the user owns (visible in the summary response's `authorized`
field, which was `false` for the 208/222 games we don't own but `true`
for 14 we do — matches the count of installed Catalog games).

## Purchase flow

1. User taps "Buy" on a game in the on-device Catalog.
2. Device Catalog client POSTs `/games/<bundleID>/purchase/`.
3. Backend either:
   - Debits a stored payment method + returns a download token, OR
   - Returns a redirect: "please complete on web".
4. Device downloads the `_protected.zip` from a signed CDN URL.
5. Device unpacks, installs to `Games/Purchased/`, marks `authorized=true`
   in local metadata.

## Web-side purchase

Web checkout at `play.date/games/<slug>/`:

1. Standard e-commerce flow (Stripe or Panic's own payment stack).
2. On success, backend marks the account as owning the game.
3. Next device sync (Catalog app poll) shows the game as `authorized`.
4. User initiates download on device.

## Session refresh

Device tokens presumably don't expire (or expire on years-scale).
Refresh mechanism unknown. Web sessions use standard browser cookie
expiry.

## Un-pairing

Panic account settings on web have an "Un-pair device" option that
revokes the token. Next Catalog API call from the device fails 401 →
device prompts to re-pair.

## Multi-device

A single Panic account can pair multiple Playdates. All purchased
games appear as `authorized` on all paired devices. Download to each
happens per-device — no cloud sync of save data (see
[../formats/savefile.md](../formats/savefile.md)).

## Wi-Fi provisioning

Also handled by Setup.pdx / Settings.pdx. Firmware exposes:

```
> wifi scan
<list of nearby SSIDs>
> wifi connect <ssid> <psk>
ok
```

Panic's helper apps automate this — the user's phone shares Wi-Fi
credentials over Bluetooth-pairing-like handshake, then the device
saves to its own config.

## Observed evidence tying it together

- Every Catalog game's summary has `authorized: bool` field → server
  knows what this device owns.
- Only 14 of 222 catalog objects show `authorized: true` → matches
  purchases count.
- No auth token file visible in `Data/com.panic.catalog/` → stored on
  Boot partition, out of reach.
- HTTP client uses TLS pinned to Panic root (per network API doc) —
  no user-installable CA can MITM.

## What's not confirmed

- Exact API endpoints for pair/purchase (inferred names only).
- Token format (opaque hex string? JWT? bearer opaque?).
- Storage location on device (Boot partition assumed).
- Renewal semantics.

To confirm any of these: sniff live traffic with a MITM proxy plus a
willingness to install a custom Panic root CA (Panic doesn't offer
one — TLS pinning defeats this). Alternative: firmware decompilation
to find the HTTPS request builder.
