# Simulator Plugin ABI

How the Playdate Simulator loads and invokes a game's shared library.

## Per-platform shared library

`pdc` (or the game's Makefile invoking `clang`/`gcc`) produces one
platform-specific shared library:

- **macOS**: `pdex.dylib` (Mach-O 64-bit, x86_64 + arm64 universal)
- **Windows**: `pdex.dll` (PE32+ x64)
- **Linux**: `pdex.so` (ELF x86_64)

Placed next to `pdex.bin` in the pdx bundle root:

```
Foo.pdx/
├── pdex.bin       # ARM Cortex-M7, on-device
├── pdex.dylib     # macOS Simulator
├── pdex.dll       # Windows Simulator
├── pdex.so        # Linux Simulator
├── pdxinfo
└── main.pdz
```

Games can ship a subset — only the platforms they support. Simulator
gracefully errors if the platform-specific library is missing:

```
Missing pdex.dylib in bundle
```

## Symbol contract

The library must export exactly one C symbol:

```c
int eventHandler(PlaydateAPI* playdate, PDSystemEvent event, uint32_t arg);
```

- macOS: no decoration needed (default C linkage).
- Linux: no decoration needed.
- Windows: `__declspec(dllexport)` required; SDK header has:

```c
#ifdef _WINDLL
__declspec(dllexport)
#endif
int eventHandler(PlaydateAPI* pd, PDSystemEvent event, uint32_t arg);
```

## Loader mechanism

**macOS**:

```c
void* h = dlopen("Foo.pdx/pdex.dylib", RTLD_NOW | RTLD_LOCAL);
int (*eh)(PlaydateAPI*, PDSystemEvent, uint32_t) = dlsym(h, "eventHandler");
```

**Linux**: same as macOS.

**Windows**:

```c
HMODULE h = LoadLibraryA("Foo.pdx\\pdex.dll");
int (WINAPI *eh)(PlaydateAPI*, PDSystemEvent, uint32_t)
    = (void*)GetProcAddress(h, "eventHandler");
```

Note: Windows uses **stdcall** convention by default for exported DLL
functions in some MS toolchains — but Panic uses `cdecl` (matches the
default `int (*eh)(...)` signature). The SDK's Windows-side
build files include `/Gd` (cdecl) explicitly.

## API vtable

The Simulator populates a real `PlaydateAPI` struct with function
pointers into its own C++ implementations:

```cpp
class PlaydateAPIImpl {
    PlaydateAPI vtable;
    playdate_sys       sys_impl;
    playdate_graphics  gfx_impl;
    // ...
public:
    PlaydateAPIImpl() {
        vtable.system   = &sys_impl;
        vtable.graphics = &gfx_impl;
        // fill in all pd_api_*.h function pointers
        sys_impl.realloc = &PlaydateAPIImpl::sim_realloc;
        // ...
    }
    PlaydateAPI* get() { return &vtable; }
};
```

Passed to `eventHandler` as the first arg on every call. The pointer is
stable for the process lifetime.

## Threading

Simulator runs `eventHandler` on the **main UI thread** (wxWidgets
event loop). Sound mixing runs on a background audio thread but never
calls back into the game's `eventHandler`.

This matches the device's cooperative model — game code assumes single
threaded operation.

## Simulator-only APIs

Two functions in the vtable behave differently in Simulator:

- `graphics->getDebugBitmap()` — returns a diagnostic overlay (sprite
  bounds highlighted). On device: `NULL`.
- Key event injection: Simulator synthesizes `kEventKeyPressed` /
  `kEventKeyReleased` when the host keyboard is used. On device: only
  fires when Mirror is attached.

Games can `#ifdef TARGET_SIMULATOR` to include Simulator-only paths.
The SDK Makefile defines this on host builds.

## Hot reload

Simulator watches the pdx source directory's mtime. On any modification:

1. Runs `pdc` to rebuild the bundle.
2. Sends the game `kEventTerminate`.
3. `dlclose` (or `FreeLibrary`) the old library.
4. `dlopen` the new one.
5. Calls `kEventInit` again.

Games must clean up in `kEventTerminate` for hot reload to work — leaks
otherwise accumulate across reloads.

## Debugging

Simulator supports attaching a debugger (LLDB, GDB, MSVC) directly to
the process — the game's dylib symbols show up in the debugger like any
other dynamically-loaded module.

Recommended `Makefile` flags for debug builds:

```make
-g -O0 -fno-omit-frame-pointer
```

## Symbol visibility

By default the shared library exports all C-linkage symbols. To keep
the ABI clean, use `-fvisibility=hidden` and mark `eventHandler` with
`__attribute__((visibility("default")))`:

```c
__attribute__((visibility("default")))
int eventHandler(PlaydateAPI* pd, PDSystemEvent event, uint32_t arg) {
    // ...
}
```

## Static vs dynamic linkage

Games link with:

- macOS: `-shared -o pdex.dylib`
- Linux: `-shared -fPIC -o pdex.so`
- Windows: `-shared -o pdex.dll` (mingw) or `/DLL /OUT:pdex.dll` (MSVC)

No dynamic libraries required — everything the game needs comes
through the `PlaydateAPI` vtable. Standard-library calls that the
game makes (via stdlib headers) resolve to the host's libc, not to
anything Panic ships. This is why Simulator games often have
memory-safety issues that don't reproduce on device (glibc's `malloc`
behavior differs from `system->realloc`).

## No dynamic linker on device

The Simulator ABI is a **superset** of the device ABI in this respect:
Simulator can dlopen anything the host allows; device runs the flat
`pdex.bin` payload and can only call through the vtable.

Games that use host-only libraries (e.g. `libcurl`) will build for
Simulator but crash at runtime on device (`pdex.bin` linker warnings
notwithstanding). SDK's Makefile is set up to catch this at link time.

## No pure-Lua Simulator plugin

Pure-Lua games (no C entry point) omit `pdex.*` entirely. Simulator's
built-in Lua VM runs `main.pdz` directly. This makes Simulator behave
identically to device for Lua-only games.
