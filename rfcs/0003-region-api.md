# RFC 0003: Region API (access, loading and results)

**Status:** Draft. The project lead asked for this design; it is not
accepted yet.
**Topics:** 1a (where API calls run), the loading part of 1 (worlds, regions,
chunks) and the async part of 5b (scheduler).
**Builds on:** [RFC 0001](0001-plugins.md), [RFC 0002](0002-core.md).

## Problem

RFC 0002 decides that worlds are split into regions that tick in parallel and
that crossing regions is explicit. It does not say what plugin code looks
like. The project lead's requirements:

- Simple for common tasks, and advanced down to the lowest level.
- Plugins can load and unload worlds, areas, chunks and players, and
  influence regions.
- Fast: no locks, and no call that waits on another region.

## Decisions

### 1. Plain objects that check their owner

Listeners receive plain objects (`*Player`, `*Entity`, `Block`, `*World`) and
call methods on them directly. A handle valid only during the call
(Dragonfly's `world.Tx`) was considered and rejected: it adds a parameter to
every call for a check that one pointer comparison already does.

```go
r.PlayerJoin(func(e *event.PlayerJoin) {
    e.Player.Message("Welcome!")
    e.Player.Inventory().Add(item.New("minecraft:bread", 16))
})
```

Every region-owned object stores its current owner. A method that reads or
changes region state first compares that owner with the region running the
current goroutine. This is one comparison, with no lock.

### 2. Every method says where it may run

Each exported method's doc comment starts with one of:

| Tag | Meaning | Examples |
|---|---|---|
| `Runs: anywhere` | Safe from any goroutine | `p.Name()`, `p.UUID()`, `p.WritePacket()`, `p.Message()`, `world.Name()` |
| `Runs: owner` | Only on the region that owns the object | `p.SetHealth()`, `p.Inventory()`, `world.SetBlock()`, `e.Position()` |
| `Runs: global` | Only on the global owner | `scoreboard.Set()`, `gamerules.Set()` |

Calls that only send data to a client (`Message`, `WritePacket`, titles,
sounds) are `anywhere`: they go through the connection's own send queue and
touch no region state.

### 3. Wrong-place calls are refused, never redirected

An `owner` or `global` call from the wrong place does nothing and returns
`helix.ErrWrongOwner` (methods without an error log it once per call site).
In debug mode it fails loudly, naming the caller and the owner. It is never
silently forwarded: forwarding changes a call from "happens now" to "happens
later" without the author knowing, which causes bugs that are hard to find.
The fix is always the same: wrap it in `Run`.

### 4. Crossing regions: messages, never locks

| Call | Runs `fn` on |
|---|---|
| `p.Run(fn)`, `e.Run(fn)` | Whichever region owns the player or entity when the task runs (follows it as it moves) |
| `world.RunAt(pos, fn)` | The region that owns `pos`, loading the chunk if needed |
| `world.EachRegion(fn)` | Every region of the world, in parallel |
| `server.Global(fn)` | The global owner |

All of them return at once. The work runs on the target's next tick.

One call can never change two regions in the same instant: the other region
is running on another core, so it would be either a data race or a pause
(locks, deadlocks, slow regions slowing others). Things that interact are
close together, and close regions merge, so vanilla does not need this.
Plugins split such work into steps (take, confirm, give, undo on failure).

### 5. Results

Anything that takes time returns `helix.Result[T]`:

```go
world.LoadChunk(pos).Then(func(c *world.Chunk, err error) {
    // runs back on the caller's owner, so c can be used directly
})
```

- `Then(fn)` runs `fn` on the owner that made the call, on its first tick
  after the result is ready, so no extra `Run` is needed.
- `Wait(ctx)` blocks and is allowed only outside any region or the global
  owner (`main`, a plugin's own goroutine). On a region it returns
  `helix.ErrWouldBlock`: a region never waits.
- `helix.All(results...)` combines several results.
- `world.EachRegion(fn)` returns `Result[[]T]`, one value per region.

### 6. Reading from anywhere: snapshots

At the end of each tick a region publishes a read-only snapshot of cheap
values: player and entity positions, health, dimension. `p.Snapshot()` is
`Runs: anywhere` and may be up to one tick old. Maps, dashboards and Discord
bots use snapshots; game logic uses the live object.

Snapshots must cost nothing when nobody reads them (goal 6); how is decided
by benchmark.

### 7. Loading and unloading

Everything a plugin keeps loaded is held by a **ticket**. A ticket belongs to
a plugin; disabling the plugin releases all its tickets (RFC 0001).

| Simple | What it does |
|---|---|
| `world.LoadChunk(pos)` | Load one chunk briefly (unloads when nothing else holds it) |
| `world.KeepLoaded(area) *Ticket` | Keep a chunk, box or radius loaded and ticking until `ticket.Release()` |
| `world.UnloadChunk(pos)`, `world.UnloadArea(area)` | Save and unload; the error names what still holds it (a player, another plugin's ticket) |
| `server.LoadWorld(name, opts)`, `server.UnloadWorld(name)` | Load or unload a world at runtime (RFC 0002) |
| `server.CreateWorld(name, opts)` | New world with a chosen generator |
| `server.OfflinePlayer(uuid).Load()` | Read and change a player who is not online; saved on `Save()` |

An **area** (`helix.Area`) is a group of chunks picked out at once by
describing a shape. Helix works out which chunks are inside it. Nothing is
stored in the world; it only says "these chunks", so one call works for any
size instead of separate functions per shape.

```go
helix.Chunk(10, 4)          // one chunk
helix.Box(-64, -64, 64, 64) // every chunk in a rectangle between two corners (block coordinates)
helix.Radius(pos, 100)      // every chunk within 100 blocks of a point
```

| Thing | Decided by | Size |
|---|---|---|
| Chunk | Minecraft | Fixed: 16 × 16 blocks, full height |
| Area | The plugin | Any shape it describes: one chunk, a box, a radius |
| Region | Helix, automatically | Changes as players move; merges and splits |

Ticket options:
`helix.Ticking(false)` (loaded but frozen, for lobbies and arenas between
games) and `helix.LoadPriority(n)`.

| Advanced | What it does |
|---|---|
| `world.Regions()` | Snapshot of the current regions: sections, owners, tick times |
| `world.KeepTogether(area) *Ticket` | Force an area into one region, so everything in it interacts in the same instant (a minigame arena) |
| `world.KeepApart(area) *Ticket` | Ask that an area never merges with its neighbours |
| `world.SetRegionizer(r)` | Replace how sections are grouped into regions |
| `world.SetChunkSource(s)` | Replace where chunks come from (storage, generator, memory only) |
| `world.SetChunkSink(s)` | Replace where chunks are saved, or discard them (`helix.NoSave`) |

Unloading never removes a chunk that holds a player. Plugins move players
first with `p.Teleport`, which always returns a `Result`.

### 8. Layers

The same rules apply at every layer of `API_DESIGN.md`: game objects, packet
hooks and raw byte hooks run on the owner of the player they belong to, and
their doc comments carry the same tags.

## Example

```go
type Arena struct {
    helix.Plugin
    ticket *helix.Ticket
}

func (a *Arena) Enable() {
    w, ok := a.Server().World("arena")
    if !ok {
        return
    }
    a.ticket = w.KeepTogether(helix.Box(-64, -64, 64, 64))
}

func (a *Arena) start(players []*helix.Player, spawn helix.Pos) {
    for _, p := range players {
        p.Teleport(spawn).Then(func(err error) {
            if err != nil {
                p.Message("Could not join the arena.")
            }
        })
    }
}
```

Disabling the plugin releases the ticket automatically.

## Open questions

- An advanced escape hatch that pauses two regions for one call
  (`server.Together(a, b, fn)`). Not built until someone needs it.
- Which values snapshots carry, and their cost.
- Offline player data format (topic 9, storage).
- Exact names (`EachRegion`, `KeepLoaded`, `Ticket`) before code is merged.

## Alternatives considered

- **Handle valid during the call** (`world.Tx`): safer against keeping
  objects, but wordier and no faster. Rejected for decision 1.
- **Silently forwarding wrong-place calls:** easier at first, but hides when
  work happens later. Rejected for decision 3.
- **Locking other regions** (Minestom's `Acquirable`): deadlocks, and slow
  regions slow down others. Rejected.
