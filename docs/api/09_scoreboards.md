# playdate_scoreboards — Panic-hosted leaderboards

Source: `C_API/pd_api/pd_api_scoreboards.h`. Added in firmware 2.x.

Simple key/value leaderboard service. Board IDs are strings agreed with
Panic (via developer portal); values are `uint32_t`. All ops are
asynchronous — callbacks fire from the main thread after the network
transaction completes.

## Types

```c
typedef struct {
    uint32_t rank;    // 1 = top
    uint32_t value;
    char*    player;  // display name
    char*    boardid; // owning board
} PDScore;

typedef struct {
    char*      boardID;
    unsigned int count;
    uint32_t   lastUpdated;  // seconds since epoch
    int        playerIncluded; // 1 if the local player's row is in scores[]
    unsigned int limit;      // cap the server enforces
    PDScore*   scores;       // count entries
} PDScoresList;

typedef struct {
    char* boardID;
    char* name;              // human-readable
} PDBoard;

typedef struct {
    unsigned int count;
    uint32_t     lastUpdated;
    PDBoard*     boards;
} PDBoardsList;
```

## Callbacks

```c
typedef void (*AddScoreCallback)     (PDScore* result, const char* err);
typedef void (*PersonalBestCallback) (PDScore* result, const char* err);
typedef void (*BoardsListCallback)   (PDBoardsList* boards, const char* err);
typedef void (*ScoresCallback)       (PDScoresList*  scores, const char* err);
```

`err == NULL` on success. On failure, the payload pointer is `NULL`.

## API

```c
struct playdate_scoreboards {
    int  (*addScore)      (const char* boardId, uint32_t value, AddScoreCallback);
    int  (*getPersonalBest)(const char* boardId,               PersonalBestCallback);
    void (*freeScore)     (PDScore*);

    int  (*getScoreboards)(BoardsListCallback);
    void (*freeBoardsList)(PDBoardsList*);

    int  (*getScores)     (const char* boardId, ScoresCallback);
    void (*freeScoresList)(PDScoresList*);
};
```

`add*`/`get*` return `1` if the request was queued, `0` if not
(offline, no board configured, wifi disabled, etc.).

## Memory ownership

The runtime allocates the payload structs and their inner strings via
`system->realloc`. Games **must** call the matching `free*` function
(not `system->realloc(p, 0)`) — the runtime destructor walks nested
allocations.

## Persistence

`addScore` locally caches the value if offline and retries on next
network access. `getPersonalBest` returns the cached best if offline.
