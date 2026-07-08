# playdate_json — Streaming JSON

Source: `C_API/pd_api/pd_api_json.h`.

Streaming SAX-style encoder/decoder. Deliberately small (no full DOM
tree) so RAM cost is bounded. Values are 8-byte tagged unions.

## Types

```c
typedef enum {
    kJSONNull,
    kJSONTrue,
    kJSONFalse,
    kJSONInteger,
    kJSONFloat,
    kJSONString,
    kJSONArray,
    kJSONTable      // "table" = object
} json_value_type;

typedef struct {
    char type;      // json_value_type
    union {
        int   intval;
        float floatval;
        char* stringval;   // owned by parser during decode
        void* arrayval;    // opaque, only valid via sublist callback
        void* tableval;
    } data;
} json_value;
```

Coercion helpers (inline):

```c
static inline int   json_intValue   (json_value);
static inline float json_floatValue (json_value);
static inline int   json_boolValue  (json_value);
static inline char* json_stringValue(json_value);
```

`json_stringValue` returns `NULL` for non-string types (no conversion).
Note the header comment on `json_floatValue`: string→float is stubbed to
0 due to missing `strtof` support in the earlier firmware libc.

## Decoder

```c
typedef struct json_decoder json_decoder;
struct json_decoder {
    void   (*decodeError)(json_decoder*, const char* error, int linenum);

    // Optional per-event callbacks — set what you need, leave rest NULL
    void   (*willDecodeSublist)(json_decoder*, const char* name, json_value_type);
    int    (*shouldDecodeTableValueForKey)(json_decoder*, const char* key);
    void   (*didDecodeTableValue)(json_decoder*, const char* key, json_value);
    int    (*shouldDecodeArrayValueAtIndex)(json_decoder*, int pos);
    void   (*didDecodeArrayValue)(json_decoder*, int pos, json_value);
    void*  (*didDecodeSublist)(json_decoder*, const char* name, json_value_type);

    void*  userdata;

    int    returnString;   // set to 1 to have parser skip and hand back raw JSON
    const char* path;      // "root.foo[3].bar" - current dotted path
};

typedef int json_readFunc(void* ud, uint8_t* buf, int bufsize);
typedef struct {
    json_readFunc* read;
    void*          userdata;
} json_reader;
```

Callbacks fire in document order:
1. `willDecodeSublist(name, kJSONArray|kJSONTable)` — opening `[`/`{`.
2. `should*ValueFor*` — return 0 to skip; the parser drops values.
3. `did*Value` — value delivered. `pos==0` on `didDecodeArrayValue`
   signals a bare top-level value (not inside `[...]`).
4. `didDecodeSublist(name, type)` — closing; the return value becomes
   the `sublist` result for the parent.

`returnString = 1` inside `willDecodeSublist` tells the parser to skip
descending and hand you the raw JSON substring back via
`didDecodeTableValue`/`didDecodeArrayValue` as a `kJSONString`. Useful
for lazy sub-parsing.

## Encoder

```c
typedef void json_writeFunc(void* ud, const char* str, int len);

typedef struct json_encoder {
    json_writeFunc* writeStringFunc;
    void*           userdata;

    int pretty       : 1;   // insert newlines + indent
    int startedTable : 1;   // internal state
    int startedArray : 1;
    int depth        : 29;

    void (*startArray)   (struct json_encoder*);
    void (*addArrayMember)(struct json_encoder*);
    void (*endArray)     (struct json_encoder*);
    void (*startTable)   (struct json_encoder*);
    void (*addTableMember)(struct json_encoder*, const char* name, int len);
    void (*endTable)     (struct json_encoder*);
    void (*writeNull)    (struct json_encoder*);
    void (*writeFalse)   (struct json_encoder*);
    void (*writeTrue)    (struct json_encoder*);
    void (*writeInt)     (struct json_encoder*, int);
    void (*writeDouble)  (struct json_encoder*, double);
    void (*writeString)  (struct json_encoder*, const char*, int);
} json_encoder;
```

## Convenience decoder setup

```c
static inline void json_setTableDecode(json_decoder*, ...);   // fills 3 callbacks
static inline void json_setArrayDecode(json_decoder*, ...);
```

Zero-out the decoder first, then use these; they leave the other
callback set to `NULL` so the parser knows not to invoke.

## API

```c
struct playdate_json {
    void (*initEncoder)(json_encoder*, json_writeFunc*, void* ud, int pretty);
    int  (*decode)      (struct json_decoder*, json_reader,   json_value* outval);
    int  (*decodeString)(struct json_decoder*, const char* s, json_value* outval);
};
```

`decode`/`decodeString` return 1 on success, 0 on error. On success,
`outval` holds the root value (which will always be `kJSONArray` or
`kJSONTable` for real JSON docs; can be a scalar if `returnString`
tricks were used).
