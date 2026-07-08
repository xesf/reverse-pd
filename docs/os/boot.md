# Boot Flow

## Power-on sequence

```
1. Power switch closed → 3V3 rail up.
2. STM32 ROM bootloader runs (fixed at 0x1FF00000, from mask ROM).
3. BOOT0=0 → jumps to 0x08000000: primary firmware image.
4. Firmware brings up SDRAM (FMC), QSPI flash, LCD, audio codec,
   crank encoder, USB, ESP32 co-processor (Rev D+).
5. Firmware mounts internal flash filesystem (proprietary),
   verifies /System/* signature block.
6. If a firmware update image is staged, boots into updater.pdx.
7. Else if this is first boot after factory reset:
   → launches Setup.pdx.
8. Else:
   → launches Launcher.pdx (the tile grid).
```

## `/System/` layout on device

Ships in the SDK under `Disk/System/`:

```
System/
├── Setup.pdx        first-run onboarding
├── Launcher.pdx     app grid + card animations
├── Settings.pdx     Wi-Fi, Bluetooth (unused publicly), audio, date/time,
│                     accessibility, developer mode
├── GameLibrary.pdx  Season-1 games listing / re-download
└── InputTest.pdx    diagnostics (crank, buttons, IMU) — hidden shortcut
```

## Launcher → game handoff

1. User picks a tile.
2. Launcher calls a private launch API on the firmware (essentially
   `runGame("/Games/com.example.foo.pdx")`).
3. Firmware:
   - Frees Launcher's heap (Launcher itself is a `.pdx`).
   - Mounts the target `.pdx`.
   - Fires the game process:
     - If `main.pdz` contains `pdex.bin`: load ELF → call
       `eventHandler(pd, kEventInit, 0)`.
     - Else: bootstrap Lua VM → call `eventHandler(pd, kEventInitLua, 0)`.
4. On exit or terminate: firmware reverse the process, remounts Launcher.

## Setup.pdx flow

Content of `Setup.pdx`:

```
Setup.pdx/
├── main.pdz         onboarding logic (Lua)
├── images/          UI illustrations
├── videos/          menu_transition.pdv, outro.pdv
├── sfx/             UI clicks and progression sounds (Playdate_UI_*)
├── fonts/           Cuberick, Asheville (2 sizes each)
├── en.pds, jp.pds   localized strings
├── Server/          web assets served over USB CDC — Panic's setup wizard
│                     pushes a WebRTC-ish handshake through here to
│                     transfer wifi credentials from the paired device
└── crankButton.pdz  extra library for the crank-input tutorial
```

The `Server/` subdirectory is remarkable: Setup runs a tiny HTTP server
over USB CDC to accept Wi-Fi credentials from the desktop companion
app. Only Setup.pdx (and later Settings.pdx) has this capability;
regular games can't spin up USB servers.

## Update flow

Firmware updates arrive as `.pdz`-wrapped images under `/System/`. The
current firmware writes the new image to a staging partition on flash,
sets a boot flag, and reboots. The bootloader detects the flag, runs
`updater.pdx` which flashes the new image, clears the flag, reboots
into the new firmware.

## Recovery mode

Hold `B + Menu` at power-on to enter recovery. Firmware skips the
normal boot and enters a `pdutil`-compatible mode where the desktop
tool can push a rescue firmware image via USB. Presented as USB Mass
Storage on macOS/Windows for drag-and-drop of `.pdz` firmware bundles.
