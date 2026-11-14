# API design

**Status: core design decided, building has started.** Accepted: plugins and
listeners ([RFC 0001](rfcs/0001-plugins.md)), core and threads
([RFC 0002](rfcs/0002-core.md)), and the engine design decided by the project
lead on 2026-10-29 ([Engine design](#engine-design-decided-2026-10-29) below:
registry, world in memory, regions, players and entities, events, storage).
Engine code may be written for those topics, in the build order given there.

Topics still marked "Not started" in the table at the end are not designed
yet. Nobody (human or AI) writes public interfaces for them until this file
says they are decided. Prototypes are fine in a folder named `experiments/`
that is never merged.

## Goals (agreed)

These are fixed. Every RFC must show how it meets them.

**Helix is an engine for any kind of server.** A survival SMP turns on all of
vanilla and it works. A Skyblock or minigame server turns vanilla off, keeps
only the fast core, and hard-codes its own rules. Both are first-class.

1. **Expose everything.** Every part of the engine, including the fundamentals
   (tick loop, chunk storage, physics, movement validation, damage, inventories,
   packet handling, entity spawning), is reachable and replaceable through the
   API. Nothing is internal-only by default; hiding something needs a written
   reason in the RFC.
2. **Vanilla is just modules.** Vanilla content uses only the public API, with
   no private shortcuts. If vanilla can build it, anyone can build something
   different.
3. **Replaceable layers.** Core (tick, network, chunks, players), then systems
   (physics, AI, pathfinding, inventories, world gen, storage), then vanilla
   content, then the server's own code. Each piece is an interface with a
   default implementation that can be swapped on its own.
4. **Presets.** At minimum `vanilla` (everything on, for SMP) and `minimal`
   (core only, for Skyblock and minigames). Servers can start from either and
   add or remove modules.
5. **Helix is a Go library first.** Servers can import Helix into their own Go
   program and configure or replace anything in code, at full speed. WASM
   plugins are a sandboxed layer on top for drop-in extensions, and expose as
   much as safely possible.
6. **Pay only for what you use.** A disabled module or an event with no
   listeners costs nothing at runtime. Hot paths (movement, physics, chunk
   ticking) are customised by swapping implementations, not by events.
7. **Two API tiers.** A **stable** tier with a semantic-versioning promise, and
   an **advanced** tier that exposes the deep internals, clearly marked as
   allowed to change between minor versions. Everything is in one tier or the
   other; neither tier is hidden.
8. **Multi-core by design.** No part of the engine is limited to one core, and
   the server owner chooses how many cores to use, from one to all of them,
   overall and per system (for example ticking, world generation, network).
   - Parallel work (chunk generation, lighting, saving, compression, packet
     encoding) runs on worker pools.
   - Each world and dimension ticks independently.
   - Large worlds are split into regions that tick in parallel when they cannot
     affect each other within a tick.
   - Mob AI thinks in parallel (sensors, pathfinding, decisions), then applies
     its results to the world in a fixed order.
   - Results must be identical on 1 core and on many: order-sensitive systems
     such as redstone are deterministic within a region.
   - Every API call documents where it may run (inside a world or region tick,
     from any goroutine, or scheduled onto the owner of a chunk or entity). A
     debug mode reports calls made from the wrong place.
   - `cores: 1` must remain a fully supported mode for debugging and exact
     reproduction.

## Engine design (decided 2026-10-29)

Decided by the project lead in design discussions on 2026-10-29. The engine
was restarted from scratch in `helix` the same day; the old Dragonfly-based
engine is not part of it. Where this section and an older RFC disagree, this
section wins.

### Rule zero: one-to-one with BDS

- **Vanilla behaviour is one-to-one with BDS** (currently 1.26.60), not
  "close enough": world generation, redstone (required), every other
  vanilla mechanic, and the world storage format. Helix may only be lenient
  where the project lead says so, such as Helix's own APIs and internals
  that do not change any result.
- Where regions or parallel work could change a vanilla result (redstone
  update order, scheduled ticks, hoppers), Helix follows BDS's order inside
  a region. This replaces RFC 0002's "the tick order is Helix's own".
- **No felt lag, even with 400 players in one spot.** Every design choice is
  checked against that case and proven with `helix-bench`: 400 fake players
  in one spot for 10 minutes, no tick over 50 ms, 99% of ticks well under,
  run on every engine change.

### 1. Registry

- **One `Setup` builder at boot.** Vanilla, built-in features and plugins all
  register through it; there is no back door.
- **Frozen at boot:** everything the client is told once when it joins
  (blocks and their states, items, biomes, entity types). Transports open
  only after the registry is frozen.
- **Changeable at runtime:** recipes, creative inventory and commands
  (re-sent to online players; to be confirmed against BDS per packet), and
  server-only data (loot, trades, spawn rules).
- Registering an existing name overrides it, and the log says which plugin
  replaced what. Plugins may **delete** vanilla content at boot; saved data
  that still holds it loads a replacement (blocks default to
  `minecraft:unknown`, as BDS; biomes default to plains), and world
  generation places the replacement.
- **Blocks:** every block state is a dense `uint32` ID. Properties
  (hardness, friction, light, ...) live in one flat table per state;
  network hashes are computed once. Behaviour is a set of small optional
  hooks, resolved at boot so unused hooks cost nothing.
- **Live changes:** behaviour and properties of any type, vanilla included,
  can be changed while running. Tables are copied, edited and swapped
  atomically; each region picks up the change at the start of its next tick.
  Plugins can **replace** a behaviour or **wrap** it (run code before or
  after vanilla's); wraps stack in plugin load order.
- **Hooks cover everything BDS blocks and items can do** (its C++ block and
  item hooks, JSON components and Script API events): placing, breaking,
  using, ticking, neighbours, entities inside or on a block, redstone,
  pistons and falling, liquids and waterlogging, shapes, block entities,
  multi-block parts, item use, use-on, charging and releasing, eating,
  mining speed, hitting, projectiles, dispensers, durability and inventory
  ticks. Numbers are properties, actions are behaviour.
- **Item stacks:** a plain stack is about 8 bytes with no allocation; names,
  lore, enchantments, NBT and **namespaced plugin data** (saved with the
  item, never sent to the client) live in an optional extra part.
- **Biomes:** frozen at boot, custom biomes allowed. **Entity types:** name,
  definition (components, groups, events, parsed by the `entity` library)
  and Helix's engine hooks (spawn, hurt, death and drops, interact, tick).
  AI and components stay in the `entity` library. The player is the
  `minecraft:player` entity type.

### 2. World in memory

- Chunks use the `world` library's format (paletted sub-chunks, two block
  layers, half-byte light with shared empty and full arrays), improved in the
  library: the registry's tables replace the old name-and-property lookups,
  and light is server-only (clients light the world themselves).
- **Two zones per player.** The simulation zone (BDS `tick-distance`,
  default 4 chunks) is fully loaded, owned by a region and ticked. The view
  zone (BDS `view-distance`) is loaded or generated on a worker pool,
  encoded, sent, then **released**; an optional compressed cache with a
  memory budget keeps busy areas ready. Memory follows the simulated area,
  not world size or view distance.
- **Client-side chunk generation** is on by default for vanilla worlds, as in
  BDS: the server builds the inner part of the view and the client draws
  vanilla terrain beyond it. Measured on BDS: about 3.3 times fewer chunk
  packets. Off automatically for custom generators.
- A chunk always has exactly one owning region and moves between regions by
  pointer. Loading and saving run on worker pools; a region never waits.
  Saving copies only changed sub-chunks at the end of a tick. Each chunk
  keeps its encoded network form and shares it with every player.

### 3. Regions and ticking

RFC 0002 stands (regions that merge and split, per-region 20 TPS on a worker
pool, global owner, async teleports), with these additions:

- Regions are built from simulation zones only, so spread-out players give
  many small regions.
- **Crowded regions work in parallel inside the tick:** a per-player phase
  runs every player's own input (movement, inventory, UI) at once, and mob
  AI thinks in parallel; anything that changes the shared world is applied
  afterwards in a fixed order, so results are the same on any core count.
- **Encode once, send to many:** each entity update is built once per tick
  and the same bytes go to every viewer. Compression and encryption run off
  the region.
- Entities that leave a region during a tick are handed to the new owner at
  the end of the tick. Chunk loads never block: an entity stops at an
  unloaded chunk.
- Packets are flushed at the end of each tick, not on a timer. Near the tick
  budget, non-gameplay work (chunk sending, cosmetic packets) slows first;
  anything that changes gameplay is only reduced when the owner turns it on.
- Tiles inside a region are designed for but built only if the benchmark
  shows crowds still need them.

### 4. Players and entities

- **A player is three parts:** the session (connection, identity, send
  queue, what the client has; safe from anywhere), the entity (position,
  health, inventory, game mode; owned by the region it stands in) and the
  profile (permissions and server-wide facts; owned by the global owner,
  read through a published copy).
- Every entity belongs to its chunk's region, which keeps an entity list and
  a per-chunk grid for nearby queries. Helix implements the `entity`
  library's world and mob interfaces per region; each region has its own
  seeded random so results do not depend on core count.
- **Visibility and join follow BDS:** entities are shown within BDS's
  57-chunk disc around the player; the join sends BDS's packet sequence in
  BDS's order, with the large registry packets built once at boot and shared.
- **Anti-cheat is optional:** Helix's own anti-cheat module, off by default,
  switchable. BDS's default position correction is separate from it.

### 5. Events

RFC 0001 stands, with these additions:

- **Priorities are named by when they run:** `First`, `Early`, `Normal`,
  `Late`, `Last`, then `Monitor` (read-only, after everything). The last
  listener to run has the final say.
- Almost every event can be cancelled and changed.
- **No allocations:** an event with no listeners is never built. Each worker
  reuses its own event structs (nested events use a small stack per type),
  so firing allocates nothing after warm-up. Listeners must not keep an event
  after they return. Dispatch benchmarks must show 0 allocations; player
  movement must handle about 50,000 events a second.
- A hook and its events run in order: "before" event (can cancel or change),
  the type's hook, then "after" event (read-only).
- **Listeners run in parallel by default** where the event fires in a
  parallel phase; `helix.Sequential()` runs a listener one call at a time.
  Plugins get per-player data that needs no locks.
- Packet hooks run on the player's network goroutine and never touch the
  world.

Chests, ender chests and drops (lead asked 2026-11-02, built in `player`): events fire before
anything happens and may cancel or change it (`OnChestOpen`, `OnChestItem`, `OnChestPair`,
`OnChestSpill`, `OnBlockBreak`, `OnBlockDrops`, `OnExperience`). `OnChestItem` fires per slot once
a request has passed BDS's checks, and cancelling it refuses the whole request, because one request
also changes the player's slots and refusing part of it would lose or double items. Methods
(`Chest.SetItem` and so on) fire no events, so a listener can call them without looping.
Crafting (2026-11-02, built in `crafting` and `player`) works the same way: `OnCraft` fires once a
request's recipe passes BDS's checks, before any item is used, and may change the results or refuse
the request. Recipes are the `Registry.Crafting` packet, so changing it changes what can be crafted.
Smithing recipes (lead's design, 2026-11-02): every vanilla recipe is there by default, and a plugin
may clear them all, remove one, remove those for an item, or change one, at boot or while running
(`EditSmithing`, `Smithing`). `OnSmith` changes or refuses one use of the table. Every other recipe
works the same way through `EditRecipes` and `Recipes`, whichever block makes it.
`Player.EditRecipes` takes recipes away from one player only; the server refuses what that player
does not have. The rules last until the player leaves.

### 6. Storage

- **Superseded 2026-11-13:** worlds are stored in Helix's own format, and BDS
  worlds are converted in and out losslessly. See "World data redesign",
  sections 4 to 6. (Was: one LevelDB per world, exactly like BDS.)
- Player records keep BDS's content: `player_server_<uuid>` holds the data,
  `player_<id>` records map each login ID to it.
- **Storage is pluggable.** The world store and the player store are separate
  interfaces (both default to the Helix format), so a network can keep
  players in one shared database.
- Store calls never block a region. A chunk and its entities and block
  entities are written in one batch. Bad data is logged, never a panic.
- Each plugin gets its own namespaced key-value store in the same backend.

### Build order

From the ground up, each step tested before the next:

1. Storage in the `world` library: the full BDS 1.26.60 key inventory, every
   key one-to-one, the pluggable store interfaces with LevelDB as default.
   Done 2026-10-29: two worlds BDS 1.26.60 saved round-trip byte for byte;
   `KV`, `WorldStore` and `PlayerStore` in `world/mcdb`. Not yet checked in
   game.
2. Registry in `helix`, blocks first. Blocks done 2026-10-29: `Setup`
   (register, replace, delete), the frozen table with network hashes and
   a live-swappable property table read from BDS 1.26.60 at runtime
   (helix-data `bds/block_properties.json`). Two BDS worlds still
   round-trip byte for byte with it, heightmaps included. Items done
   2026-10-30: BDS's 2,080 items with the network IDs, versions and
   components it sends at join (captured from a real join, helix-data
   `bds/packets/`). Biomes, entity types, creative inventory and recipes
   done 2026-10-30: biome IDs read from the running BDS, every registry
   packet byte for byte BDS's when untouched and rebuilt when a plugin
   changes it. Left for later steps: block and item behaviour hooks
   (with the world), entity engine hooks (with entities), what saved
   stacks of deleted items load as (with item stacks), and whether BDS
   re-sends creative and recipe packets the way Helix does.
3. World in memory: chunks, loading and saving pools, both zones.
   First pass done 2026-10-30 (`helix/world`): chunks owned by one tick
   goroutine per dimension until regions take over, a worker pool for
   loads, generation, encoding and saves, BDS's sub-chunk request mode with
   block hashes, encoded chunks shared and cached, changed chunks saved at
   the end of each tick from copy-on-write snapshots, flat and void
   generators. Left for later: hooking up vanilla generation (then
   client-side generation), the blob cache, block entities in chunks,
   server-side light, and a BDS capture to check LevelChunk's sub-chunk
   count and limit and the SubChunk height maps byte for byte.
4. Regions and ticking. First pass done 2026-10-30 (`helix/world`):
   regions of 8 by 8 chunk sections, each owning its chunks and players
   and ticking at 20 TPS on a shared scheduler (earliest deadline first,
   `TickCores` goroutines), merging and splitting between ticks only;
   a region that must merge with one still ticking parks until it ends.
   `RunAt` runs code on a chunk's owner, loading it. Left for later: the
   global owner (with the player list, step 5), the tick phases that need
   entities, the over-budget
   event (a log warning for now) and the rest of RFC 0003. Section size
   and merge distance chosen 2026-10-30 with an in-process benchmark of
   400 fake players (`BenchmarkRegionLayout`, until `helix-bench` exists):
   8 by 8 chunks and 1 section, the defaults; to be measured again once
   gameplay adds work to the tick. Entity
   hand-off done 2026-10-30: entities other than players stay in loaded
   chunks, pass to the region of the chunk they move into at the end of
   the tick, and unload with their chunk. Block ticks done 2026-10-30:
   random ticks and both of BDS's tick queues per chunk, saved with it;
   block behaviour comes through a `BlockTicker`. Chunk tickets done
   2026-10-30: `KeepLoaded(area, ticking)` keeps a box, radius or list of
   chunks loaded, ticking or frozen, until released. Still open from RFC
   0003: load priority, `UnloadChunk`, `KeepTogether` and `KeepApart`.
   Teleports done 2026-10-30: a player moving more than `MergeDistance`
   sections from its region goes straight to the region there, with its
   entity, instead of merging the two.
5. Players and entities, replacing the temporary `play.go`. First pass
   done 2026-10-30: `Player` (session, entity, saved data), the global
   owner holding the player list, and `world.Entity` (IDs, position,
   rotation, owned by its region). Players see each other within BDS's
   57-chunk disc (AddPlayer, RemoveActor), get each other's movement as
   MovePlayer built once per tick, and are saved on leave into their BDS
   data. Left for later: inventory, abilities and game modes, AddPlayer's
   synched data checked against BDS, BDS's unique IDs for new players,
   the rest of the join packets, mobs and the `entity` library, encoding
   a packet once for every connection, and moving teleports.

### Commands (decided 2026-11-05)

- **No namespace prefix.** Commands are internal: a plugin's `home` is
  `/home`, never `plugin:home`. Vanilla and plugin commands are equal and
  go through the same API; registering a taken name replaces it, and the
  log says which owner replaced which.
- **Two ways to declare, one result.** Typed structs, one per overload,
  whose fields are the arguments and whose `Run` method runs it (the main
  way), and a `Builder` for commands made while running, such as from a
  config file.
- **Suggestions as BDS's client shows them.** Fixed enums, soft enums
  changed live, per-player commands and options (players get only what they
  may use), BDS's own lists prebuilt with constants (Item and Block with
  icons, entity types, effects, game rules and the rest), and plugin enum
  types.
- Also: middleware around every command, BDS's error messages and
  permission levels, commands added or removed while running.
- Every player's state can be read from any region (`Player.State`), so
  selectors see players everywhere.

### Still open

The full event list, custom events, the entity visibility API (leaning
towards `CanSee`, `HideActor`, `ShowActor`), permissions beyond BDS's
command levels, the server config file, and BDS's autosave timing.

## World data redesign (DECIDED 2026-11-13, building)

**Status: decided by the project lead on 2026-11-13 as a full rewrite, not a
patch of the inherited model, and the top priority.** It includes Helix's own
world format on disk (section 4) with built-in backups (section 5) and a
converter library to and from BDS worlds (section 6). It replaces parts of
"World in memory" (2), "Storage" (6) and RFC 0003's load priority. The
targets and the open questions at the end are still to be settled. Research, code references and
sources: `memory/research-storage-redesign.md` (workspace, not in a repo).

### Why

A test world built with `/chestbomb` (1,020 dropped shulker boxes of books at
1 MB each, up to 223 per chunk; chests of 300 MB per chunk) showed that Helix
assumes world data is small:

- **A chunk is all or nothing.** Every load reads terrain, block entities, tick
  queues and every entity record, decoded into NBT maps, before the client gets
  anything, even for view-zone chunks that are then thrown away. The client
  never needed entities in the chunk packet.
- **One queue for everything.** Loads, generation, encodes, saves and cold
  chest reads share one FIFO per worker: no priority, no cancelling, and one
  huge save blocks every chunk behind it.
- **One lock between saving and loading.** A flush holds a lock for as long as
  it writes, and every chunk load waits for it.
- **Data is copied and re-encoded at every layer.** Entities and block entities
  live as pointer-rich NBT maps; every save encodes every sub chunk and every
  entity again, changed or not.
- **Limits are scattered.** Load gate, cache budget, save delay and item caps
  each have their own number; nothing knows the total memory or I/O in flight.

Paper's chunk system (Moonrise), Folia, vanilla Java since 1.17, Valence and
Pumpkin all separate terrain from entities, read in stages, and schedule work
in prioritised, cancellable queues with rate limits. Helix should too.

### Targets

With the hostile world (below) and 20 bots joining at once:

- A player joins at once: spawn within 3 seconds, the first chunk within 1
  second, the nearest three rings within 2 seconds.
- Memory does not follow world contents: peak heap at most 300 MB above the
  same scenario on a plain world.
- No tick over 50 ms is caused by loading or saving, and no chunk load ever
  waits for a save or flush of another chunk.
- Nothing is lost: every entity and block entity round-trips through save and
  reload, and two BDS worlds still round-trip byte for byte.

The numbers are first guesses, to be set by the lead after step 0 measures
today's code.

### 1. Data model: shells with lazy payloads

A loaded chunk is a set of **parts**, each with its own state and memory cost:

| Part | Holds | Form |
| --- | --- | --- |
| Terrain | sub chunks, biomes, height map, light | flat arrays, no pointers (as today) |
| Network | encoded chunk packet, per sub chunk | bytes, shared, in a budgeted cache |
| Block entity shells | position, type, the few fields used every tick (a chest's slot counts for comparators, a furnace's state) | small structs |
| Block entity payloads | everything else | raw NBT bytes in memory, or on disk only |
| Entity shells | unique ID, type, position, rotation, box, what visibility and saving need; for an item, its stack header | small structs |
| Entity payloads | the rest of the record (inventories, nested items, books) | raw NBT bytes in memory, or on disk only |
| Tick queues | scheduled and random ticks | as today |

- **Raw by default.** A record is kept as the bytes it was stored as (one
  pointer-free slice) and decoded only where the engine reads it. An entity
  type Helix does not run is never decoded at all and saves by copying its
  bytes. This generalises `nbt.Raw` nested items and cold chests to every
  entity and block entity.
- **Payloads can go cold** (`ColdBlockEntity` for all, and for entities): a
  payload unused for a while and stored current is dropped from memory and
  read back on the load worker when needed, never on the tick.
- **Not an ECS, for now.** The `entity` library, plugin API and regions are
  built on entity values; shells give the flat, scannable storage that is most
  of an ECS's benefit. Regions may later keep shells in per-type arrays behind
  the same API if the benchmarks ask for it.

### 2. Staged chunk loading, terrain first

Each chunk moves through states, owned by its region:

1. **Terrain**: terrain part read and decoded, light worked out, network form
   built and sent. This is all the view zone ever loads. Block entity network
   data (signs, chest pairing, banners) comes from the shells.
2. **Blocks ready**: block entity shells and tick queues loaded.
3. **Full**: entity shells loaded and live entities built. Only a Full chunk
   ticks, so no rule ever runs with a chunk's entities missing (vanilla Java's
   MC-211601: pressure plates turned off because entities came after ticks).
4. **Payloads**: read one by one when used (a chest opened, an entity hurt),
   never as a whole.

- The simulation zone asks for Full, the view zone for Terrain. A chunk moving
  from view to simulation zone only reads what it lacks.
- An entity walking into a chunk whose entities are not loaded yet joins a
  "transient" list, merged with the stored ones when they load or when the
  chunk saves (as Moonrise does).
- Building live entities from shells is time-budgeted per region tick (for
  example 2 ms); the rest waits for the next tick, and the chunk counts as
  Full only when done.
- A chunk's entity and block entity shells are one Objects record (section
  4), read in one go; payloads stay on disk until used.

### 3. Work queues and priorities

- **Two pools.** An I/O pool (a few goroutines, only store reads and writes)
  and a CPU pool (decode, encode, generate, light, compress).
  Region ticks stay on the tick scheduler. No pool ever waits on a tick, and
  no tick ever waits on a pool.
- **One task record per chunk and part**, not per worker: at most one pending
  read and one pending write each. Writes before the I/O runs collapse to the
  newest snapshot; a read that finds a pending write uses its data; a task
  nobody wants any more is cancelled. Order per chunk comes from this record,
  so jobs no longer have to be pinned to one worker.
- **Priorities** (highest first), recomputed when a player moves:
  1. what a join or a teleport waits on (the player's own chunk and sim zone);
  2. view-zone terrain, nearest ring first;
  3. payload reads a player waits on (opening a chest);
  4. saves of unloading chunks (memory is freed only after them);
  5. auto-saves, cooling, compaction.
- **Fair shares.** Each queue gets a time slice, as Moonrise's balanced pool
  does, so saves always make progress and loads are never starved by them.
- **Per-player limits**, as Paper: concurrent loads (default a fifth of the
  view area, at least 5), loads and sends per second with short bursts
  allowed. One player flying fast cannot take the pools from the others.

### 4. The Helix world format (decided 2026-11-13)

**Decided by the project lead on 2026-11-13:** Helix stops storing worlds in
BDS's LevelDB and uses its own format, `.hx` (for Helix). BDS worlds are
converted in and out by a converter library (section 6). One-to-one with BDS
now means: every BDS world converts to Helix, every Helix world converts to
BDS, and a world converted there and back without being played comes back
identical. Helix's own LevelDB engine (`world/lsm`, built first) stays as the
converter's BDS reader and writer.

The lead's rules for it:

1. **Reads and writes are as fast as the disk allows, with no allocations** in
   the steady state.
2. **Size never slows it down.** A 100 GB or 1 TB world reads and writes as
   fast as a small one, and the server starts as fast.
3. **Backups are built in and live:** made while the server runs, without
   stopping anything, and usable by plugins (section 5).
4. **A converter library**, BDS to Helix and Helix to BDS, seamless. Helix
   gets a live converter on top of it later (section 6).

Byte-level layouts go in `world/hx/FORMAT.md`, written with the code.

**Files on disk:**

```
worlds/<name>/
  level.dat            BDS's own file, unchanged
  world.hx             format version, settings
  blocks.hx            the world's block table
  journal/             save commits not yet checkpointed
  overworld/r.<x>.<z>.hx   one region file per 32×32 chunks
  nether/  end/  (and other dimensions)
  players.hx           every player
  level.hx             maps, scoreboard, villages, portals, structures,
                       ticking areas and every other world-wide BDS key
```

**The record store** (`world/hx/store`). Every `.hx` file is a record store:
records plus an index of where each one is. One engine serves regions,
players, level data and the block table.

- **A record** has a header (type, compression, key, the save round that wrote
  it, length), its body and a CRC-32C checksum. Records describe themselves,
  so a lost index can be rebuilt by scanning the file.
- **Saved data is never written over.** A new version goes to free space; the
  old one is reused only after the commit that replaced it, and only once no
  reader or backup can still see it (epochs).
- **The index** is flat, pointer-free sorted arrays in memory. On disk it is one
  index record plus two superblock slots (round number and checksum), written
  alternately at checkpoints, so a torn write can never lose both.
- **64-bit offsets and lengths everywhere:** no limit on a record, a chunk or a
  file. Java's Anvil caps a chunk at 1 MB.
- **Keys.** In a region: the chunk within the region, the part, a payload
  number. In `players.hx` and `level.hx`: the BDS key name.
- **Compression per record:** zstd from `klauspost/compress` (already a
  dependency), with an ID byte so more (zstd dictionaries) can be added later
  without converting worlds. A stored record can be copied without being
  decompressed (backups, the converter, an entity moving to another region).
- **Unknown record types are kept and copied, never dropped.** Every file
  starts with a magic number and a format version.

**Saving: rounds and the journal.** Changes are tracked per part. A save round
runs every 5 seconds (as BDS flushes), off the tick:

1. Encode the changed parts on the CPU pool.
2. Write them into free space.
3. Force them to disk.
4. Append one commit to the journal (every new location, with a checksum) and
   force it to disk. This is the moment the round counts.
5. Later, a checkpoint writes the new locations into each file's index and the
   journal is trimmed.

- **A round lands whole or not at all,** across every file: an item moved from
  a chest to a player's inventory is saved in both or neither, so a crash
  cannot duplicate items.
- **Bytes written are about the bytes changed** (around 1.0 to 1.1 times), with
  no compaction. A file that ends up mostly free space is repacked on its own,
  rarely.
- **The journal is bounded** (checkpoints by size and time), so starting a
  world reads at most that much, whatever the world's size.
- **Forcing to disk is on by default** (survives power cuts). Off, a world
  survives a crash of the server but not of the machine, like BDS's default.

**Reading.**

- An index lookup, then one positioned read into the caller's buffer. Any
  number of reads run in parallel, with no lock and no wait on a save: a
  reader takes the current index and epoch in one step.
- Each read sees one finished save round, never half of one.
- A chunk that does not exist costs no read at all.
- No block cache and no in-memory table: the operating system's file cache
  does caching, outside the Go heap. The store's own memory is a few MB: an
  index cache with a byte budget and a bounded set of open files.

**Chunk parts.** A chunk is stored as parts, each its own record, read and
written on its own, written next to each other so a whole chunk is usually one
or two reads:

| Part | Holds | Read when |
| --- | --- | --- |
| Terrain | sub chunks, biomes, height map, light (planned), and every small BDS chunk key (version, finalisation, border blocks, blending, structure starts...) byte for byte | the chunk comes into view |
| Objects | every block entity and entity shell, and the scheduled and random block ticks | the chunk enters the simulation zone |
| Payloads | the big fields cut out of shells, one record each | when used: a chest opened, an item picked up, an export |

(Built 2026-11-13: Terrain, in `world/hx`, `terrain.go` and `chunk.go`. The
small chunk keys sit in Terrain instead of a part of their own: they are a
few bytes each and a separate record would cost a read for nothing.)

- **Shells are cut by size, not by type:** a block entity or entity record's
  fields over a threshold (to start: 4 KB) move into payload records, largest
  first. A normal chest stays one small shell. No per-type code, so new mobs,
  new blocks and plugin data work too.
- **Terrain encoding:** the packed block words are the same bytes BDS and the
  client use. The palette is numbers into the world's block table, not NBT
  names: loading is a number lookup, sending is a swap to network hashes. Light
  is stored as a cache (missing or outdated: worked out again).
- **The block table** (`blocks.hx`) holds every block state the world uses,
  once, with the exact bytes BDS wrote for it, so an export writes BDS's
  palette entries back unchanged. A Minecraft update that renames a block
  fixes its row once at start-up, not every palette on every load.
- **NBT stays where it fits:** entities, block entities, players and level
  data are little-endian NBT as in BDS, plus zstd. Their data is open-ended
  (every mob differs, plugins add fields), every Bedrock tool reads NBT, and
  conversion is a copy. Terrain is the one place a custom layout wins.

**Speed and allocations (lead's rule 1).**

- The store moves bytes; it never decodes NBT. Reads fill a buffer the caller
  owns; zstd decoders and encoders are pooled and write into reused buffers;
  index lookups are binary searches over flat arrays; save rounds reuse their
  buffers.
- Benchmarks with `b.ReportAllocs()` must show 0 allocations per operation for
  index lookups, warm part reads, terrain decode, and a save round in the
  steady state.
- Estimate, to be measured: a normal chunk loads in 0.1 to 0.4 ms of CPU on one
  core; an NVMe disk serves tens of thousands of chunk parts a second.
- Storage never runs on the tick. A slow disk makes chunks arrive later; it
  never makes a tick late.

**Size (lead's rule 2).**

- Finding a chunk is the same work in any world: the region file follows from
  the coordinates, then one index search and one read.
- Memory does not grow with the world: the index cache and open files are
  bounded.
- No compaction: a 1 TB LevelDB keeps rewriting its data in the background;
  this format writes each change about once.
- Start-up reads the bounded journal, never the world. A 1 TB world is tens of
  thousands of region files, opened on demand.

### 5. Backups (decided 2026-11-13)

- **Live and built in.** A backup pins the current saved state (an epoch):
  records it can see are not reused until the copy is done, while the server
  keeps playing and saving into other free space. The copy is paced so it never
  slows loading. The only cost: files may grow a little while it runs.
- **API in `world/hx`** for Helix and plugins: back up to a target (a folder, an
  archive, or a plugin's own writer, such as cloud storage), incremental since
  an earlier backup, with progress and a rate limit. Restore a whole world
  (offline), restore chunks or an area into a running world, and open a backup
  read-only as a world (for rollback tools).
- **Incremental:** every record carries the save round that wrote it, so a
  backup copies only what changed since the last one.
- **A backup is itself a compact Helix world** (or a delta on top of one).
  Records are copied as stored, with their checksums, never re-encoded.
- **In Helix:** the plugin API and events, a `backup` command, and BDS's
  `save hold`, `save query` and `save resume`, so BDS backup tools keep working.

### 6. The converter library (decided 2026-11-13)

`world/convert`, BDS to Helix and Helix to BDS:

- **Whole worlds and single pieces.** Functions convert a whole world, and also
  one chunk, one player or one level record, so Helix's later live converter
  (converting chunks as they load) uses the same code.
- **BDS side:** `world/lsm` for LevelDB, the BDS key and value knowledge of
  `world/mcdb`.
- **Lossless.** Every BDS key becomes a Helix record or is kept with its exact
  bytes (Extra, `level.dat`, `level.hx`). A world converted there and back
  without being played comes back identical; chunks that were played on come
  back in the current BDS format, as BDS writes them.
- **Streams, in parallel per region,** never holding a world in memory, with
  progress and an option to convert back in memory and compare.
- **Tested** on real BDS worlds (identical round trip) and by running BDS on
  converted worlds.

### 7. Saves that do not encode unchanged data

- **Only changed parts are written.** A chunk whose entities moved writes its
  Objects part, not its terrain.
- **Per sub chunk:** a dirty bit; the terrain part is rebuilt from the stored
  bytes of unchanged sub chunks plus the changed ones encoded. The network form
  is also rebuilt per changed sub chunk.
- **Per entity and block entity:** a dirty bit; an unchanged shell is copied as
  stored bytes, and a payload is never rewritten unless it changed. Mobs that
  move save on the save round; item frames, armour stands and dropped shulker
  boxes that sit still are never rewritten.
- What is stored after a save is exactly what a full re-encode would write:
  skipping a write never changes what a load or an export reads.

### 8. Limits

One `Limits` part of the server config, Helix's own beyond BDS, with defaults
that normal play never reaches:

- **Size:** what one shulker box or bundle may hold (2 MB, today), what one
  entity or block entity record may hold when items are added to it, book size
  as BDS allows. Limits refuse what a player adds; they never drop data that
  is already stored.
- **Count, optional:** per entity type, how many a chunk saves (Paper's
  `entity-per-chunk-save-limit`), off by default; item merging as BDS does it.
- **Memory:** one memory governor with accounts (network cache, warm payloads,
  pending writes, loads in flight). Over budget it evicts caches first, then
  cools payloads, then slows new loads (back-pressure); it never stalls a tick.
  It replaces the load gate, the cache budget and the save-cost delay.
- **Time:** per region tick, a budget for arrivals and warmed payloads.
- **I/O:** bytes in flight and bytes per second, separately for foreground
  (loads, payload reads) and background (saves, backups, repacks).
- **Network:** how many block entities one chunk packet carries before the
  rest are sent as separate block entity packets (Paper uses 750); to be
  checked against what the Bedrock client accepts.

### 9. The hostile-world test (the gate for every step)

A `sim` scenario, `hostile`, that builds the worst case in game and on disk at
a size set by a flag (kept small by default; full size only when the lead
asks, under the 10 GB cap):

- chunks of chests stuffed with books (up to 300 MB a chunk), dropped shulker
  boxes of books (up to 223 a chunk), thousands of mobs, item frames and armour
  stands, pistons moving full chests;
- the server is stopped and started again so everything loads from disk;
- 20 bots join at the centre at once, then walk and fly outwards.

It reports join time (spawn, first chunk, nearest rings), heap and collector
share over time, the longest tick and late ticks, load latency per ring and
per priority, time any load spent waiting on another chunk's save or a flush,
bytes read and written, and a check that every entity and block entity is
still there after the restart. Every step below must meet the targets on this
test, must not make the plain-world scenarios worse, must keep two BDS worlds
round-tripping byte for byte through the converter, and needs the lead's
in-game test.

### Migration, in order

0. **Measure.** Build the `hostile` scenario and its report; record today's
   numbers. Set the targets from them.
1. **BDS LevelDB engine** (`world/lsm`): done 2026-11-13; now the converter's
   BDS side.
2. **Record store** (`world/hx/store`): records, index, journal, save rounds,
   recovery, epochs; kill and power-cut tests; zero-allocation benchmarks.
3. **Helix world format** (`world/hx`): chunk parts, block table, players,
   level data, the world provider Helix uses.
4. **Converter library** (`world/convert`): both directions; real BDS worlds
   round-trip identically; BDS opens converted worlds.
5. **Backups** (`world/hx`, then the Helix API and commands).
6. **Queues** (`helix/world`): I/O and CPU pools, task records per chunk and
   part, priorities, fair shares, cancelling, per-player limits.
7. **Staged loading:** Terrain / Blocks ready / Full; the view zone reads
   terrain only; arrival budget per tick; transient entity lists.
8. **Shells and raw payloads** (`helix/world`, `player`): entity and block
   entity records kept raw, shells decoded, payloads cold.
9. **Saves without re-encoding.**
10. **Limits and the memory governor**, replacing the scattered limits.

Steps 2 to 5 are in order; 6 can be built alongside them; 7 needs 3 and 6; 8
and 9 need 7.

### Open questions for the lead

- Does BDS 1.26.60 tick a chunk before its actors are loaded, or load them
  with the chunk? (Needs an IDA check; it decides whether "Full" waits for
  entities exactly as BDS does.)
- Are the targets above right?
- Optional per-type entity save limits: offered, off by default?

## API style (direction)

Helix has its own API style. It is not a copy of Bedrock's Script API or of
Dragonfly.

- **Simple first.** Common tasks take a line or two and read plainly.
- **A mix** of plain structs, events and properties, each used where it fits.
- **Everything reachable** (goal 1), without making simple code complicated.

The exact shape is decided in the RFCs, starting from examples written by the
project lead.

**Plugins (accepted, [RFC 0001](rfcs/0001-plugins.md)):** a plugin is a
struct embedding `helix.Plugin` with a required `Info` (name, description,
version, dependencies). Listeners are registered only in `Setup`, as
`r.BlockBreak(fn, helix.Priority(event.High), helix.OnlyBlock(...))`, any number
per event. Plugins can be enabled and disabled at runtime, which stops
everything they registered. Plugin panics are recovered and logged; the owner
may set a limit after which the plugin is disabled.

### Requirements

These say what the API must do, not what it looks like.

**Three layers.** Each one is reachable from the one above it (player →
connection → packets → bytes). Simple plugins never need to touch the lower
layers; advanced plugins are never blocked by them.

| Layer | Contains | Tier |
|---|---|---|
| Game | Players, entities, blocks, items, worlds, and events for real game actions | Stable |
| Packet | Every decoded packet, incoming and outgoing: read, modify, cancel, send new ones | Advanced (changes with Minecraft versions) |
| Raw | Bytes per connection, before decoding and after encoding | Advanced |

**Helix owns its whole stack.** RakNet, NetherNet, the Bedrock protocol
(packets, login, encryption) and everything above are Helix's own code, in the
`raknet`, `nethernet` and `protocol` repos. Existing libraries such as
gophertunnel and go-raknet are references to port from (see `PORTING.md`), not
dependencies, because they do not expose the low-level access below.

**Anti-cheat is optional.** Helix will have its own anti-cheat module, off by
default and switchable, and servers and plugins can build their own instead,
so everything an anti-cheat needs must be exposed and replaceable: movement and its validation, combat and reach, block
interactions, inventory transactions, packet timing, and per-player packet and
transport hooks.

**Transports.** Both RakNet (classic UDP) and NetherNet (WebRTC-based, used
for friends lists and LAN by newer Bedrock clients) are supported. The
transport sits under the raw layer and is pluggable, so packet and game code
works the same whichever one a player uses, and servers can add their own
transport. Plugins can see which transport each player is on.

**Transport events.** The transport layer is fully observable through events,
before a player exists and after:
- Connection attempts: every RakNet connection request (unconnected ping,
  open connection requests) and every NetherNet signalling message and
  WebRTC offer, with the source address, full raw bytes, the decoded fields,
  and all transport-specific details (RakNet: MTU, protocol version, GUID;
  NetherNet: network ID, signalling data, ICE candidates, connection state).
  Listeners can accept, reject or rate-limit the attempt.
- Every transport-level packet in and out (RakNet frames and NetherNet
  messages), tied to the connection and, once known, the player, with raw
  bytes, decoded content, size, timing and address.
- Connection lifecycle: connected, login, latency changes, disconnect with
  reason.

**Per-player packets.** The player is the entry point to its own connection:
- Send any packet or raw bytes to one player (for example `p.WritePacket(pk)`).
- Hook incoming and outgoing packets and raw bytes for one player, as well as
  for all players. Hooks can read, modify or cancel.
- Hooks return a function that removes them; per-player hooks are removed
  automatically when the player leaves.
- Custom plugin data attaches to the player too.

**Not bloated.**
- One way to do each thing. Listening to a game event, a packet or raw bytes
  works the same way.
- Packet types and packet hooks are generated from protocol definitions, not
  written by hand.
- A packet only gets a separate game event if it is a real game action. No
  duplicate events for the same thing.
- A small top-level API; deeper parts live in packages imported only when
  needed.
- Hooks nobody uses cost nothing.

**Stable.**
- The game layer keeps the stable tier's compatibility promise (goal 7).
- The packet and raw layers are versioned with the Minecraft protocol and
  documented as changing between Minecraft updates.
- A debug mode validates packets written by plugins and reports clear errors
  instead of disconnecting players.

## What needs designing

These are the topics, not decisions. Each will get its own RFC.

| # | Topic | Status |
|---|---|---|
| 1 | Core: tick loop, worlds and dimensions, threading model, region ticking, core-count configuration | Accepted: [RFC 0002](rfcs/0002-core.md) and [Engine design](#engine-design-decided-2026-10-29) sections 2-3; API shape in draft [RFC 0003](rfcs/0003-region-api.md) |
| 1a | Concurrency rules for the API: which calls run where, scheduling onto chunk or entity owners, wrong-thread detection | Accepted: [RFC 0002](rfcs/0002-core.md); API shape in draft [RFC 0003](rfcs/0003-region-api.md) |
| 2 | Registries: blocks, items, entities, behaviours, commands | Accepted: [Engine design](#engine-design-decided-2026-10-29) section 1; commands decided 2026-11-05 (see Commands) |
| 3 | Module and plugin system: how vanilla features and plugins register | Accepted: [RFC 0001](rfcs/0001-plugins.md) |
| 4 | Event system | Accepted: [RFC 0001](rfcs/0001-plugins.md) and [Engine design](#engine-design-decided-2026-10-29) section 5; the event list and custom events are still open |
| 5 | Entity model: components, physics, damage | Hosting accepted: [Engine design](#engine-design-decided-2026-10-29) sections 1 and 4; components, AI and physics live in the `entity` library |
| 5a | Property system | Not started |
| 5b | Scheduler | Not started |
| 6 | AI framework: memory, sensors, behaviours, controllers, pathfinding | Not started |
| 7 | Data loading: `bedrock-samples` and BedrockData formats | Not started |
| 8 | External plugins: WASM host, API surface, sandboxing | Not started |
| 9 | Storage: LevelDB worlds, player data | Accepted: [Engine design](#engine-design-decided-2026-10-29) section 6 |
| 10 | Versioning: multi-version protocol support (decided: support as many versions as possible through the `protocol` library, translating packets per client version), API compatibility promise | Not started |
| 11 | Custom content: normal Minecraft resource packs are supported and sent to clients as-is. Custom entities, blocks and items use Helix's own system (through the API), not Bedrock behaviour packs. Whether that system also has a JSON format is to be discussed later | Not started |
| 12 | Login and accounts: Xbox Live authentication with many verification checks that the connection is a real client (login chain signatures, token expiry, identity and XUID consistency, client data, protocol behaviour), studied from how other servers do it (references: PocketMine-MP login verification, PowerNukkitX Xbox login handling); offline mode also available for the server owner to choose. Every check is exposed and can be extended or replaced through the API. Skins included | Built from BDS 1.26.60's login checks: `protocol/auth` (token checks, clock correction), `nethernet` (server identity key), `helix/network` (login steps as named, replaceable steps); offline mode is the `OnlineMode` setting |
| 13 | UI: forms, scoreboards, boss bars, titles. All exposed through the API; designed when that phase is reached | Not started |
| 14 | Server networks: proxy support and moving players between servers. Undecided whether to support WaterdogPE-style proxies or build a Helix proxy | Not started |
| 15 | Plugin permissions: what a WASM plugin may do by default | Not started (open question) |
| 16 | Config: a simple `.env` file for basic server settings only; everything else is set in code through the API | Not started |
| 17 | Monitoring: Helix's own metrics, profiling and dashboards (shared with `helix-bench`) | Not started |

### Prior art for multi-core (topics 1 and 1a)

RFCs for these topics must compare against these projects:

| Project | Language | What to study |
|---|---|---|
| [Pumpkin](https://github.com/Pumpkin-MC/Pumpkin) | Rust | Worlds, players and entities ticked in parallel with Rayon (`world/mod.rs`, `server/mod.rs`); Tokio for network, Rayon pool for chunk generation and lighting. Entity order is not fixed, which conflicts with goal 8's determinism rule |
| [Folia](https://github.com/PaperMC/Folia) | Java | Region ticking: independent regions on separate threads that merge and split as players move |
| [Minestom](https://github.com/Minestom/Minestom) | Java | Thread dispatcher that assigns chunks and instances to a configurable number of threads |
| [Dragonfly](https://github.com/df-mc/dragonfly) | Go | One goroutine per world; outside code runs inside the world through transactions. The closest Go model for scheduling onto an owner |
| [MultiPaper](https://github.com/MultiPaper/MultiPaper) | Java | Scaling one world across several server processes |

## How a design gets accepted

While the project lead is the only maintainer, the lead decides designs in
discussion and they are recorded in this file, as in
[Engine design](#engine-design-decided-2026-10-29). RFC files are optional.
Once there are more core maintainers, this process applies:

1. Someone opens a Discussion on `helix-bedrock/.github` proposing the topic.
2. They write an RFC as a Markdown file in an `rfcs/` folder in this repo
   (`rfcs/0001-core.md`), opened as a PR. It covers the problem, the proposed
   design with Go interfaces, alternatives considered, and how other projects
   (Dragonfly, PowerNukkitX, Allay, Pumpkin) handle the same thing.
3. Discussion happens on the PR. Prototypes in `experiments/` can back it up.
4. **2 core maintainers approve** and the PR merges. The table above is
   updated to **Accepted** with a link. While the project lead is the only
   maintainer, the lead accepts a design by merging a PR that marks it
   **Accepted**; merging a draft RFC does not accept it.
5. Implementation can start for that topic only.

Designs can change after acceptance through a follow-up RFC.
