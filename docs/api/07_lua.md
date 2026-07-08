# playdate_lua — C↔Lua bridge

Source: `C_API/pd_api/pd_api_lua.h`.

The runtime embeds **Lua 5.4** (custom fork with 32-bit `lua_Integer` and
32-bit `lua_Number` — `float`, not `double` — for M7 FPU parity). Games
authored in Lua are compiled by `pdc` into `.pdz` archives of compiled
Lua bytecode + assets. C code can push functions and classes into the
Lua environment.

## Types

```c
typedef void* lua_State;
typedef int (*lua_CFunction)(lua_State* L);

typedef struct LuaUDObject LuaUDObject;   // Playdate-managed userdata
typedef struct LCDSprite   LCDSprite;

typedef enum { kInt, kFloat, kStr } l_valtype;

typedef struct {
    const char*    name;
    lua_CFunction  func;
} lua_reg;

enum LuaType {
    kTypeNil,
    kTypeBool,
    kTypeInt,
    kTypeFloat,
    kTypeString,
    kTypeTable,
    kTypeFunction,
    kTypeThread,
    kTypeObject
};

typedef struct {
    const char* name;
    l_valtype   type;
    union {
        unsigned int intval;
        float        floatval;
        const char*  strval;
    } v;
} lua_val;
```

`lua_val` is used to declare static class members (constants,
enum-like ints) at registration time.

## The vtable

```c
struct playdate_lua {
    // Registration (return 1 on success, 0 with *outErr)
    int  (*addFunction)   (lua_CFunction f, const char* name, const char** outErr);
    int  (*registerClass) (const char* name, const lua_reg* reg,
                           const lua_val* vals, int isstatic, const char** outErr);

    void (*pushFunction)  (lua_CFunction f);
    int  (*indexMetatable)(void);

    void (*stop)  (void);   // pause the Lua VM
    void (*start) (void);

    // Argument stack (1-indexed, per Lua conventions)
    int          (*getArgCount)(void);
    enum LuaType (*getArgType)(int pos, const char** outClass);
    int          (*argIsNil) (int pos);
    int          (*getArgBool)(int pos);
    int          (*getArgInt)(int pos);
    float        (*getArgFloat)(int pos);
    const char*  (*getArgString)(int pos);
    const char*  (*getArgBytes)(int pos, size_t* outlen);
    void*        (*getArgObject)(int pos, char* type, LuaUDObject** outud);
    LCDBitmap*   (*getBitmap)(int pos);
    LCDSprite*   (*getSprite)(int pos);

    // Return stack
    void (*pushNil)(void);
    void (*pushBool)(int);
    void (*pushInt)(int);
    void (*pushFloat)(float);
    void (*pushString)(const char*);
    void (*pushBytes)(const char*, size_t);
    void (*pushBitmap)(LCDBitmap*);
    void (*pushSprite)(LCDSprite*);

    LuaUDObject* (*pushObject) (void* obj, char* type, int nValues);
    LuaUDObject* (*retainObject)(LuaUDObject*);
    void         (*releaseObject)(LuaUDObject*);
    void         (*setUserValue)(LuaUDObject*, unsigned int slot);
    int          (*getUserValue)(LuaUDObject*, unsigned int slot);

    // Invoke Lua-side function from C (slow — has stack allocation cost)
    void (*callFunction_deprecated)(const char* name, int nargs);
    int  (*callFunction)          (const char* name, int nargs, const char** outerr);
};
```

## Class registration

Register a class named `foo`; methods become `foo:method(...)` in Lua.

```c
static int foo_new(lua_State* L)   { /* pushes userdata */ return 1; }
static int foo_bar(lua_State* L)   { /* ... */ return 0; }

static const lua_reg foo_reg[] = {
    { "new", foo_new },
    { "bar", foo_bar },
    { NULL,  NULL }        // sentinel
};

static const lua_val foo_vals[] = {
    { "MAX_ITEMS", kInt, .v.intval = 100 },
    { NULL, 0, {0} }
};

const char* err;
if (!pd->lua->registerClass("foo", foo_reg, foo_vals, 0, &err))
    pd->system->error("register failed: %s", err);
```

`isstatic=1` treats the class as a static namespace (no `new`, only
static methods); `isstatic=0` for object-style with metatable.

## Userdata lifecycle

`pushObject(ptr, type, nSlots)` wraps a raw C pointer into a Lua
userdata with:
- opaque **type tag** — checked by `getArgObject` at retrieval.
- `nSlots` **user values** — additional Lua values kept alive by the
  userdata, indexed via `setUserValue`/`getUserValue`.

Refcount is via `retainObject` / `releaseObject`. GC runs when Lua drops
its last reference AND the C side has released.

## Calling Lua from C

```c
pd->lua->pushInt(42);
pd->lua->pushString("hello");
const char* err = NULL;
int nrets = pd->lua->callFunction("mymod.hi", 2, &err);
if (nrets < 0) pd->system->error("%s", err);
```

`nargs` items are consumed from the C-push stack; return values are
pushed and can be read back via `getArg*` (positions 1..nrets).

Note: **expensive** — the runtime traps into the VM and unwinds Lua's
protected-call machinery on every call. Prefer registering C
functions callable from Lua, not the reverse.

## Metatables

`indexMetatable()` returns 1 when the current lookup is an
`__index` chain (i.e. an inherited method call). Combined with
`getArgObject`, this lets a C function differentiate `x:foo()` vs.
`Class.foo(x)` invocations.

## `stop` / `start`

`stop` pauses the VM (no bytecode runs, C callbacks still fire).
`start` resumes. Used by the runtime around `kEventLock` / `kEventPause`.
