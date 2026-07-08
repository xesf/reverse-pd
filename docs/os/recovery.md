# Boot & Recovery Partitions

Playdate flash is split into **three** logical partitions, each
independently exposable over USB Mass Storage:

- **Data**  — user files (`Games/`, `Data/`, `Shared/`, saves) — exposed via `pdutil datadisk`.
- **Boot**  — main OS partition (firmware + system pdx bundles) — `pdutil bootdisk`.
- **Recovery** — golden-image firmware — `pdutil recoverydisk`.

## Boot partition (`bootdisk`)

Contents (based on `/System/` on the SDK Disk/ shim):

```
System/
├── firmware.pdz            # main OS image (Cortex-M7 code + libs)
├── Setup.pdx               # first-run onboarding
├── Launcher.pdx            # tile grid
├── Settings.pdx            # options
├── GameLibrary.pdx         # Season re-download
├── Catalog.pdx             # store client
├── Achievements.pdx        # achievements grid
├── Mirror.pdx              # (?) Mirror endpoint tool
└── boot.cfg                # boot flags (e.g. staged update)
```

**Not verified on this device** — bootdisk not mounted here. Structure
inferred from SDK `Disk/System/` layout + firmware update notes.

## Recovery partition (`recoverydisk`)

Contents:

```
Recovery/
├── firmware.pdz            # last-known-good firmware
├── updater.pdx             # flasher tool that rewrites Boot partition
├── Setup.pdx               # simplified Setup for first-boot after wipe
└── recovery.cfg            # boot-into-recovery flag
```

Firmware boot code checks `recovery.cfg` at start; if a flag is set,
boots into `updater.pdx` (or Setup) instead of Launcher.

## USB Mass Storage exposure

When any of the three verbs is invoked, firmware:

1. Unmounts internal FS locally.
2. Detaches USB CDC device class.
3. Attaches USB Mass Storage Class exposing the selected partition as
   a FAT-ish (actually Panic proprietary) volume the host OS can read.
4. Volume label = `PLAYDATE` in all cases.
5. Host mounts as removable disk.
6. On clean unmount, firmware reverses steps.

`data` is mountable read-write. `boot` and `recovery` are typically
read-only — firmware may reject writes at the SCSI layer.

## Firmware update flow

Delivered as a `.pdz` bundle downloaded via the Catalog/Settings check:

1. Client downloads `firmware-X.Y.Z.pdz`.
2. Writes to staging area on Data partition (or Staging/).
3. Sets `boot.cfg` flag "staged update pending".
4. Reboots.
5. Bootloader sees flag, runs `updater.pdx` from Recovery.
6. Updater verifies signature (see [drm.md](drm.md) — same crypto stack likely).
7. Updater writes new firmware to Boot partition.
8. Clears flag, reboots into new Boot.
9. On failure at any step, next boot falls back to Recovery's `firmware.pdz`.

## Factory reset

Recovery includes a "factory reset" path exposed through Setup.pdx after
holding a button combo at boot (documented as "Menu + Lock + B" in
Panic's support notes). Actions:

1. Wipe Data partition (`Games/*`, `Data/*`, `Shared/*`, `crashlog.txt`,
   `errorlog.txt`).
2. Re-run Setup.pdx from Boot partition to re-provision.

Does not touch Boot or Recovery firmware.

## Bootloader

The STM32-family bootloader lives in mask ROM (not writable). It:

1. Checks BOOT0 pin at reset.
2. If BOOT0 high → enters STM32 System bootloader (USB DFU, UART, SPI
   modes). Panic disables this pathway in production (BOOT0 grounded).
3. If BOOT0 low → jumps to primary flash at `0x08000000` (Rev A) or
   `0x24000000` (Rev B, AXIM-mapped flash).

Panic's own bootloader (first-stage) then:

1. Verifies primary firmware image signature/CRC.
2. On success → jumps into it.
3. On failure → jumps into recovery firmware.
4. Handles the "staged update" flag by chain-loading `updater.pdx`.

## Recovery mode key combo

Documented: **hold B + Menu + Lock while powering on**. Firmware
detects this in early init and boots into Recovery.

## Signed firmware

Firmware `.pdz` images shipped by Panic are almost certainly signed
(ECDSA on curve P-256 is typical for STM32 firmware). The bootloader
holds the public key in its own flash region — untouchable from
userland. This is what prevents installing modified firmware without
also patching the bootloader (which requires JTAG/SWD hardware access).

## Recovery firmware source

The recovery firmware is a stripped Playdate OS that can:

- Mount `/System/` and Data partitions.
- Speak the USB CDC shell (`?`, `version`, `install`, `bootdisk`, `datadisk`).
- Run `updater.pdx` to rewrite Boot.
- Cannot: load user games, run Wi-Fi, play sound (audio driver stripped).

If you brick a Rev A/B, the recovery mode is the escape hatch. As long
as Boot signing key + recovery firmware are intact, the device can
always be re-flashed.

## Serial number & OTP

STM32 devices have 96-bit factory-programmed unique IDs at
`0x1FF07A10` (F7) or `0x1FF1E800` (H7). Playdate firmware reads this
for the device serial, exposed via `pdutil <port> version` and used as
salt for Catalog device-binding (see [drm.md](drm.md)).

## Not verified on this device

- Actual filesystem type of boot/recovery partitions (probably a Panic
  proprietary log-structured FS, same as data).
- Exact partition sizes.
- Signature algorithm / key length.
- Whether recovery firmware can update itself (usually no — recovery
  is factory-immutable).
