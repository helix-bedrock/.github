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

### 6. Storage

- **One LevelDB per world, exactly like BDS,** is the default: chunks,
  entities, players and world data in the same database, every key,
  format and read rule one-to-one with BDS, so any BDS world opens in Helix
  and any Helix world opens in BDS. Helix-only data sits under keys BDS
  ignores.
- Players are stored as BDS does: `player_server_<uuid>` holds the data,
  `player_<id>` records map each login ID to it.
- **Storage is pluggable.** The world store and the player store are separate
  interfaces (both default to the same LevelDB), so a network can keep
  players in one shared database. A backend can implement plain key-value
  calls (BDS keys and encoding are reused) or take over the typed calls
  completely.
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
   event (a log warning for now), the rest of RFC 0003, and
   section size and merge distance chosen by `helix-bench`. Entity
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

### Still open

The full event list, custom events, the entity visibility API (leaning
towards `CanSee`, `HideActor`, `ShowActor`), commands and permissions, the
server config file, and BDS's autosave timing.

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
| 2 | Registries: blocks, items, entities, behaviours, commands | Accepted: [Engine design](#engine-design-decided-2026-10-29) section 1 (commands still open) |
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
