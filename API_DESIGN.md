# API design

**Status: in design.** Accepted: plugins and listeners
([RFC 0001](rfcs/0001-plugins.md)), core and threads
([RFC 0002](rfcs/0002-core.md)). Engine code may start for those topics only.

The engine's core architecture and plugin API are being designed. Until
this file says a design is **accepted**, nobody (human or AI) writes engine
packages, public interfaces, registries, the module system or the plugin API.
Prototypes are fine in a folder named `experiments/` that is never merged.

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

**No built-in anti-cheat.** Helix ships no anti-cheat of its own. Servers and
plugins build their own, so everything an anti-cheat needs must be exposed and
replaceable: movement and its validation, combat and reach, block
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
| 1 | Core: tick loop, worlds and dimensions, threading model, region ticking, core-count configuration | Accepted: [RFC 0002](rfcs/0002-core.md); API shape in draft [RFC 0003](rfcs/0003-region-api.md) |
| 1a | Concurrency rules for the API: which calls run where, scheduling onto chunk or entity owners, wrong-thread detection | Accepted: [RFC 0002](rfcs/0002-core.md); API shape in draft [RFC 0003](rfcs/0003-region-api.md) |
| 2 | Registries: blocks, items, entities, behaviours, commands | Not started |
| 3 | Module and plugin system: how vanilla features and plugins register | Accepted: [RFC 0001](rfcs/0001-plugins.md) |
| 4 | Event system | Listeners accepted in [RFC 0001](rfcs/0001-plugins.md); events not started |
| 5 | Entity model: components, physics, damage | Not started |
| 5a | Property system | Not started |
| 5b | Scheduler | Not started |
| 6 | AI framework: memory, sensors, behaviours, controllers, pathfinding | Not started |
| 7 | Data loading: `bedrock-samples` and BedrockData formats | Not started |
| 8 | External plugins: WASM host, API surface, sandboxing | Not started |
| 9 | Storage: LevelDB worlds, player data | Not started |
| 10 | Versioning: multi-version protocol support (decided: support as many versions as possible through the `protocol` library, translating packets per client version), API compatibility promise | Not started |
| 11 | Custom content: normal Minecraft resource packs are supported and sent to clients as-is. Custom entities, blocks and items use Helix's own system (through the API), not Bedrock behaviour packs. Whether that system also has a JSON format is to be discussed later | Not started |
| 12 | Login and accounts: Xbox Live authentication with many verification checks that the connection is a real client (login chain signatures, token expiry, identity and XUID consistency, client data, protocol behaviour), studied from how other servers do it (references: PocketMine-MP login verification, PowerNukkitX Xbox login handling); offline mode also available for the server owner to choose. Every check is exposed and can be extended or replaced through the API. Skins included | Not started |
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
