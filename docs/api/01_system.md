# playdate_sys — System

Source: `C_API/pd_api/pd_api_sys.h`. Header shows 42 function pointers as of
3.1. This is the "kitchen sink" table: allocation, logging, input, menu,
i18n, time, power.

## Types

```c
typedef enum {
    kButtonLeft  = 1<<0,
    kButtonRight = 1<<1,
    kButtonUp    = 1<<2,
    kButtonDown  = 1<<3,
    kButtonB     = 1<<4,
    kButtonA     = 1<<5
} PDButtons;
```
Bitmask. `getButtonState` returns three masks: currently-held, pushed-this-frame,
released-this-frame. Pushed/released are edge triggers, cleared after the
next `update` returns.

```c
typedef enum {
    kPDLanguageEnglish = 0,
    kPDLanguageJapanese,
    kPDLanguageSystem   // "use whatever the user picked"
} PDLanguage;
```

```c
typedef enum {
    kNone           = 0,
    kAccelerometer  = 1<<0,
    // ...
    kAllPeripherals = 0xffff
} PDPeripherals;
```
Only `kAccelerometer` is documented. Enabling wakes the LIS3DH IMU on the
mainboard; ~1 mA extra draw when running.

```c
typedef enum {
    kPDPowerStatusCharging = 1<<0,  // USB VBUS present AND battery <100%
    kPDPowerStatusUsb      = 1<<1,  // USB VBUS present
    kPDPowerStatusScrews   = 1<<2   // dock/screws contact (Stereo Dock)
} PDPowerStatus;
```

```c
struct PDDateTime {
    uint16_t year;     // full year, e.g. 2026
    uint8_t  month;    // 1..12
    uint8_t  day;      // 1..31
    uint8_t  weekday;  // 1=Mon .. 7=Sun (ISO)
    uint8_t  hour;     // 0..23
    uint8_t  minute;
    uint8_t  second;
};

struct PDInfo {           // 3.0
    uint32_t osversion;   // e.g. 0x00020400 for 2.4.x
    PDLanguage language;
    uint32_t pdxversion;  // pdxinfo `pdxversion` field
};
```

## Callback typedefs

```c
typedef int  PDCallbackFunction(void* userdata);            // return 0 -> keep, non-zero -> stop
typedef void PDMenuItemCallbackFunction(void* userdata);
typedef int  PDButtonCallbackFunction(PDButtons b, int down, uint32_t when, void* ud);
```

`PDCallbackFunction` semantics for the update callback are inverted vs.
what the header comment implies — practical convention is `return 1` to
request a display flip, `return 0` to skip.

## Allocation

```c
void* (*realloc)(void* ptr, size_t size);
```

Wrapper over the runtime allocator. Semantics:

- `realloc(NULL, n)`  → `malloc(n)`
- `realloc(p,    0)`  → `free(p)`, returns `NULL`
- otherwise           → resize in-place if possible, else new alloc + memcpy + free.

Games **must** route all heap traffic through this pointer on device —
the linker script places the heap in SDRAM (0xC0000000+), and the
firmware `malloc` walks its own free-list. Using stdlib `malloc` will
link but grabs from a tiny stub heap and will fail after a few KB.

## Logging

```c
int  (*formatString)(char** ret, const char* fmt, ...);
void (*logToConsole)(const char* fmt, ...);
void (*error)(const char* fmt, ...);
int  (*vaFormatString)(char** outstr, const char* fmt, va_list args);
int  (*parseString)(const char* str, const char* fmt, ...);
```

- `formatString` allocates via the internal allocator; caller must `realloc(p, 0)`.
- `logToConsole` writes to serial + Simulator console. Not `printf`; embedded snprintf.
- `error` logs + halts. On device pops up a crash screen with backtrace.
  On Simulator throws to top level.

## Time

```c
unsigned int (*getCurrentTimeMilliseconds)(void); // uptime, wraps ~49.7 days
unsigned int (*getSecondsSinceEpoch)(unsigned int* ms_out); // UNIX epoch, subsec via out ptr
float        (*getElapsedTime)(void);          // seconds since last reset, double precision-ish
void         (*resetElapsedTime)(void);
int32_t      (*getTimezoneOffset)(void);       // seconds east of UTC
int          (*shouldDisplay24HourTime)(void);
void         (*convertEpochToDateTime)(uint32_t epoch, struct PDDateTime* out);
uint32_t     (*convertDateTimeToEpoch)(struct PDDateTime* in);
void         (*delay)(uint32_t milliseconds);  // busy wait, do NOT use in update
```

`delay` blocks the whole runtime; audio underruns after ~20 ms.

## Input

```c
void  (*getButtonState)(PDButtons* current, PDButtons* pushed, PDButtons* released);
void  (*setPeripheralsEnabled)(PDPeripherals mask);
void  (*getAccelerometer)(float* x, float* y, float* z);   // ± ~1.0 = 1g
float (*getCrankChange)(void);          // degrees since last call, signed
float (*getCrankAngle)(void);           // 0..360, absolute; 0 = pointing up
int   (*isCrankDocked)(void);           // 1 = folded flush against side
int   (*setCrankSoundsDisabled)(int f); // returns previous
void  (*setButtonCallback)(PDButtonCallbackFunction*, void* ud, int queuesize);  // 2.4
```

The button callback fires **from the input IRQ**, so `when` is a
sub-frame timestamp (ms) — use it for rhythm games where the 30 Hz
update tick isn't precise enough. `queuesize` bounds the deferred queue.

## Display / video-orientation info

```c
int  (*getFlipped)(void);           // 1 if user has flipped screen upside-down
void (*setAutoLockDisabled)(int d); // 1 = never idle-sleep
void (*drawFPS)(int x, int y);      // draws current FPS to framebuffer at x,y
int  (*getReduceFlashing)(void);    // system-level accessibility flag
```

## System menu

Games get **3** custom menu slots. Any call over the limit returns `NULL`.

```c
typedef struct PDMenuItem PDMenuItem; // opaque

PDMenuItem* (*addMenuItem)          (const char* title, PDMenuItemCallbackFunction* cb, void* ud);
PDMenuItem* (*addCheckmarkMenuItem) (const char* title, int value, PDMenuItemCallbackFunction* cb, void* ud);
PDMenuItem* (*addOptionsMenuItem)   (const char* title, const char** optionTitles, int count, PDMenuItemCallbackFunction* cb, void* ud);
void        (*removeMenuItem)       (PDMenuItem*);
void        (*removeAllMenuItems)   (void);
int         (*getMenuItemValue)     (PDMenuItem*);
void        (*setMenuItemValue)     (PDMenuItem*, int value);
const char* (*getMenuItemTitle)     (PDMenuItem*);
void        (*setMenuItemTitle)     (PDMenuItem*, const char*);
void*       (*getMenuItemUserdata)  (PDMenuItem*);
void        (*setMenuItemUserdata)  (PDMenuItem*, void*);
void        (*setMenuImage)         (LCDBitmap*, int xOffset);
```

Callback fires when the user picks the item **and** the menu closes (i.e.
after `kEventResume`). `setMenuImage` shows a 400×240 image behind the
menu with horizontal parallax offset (xOffset 0..200).

## Power & battery

```c
float          (*getBatteryPercentage)(void);   // 0..100
float          (*getBatteryVoltage)(void);      // ~3.0..4.2 V (LiPo)
PDPowerStatus  (*getPowerStatus)(void);         // 3.1
float          (*getVolume)(void);              // 0..1 system volume, 3.1
void           (*exitToLauncher)(void);         // 3.1, hard-return to Launcher.pdx
```

## Localization / user info (3.0+)

```c
const struct PDInfo* (*getSystemInfo)(void);                    // 3.0
char*                (*getLocalizedText)(const char* key, PDLanguage); // 3.1, in pds file lookup
```

## Restart / launch args (2.7)

```c
void         (*restartGame)(const char* launchargs);
const char*  (*getLaunchArgs)(const char** outpath);
```

`restartGame` re-execs current pdx with args; args become
`getLaunchArgs`' first return string on next boot. Use case:
in-game "level select" jumping across pdz bundles.

## Cross-device Mirror (2.7)

```c
bool (*sendMirrorData)(uint8_t command, void* data, int len);
```
Only meaningful when `kEventMirrorStarted` has fired. `command` is user-defined
1..255. Used by Playdate Mirror desktop tool to receive telemetry.

## Serial (2.4)

```c
void (*setSerialMessageCallback)(void (*cb)(const char* data));
```
Callback receives full lines from USB CDC console, one per invocation.

## Network time (2.7)

```c
void (*getServerTime)(void (*callback)(const char* time, const char* err));
```
Async NTP-ish query to Panic's time server. `time` is ISO 8601 UTC on success.

## Instruction cache (2.0)

```c
void (*clearICache)(void);
```
Required after emitting code into RAM (JIT-ish). ARMv7-M `DSB; ISB; DCCMVAC; ICIMVAU` sequence.
