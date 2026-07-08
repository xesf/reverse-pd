# playdate_file — Filesystem

Source: `C_API/pd_api/pd_api_file.h`.

## Filesystem layout

Two overlaid roots (searched in order):

1. **Data/** — `/Data/<bundleID>/` on flash. RW, ~50 MB per-game quota.
   Writeable paths land here.
2. **PDX bundle** — the `.pdx` directory. RO. Contains `main.pdz`,
   assets, `pdxinfo`.

Reads try Data/ first, then bundle. Writes always go to Data/. `stat`,
`mkdir`, `unlink` on paths outside Data/ fail with `-1` and
`geterr()` set.

`kFileReadData` mode variant forces bundle-only read (skip Data/); useful
for loading pristine assets when the user has a save file with the same
name.

## Types

```c
typedef void SDFile;   // opaque handle

typedef enum {
    kFileRead     = 1<<0,   // "r"  — Data/ then bundle
    kFileReadData = 1<<1,   // "r"  — Data/ only? no, bundle only
    kFileWrite    = 1<<2,   // "w"  — truncate + write, Data/
    kFileAppend   = 2<<2    // "a"  — append, Data/  (note the shift by 2 not 3)
} FileOptions;
```

Note the unusual encoding: `kFileAppend == 8`, `kFileWrite == 4`, so
`kFileAppend & kFileWrite == kFileWrite`. Callers pass exactly one mode
flag; don't OR them.

```c
typedef struct {
    int isdir;
    unsigned int size;
    int m_year;   // full year
    int m_month;  // 1..12
    int m_day;    // 1..31
    int m_hour;
    int m_minute;
    int m_second;
} FileStat;
```

## API

```c
struct playdate_file {
    const char* (*geterr)(void);   // last error string, or NULL

    // Directory ops
    int (*listfiles)(const char* path,
                     void (*cb)(const char* path, void* ud),
                     void* ud, int showhidden);
    int (*stat)(const char* path, FileStat* out);
    int (*mkdir)(const char* path);          // recursive
    int (*unlink)(const char* name, int recursive);
    int (*rename)(const char* from, const char* to);

    // Stream ops
    SDFile* (*open)(const char* name, FileOptions mode);
    int     (*close)(SDFile*);
    int     (*read) (SDFile*, void* buf, unsigned int len);
    int     (*write)(SDFile*, const void* buf, unsigned int len);
    int     (*flush)(SDFile*);
    int     (*tell) (SDFile*);
    int     (*seek) (SDFile*, int pos, int whence);   // SEEK_SET/CUR/END
};
```

Return conventions:
- Directory ops: `0` OK, `-1` error (check `geterr`).
- `read`/`write`: bytes transferred, or `-1` on error.
- `open`: `SDFile*` handle or `NULL`.

## Hidden files

`listfiles(..., showhidden=1)` includes dotfiles. Playdate uses `.` prefix
for system meta (`.pdx`, `.playdate/`), rarely visible to games.

## Storage medium

On device, both roots live on the 16 MB NOR flash under a proprietary FS
managed by the STM32 firmware. Wear-leveled log-structured. Games see
POSIX semantics but the underlying write granularity is 4 KB sectors.

## Simulator quirk

On Simulator, "Data/" maps to
`~/Developer/PlaydateSDK/Disk/Data/<bundleID>/` (macOS) or the
equivalent user path on Windows. The bundle is opened from wherever the
`.pdx` directory sits.
