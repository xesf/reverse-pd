# playdate_network — Wi-Fi, HTTP, TCP

Source: `C_API/pd_api/pd_api_network.h` (header title `pd_api_http.h`).
Added in **firmware 3.0**. Wi-Fi hardware (ESP32-family radio) is
present on both Rev A and Rev B boards but was disabled in launch
firmware; it activates once the device is updated to FW ≥ 2.x, and the
C `playdate_network` API arrives in FW 3.0. Units on stale firmware
return `NET_NO_DEVICE`.

## Error codes

```c
typedef enum {
    NET_OK                    =  0,
    NET_NO_DEVICE             = -1,   // no radio
    NET_BUSY                  = -2,
    NET_WRITE_ERROR           = -3,
    NET_WRITE_BUSY            = -4,
    NET_WRITE_TIMEOUT         = -5,
    NET_READ_ERROR            = -6,
    NET_READ_BUSY             = -7,
    NET_READ_TIMEOUT          = -8,
    NET_READ_OVERFLOW         = -9,
    NET_FRAME_ERROR           = -10,
    NET_BAD_RESPONSE          = -11,
    NET_ERROR_RESPONSE        = -12,
    NET_RESET_TIMEOUT         = -13,
    NET_BUFFER_TOO_SMALL      = -14,
    NET_UNEXPECTED_RESPONSE   = -15,
    NET_NOT_CONNECTED_TO_AP   = -16,
    NET_NOT_IMPLEMENTED       = -17,
    NET_CONNECTION_CLOSED     = -18
} PDNetErr;
```

## Wi-Fi status

```c
typedef enum {
    kWifiNotConnected = 0,
    kWifiConnected    = 1,
    kWifiNotAvailable = 2    // tried, no configured AP in range
} WifiStatus;
```

## Access control

Both HTTP and TCP calls hit `requestAccess` first; on cold state the OS
shows a user-facing prompt. `AccessRequestCallback` fires when the
user answers.

```c
enum accessReply { kAccessAsk, kAccessDeny, kAccessAllow };
typedef void AccessRequestCallback(bool allowed, void* ud);
```

`requestAccess(server, port, ssl, purpose, cb, ud)`:
- if the user has previously granted → returns `kAccessAllow` synchronously.
- if denied → returns `kAccessDeny` synchronously.
- otherwise → returns `kAccessAsk`, `cb` fires later with the answer.

## The top-level struct

```c
struct playdate_network {
    const struct playdate_http* http;
    const struct playdate_tcp*  tcp;

    WifiStatus (*getStatus)(void);
    void       (*setEnabled)(bool on, void (*cb)(PDNetErr));

    uintptr_t reserved[3];   // future
};
```

`setEnabled(true, ...)` powers on the radio and joins the configured AP;
callback fires with `NET_OK` on join or an error code otherwise.

## HTTP

```c
typedef void HTTPConnectionCallback(HTTPConnection*);
typedef void HTTPHeaderCallback(HTTPConnection*, const char* key, const char* value);

struct playdate_http {
    enum accessReply (*requestAccess)(const char* server, int port, bool ssl,
                                       const char* purpose,
                                       AccessRequestCallback*, void* ud);

    HTTPConnection* (*newConnection)(const char* server, int port, bool ssl);
    HTTPConnection* (*retain) (HTTPConnection*);
    void            (*release)(HTTPConnection*);

    // Setup
    void  (*setConnectTimeout)(HTTPConnection*, int ms);
    void  (*setKeepAlive)     (HTTPConnection*, bool);
    void  (*setByteRange)     (HTTPConnection*, int start, int end); // Range: bytes=
    void  (*setUserdata)      (HTTPConnection*, void*);
    void* (*getUserdata)      (HTTPConnection*);

    // Requests
    PDNetErr (*get)  (HTTPConnection*, const char* path,
                      const char* headers, size_t headerlen);
    PDNetErr (*post) (HTTPConnection*, const char* path,
                      const char* headers, size_t headerlen,
                      const char* body, size_t bodylen);
    PDNetErr (*query)(HTTPConnection*, const char* method, const char* path,
                      const char* headers, size_t headerlen,
                      const char* body, size_t bodylen);
    PDNetErr (*getError)(HTTPConnection*);

    // Progress + response reading
    void  (*getProgress)     (HTTPConnection*, int* read, int* total);
    int   (*getResponseStatus)(HTTPConnection*);
    size_t(*getBytesAvailable)(HTTPConnection*);
    void  (*setReadTimeout)  (HTTPConnection*, int ms);
    void  (*setReadBufferSize)(HTTPConnection*, int bytes);
    int   (*read)            (HTTPConnection*, void* buf, unsigned int buflen);
    void  (*close)           (HTTPConnection*);

    // Async event hooks
    void (*setHeaderReceivedCallback)  (HTTPConnection*, HTTPHeaderCallback*);
    void (*setHeadersReadCallback)     (HTTPConnection*, HTTPConnectionCallback*);
    void (*setResponseCallback)        (HTTPConnection*, HTTPConnectionCallback*);
    void (*setRequestCompleteCallback) (HTTPConnection*, HTTPConnectionCallback*);
    void (*setConnectionClosedCallback)(HTTPConnection*, HTTPConnectionCallback*);
};
```

### Lifecycle

1. `newConnection` — create handle (refcount = 1).
2. `setKeepAlive`, timeouts, callbacks.
3. `get` / `post` / `query` — issue request. Returns immediately with
   status; actual bytes arrive later.
4. Callbacks fire on main thread:
   - `headerReceived` — per header line, streaming.
   - `headersRead` — full header block done, `getResponseStatus` valid.
   - `response` — body byte available (fires whenever new data lands).
   - `requestComplete` — body fully read (or error).
   - `connectionClosed` — TCP FIN.
5. Read from `response` callback or when `getBytesAvailable` > 0.
6. `close` (optional; auto on refcount drop).
7. `release` — drop your reference.

`getError` returns any latched error (`NET_OK` when clean).

### TLS

`ssl=true` enables HTTPS via mbedTLS (bundled in firmware). Certificate
chain is validated against a Panic-baked root store; no way to add custom
roots from userland.

## TCP

```c
typedef void TCPConnectionCallback(TCPConnection*, PDNetErr);
typedef void TCPOpenCallback(TCPConnection*, PDNetErr, void* ud);

struct playdate_tcp {
    enum accessReply (*requestAccess)(const char* server, int port, bool ssl,
                                       const char* purpose,
                                       AccessRequestCallback*, void* ud);
    TCPConnection* (*newConnection)(const char* server, int port, bool ssl);
    TCPConnection* (*retain) (TCPConnection*);
    void           (*release)(TCPConnection*);
    PDNetErr       (*getError)(TCPConnection*);

    void  (*setConnectTimeout)(TCPConnection*, int ms);
    void  (*setUserdata)      (TCPConnection*, void*);
    void* (*getUserdata)      (TCPConnection*);

    PDNetErr (*open) (TCPConnection*, TCPOpenCallback*, void* ud);
    PDNetErr (*close)(TCPConnection*);

    void (*setConnectionClosedCallback)(TCPConnection*, TCPConnectionCallback*);
    void (*setReadTimeout)             (TCPConnection*, int ms);
    void (*setReadBufferSize)          (TCPConnection*, int bytes);

    size_t (*getBytesAvailable)(TCPConnection*);
    int    (*read) (TCPConnection*, void* buf, size_t len);   // -> bytes or PDNetErr
    int    (*write)(TCPConnection*, const void* buf, size_t len);

    // 3.1
    size_t (*getSentBytesPending)(TCPConnection*);   // socket send buffer occupancy
};
```

`read`/`write` return the byte count on success, or a negative
`PDNetErr` on failure. `getSentBytesPending` (3.1) lets you throttle
producer-side to avoid the ESP32's tiny TX ring overflowing.
