# RFC 0004: Entity packets (vanilla-exact network sync)

**Status:** Draft. Not accepted.
**Topics:** the network side of 5 (entity model): which entity packets a
server sends, to whom, when, and with what values. No topic in
[API_DESIGN.md](../API_DESIGN.md) covers entity networking yet; this RFC
proposes adding it under 5.
**Builds on:** [RFC 0002](0002-core.md) (regions and owners),
[RFC 0003](0003-region-api.md) (where calls run).

## Problem

A Bedrock client only sees what the server sends. To match vanilla (goal:
"vanilla is the reference"), a server has to send the same entity packets as
the official Bedrock Dedicated Server (BDS): the same packet types, in the same
order, on the same ticks, with the same rounding. Getting this right means
knowing details that are not documented anywhere:

- when a mob's movement is "significant enough" to send,
- how BDS skips movement packets for far players (bandwidth optimisation),
- how rotations are rounded,
- when a player is told an entity exists (AddActor) or is gone (RemoveActor),
- in which order equipment, data, movement and sounds go out in a tick.

Most engines get these wrong. For example, Dragonfly (the base of
`helix-new`) sends a full `MoveActorAbsolute` with unrounded float rotations
to every viewer whenever an entity moves, and shows entities by the player's
chunk view radius. BDS does neither.

We researched BDS 1.26.60 (decompiled code plus live captures of a real
client joining a BDS run under gdb). This RFC proposes a **separate
library** that holds these rules, so that Helix and any other engine can send
vanilla-exact entity packets through a write hook.

The research notes and capture data are private (they reference BDS
functions); the rules below are what the library implements. Each rule says
whether it was confirmed live or only read from the code.

## What BDS does (1.26.60)

### Tick order

Within one server tick, per dimension (confirmed live):

1. Actors tick. Sounds raised now are **queued**, not sent.
2. **Data pass**, every actor in actor order: `UpdateAttributes` (if any
   attribute is dirty), then `SetActorData` (if any synched data or entity
   property is dirty). Broadcast to viewers.
3. **Movement pass**, every actor in the same order: `MoveActorDelta` to each
   viewer that passes the throttle.
4. **Sound flush**: that tick's `LevelSoundEvent` packets.

View changes (AddActor, RemoveActor and the equipment that follows AddActor)
come from a per-dimension scan and the player's own tick (see *View*). Where
those two run relative to passes 2-4 is **not confirmed yet** (open
question 1).

### Movement

The library keeps a **last-sent state** per actor: position (3 float32), and
pitch, yaw and head yaw as **bytes**.

- **Rotation byte**: `wrap(d) = fmod(d + 180, 360)`, plus 360 if negative,
  minus 180; then `byte = int32((wrap(d) / 360) * 256)` in float32, truncated.
  Back to degrees: `float32(int8(b)) * (1/256) * 360`.
  Head yaw is only sent for mobs; for other actors it is 0.
- **Significant** (checked every tick): any of
  - squared distance from the last-sent position > 0.0001,
  - |Δpitch| + |Δyaw| + |Δhead yaw| > 4°, using the byte→degree values and no
    wrap-around handling,
  - the actor has (almost) stopped (motion² < 1e-6) and squared distance >
    1e-6: the final "settle" update.
- If not significant, nothing is sent.
- If significant, a `MoveActorDelta` is built against the last-sent state.
  Header bits:

  | Bit | Meaning |
  |---|---|
  | 0, 1, 2 | x, y, z float changed (exact `!=`) |
  | 3, 4, 5 | pitch, yaw, head yaw byte changed |
  | 6 | on ground |
  | 7 | teleported |
  | 8 | force move local |
  | 9 | the third force flag |

  Changed axes carry the **full** float, not a difference. Bits 10-15 were
  never set in 4,258 captured deltas.
- A server always sends **deltas**, players included. The only forced
  `MoveActorAbsolute` found is when a player rider is dismounted.
- **Last-sent is updated only if at least one viewer received the delta.**

### Throttle (confirmed live: 4,764 of 4,990 sends exact, 0 violations)

Per actor and per viewer, BDS stores the tick of the last delta sent to that
viewer (never sent counts as tick 0).

- `score = int(0.75 * maxDroppedTicks * min(dist, maxDist) / maxDist)`, with
  `dist` the float32 3D distance between actor and viewer and the defaults
  `maxDist = 80`, `maxDroppedTicks = 10`. Far actors therefore score 7.
- Score is 0 (send every tick) when on-ground changed since last-sent, when
  the actor has no bandwidth settings, or when one other tracked per-actor
  value changed (open question 2).
- Send when `tick - lastSentTick(viewer) >= score`.
- Some actors skip the throttle and always send.
- A viewer only gets the delta if it knows the actor (see *View*).

### Motion, events, swing

- `SetActorMotion` is sent only on events (knockback on players, pulls,
  motion zeroed), never per tick. No server-sent `MotionPredictionHints` was
  found.
- `ActorEvent` and `Animate` are broadcast to every viewer that knows the
  actor, at once.
- Arm swing: `Animate` action 1, at most once per 10 ticks per mob.

### View: AddActor and RemoveActor (confirmed live)

Each player has a set of known actors. Every tick:

1. **Scan.** For each player, take the chunk it stands in
   (`floor(x)>>4`, `floor(z)>>4`) and add a fixed list of offsets: **all
   57 chunks with dx² + dz² <= 17** (a radius of about 4.1 chunks). This is
   not the client's view distance, and changing `tick-distance` to 6 did not
   change it. Every actor in those chunks (loaded chunks only, not removed,
   not the player itself) is marked seen: new entries are *new*, existing
   ones *seen again*. Other players standing in those chunks are added the
   same way.
2. **Player tick.** For each entry:
   - *new*: send `AddActor`, then `MobArmorEquipment` if any armour slot is
     filled, then `MobEquipment` (main hand) if the carried item is not
     empty, then the offhand. All in the same tick, back to back.
   - *seen again*: nothing.
   - *not seen this tick*: send `RemoveActor` and forget the entry. Some
     actors are kept until they stop existing even when out of range.
3. On a dimension change or respawn, every known actor gets `RemoveActor`.

There is no distance check other than the chunk disc and no grace period.
After a teleport, the new area's AddActors come on tick T and the old area's
RemoveActors on T+1.

`AddActor` carries the **last-sent** position and rotations (converted back
from bytes, so always multiples of 1.40625°), not the live values.

### Equipment changes (confirmed live)

- A newly spawned mob: `MobEquipment` (main hand), `MobEquipment`
  (offhand), `MobArmorEquipment`, broadcast in that order in one tick.
- A changed armour slot: `MobArmorEquipment` (all slots) on that tick. A
  changed held item: `MobEquipment` for that hand. Only the changed kind is
  sent.

## Proposal

### A library, not engine code

A new library repository, working name **`actornet`** (name open), MIT, with
no dependency on the Helix engine. It depends on `helix-bedrock/protocol` for
packet types (an org repo; still needs maintainer approval under rule 4).
Older clients are handled the usual way: the library emits current-version
packets and `helix-bedrock/multiversion` adapts them in `conn`.

It does not move entities, run AI, or know block or item names. The engine
tells it the state of each actor and each viewer every tick; the library
decides which packets go to whom, builds them, and hands them to the
engine's writer.

Considered and rejected:

- **Inside the Helix engine.** Other engines could not use it, and the
  rules are useful on their own.
- **Inside `entity`.** `entity` is mob AI maths and must not know about
  packets or players.
- **Inside `protocol`.** `protocol` is wire format only; these are game
  rules.

### Shape

```go
package actornet

// Writer is the engine's write hook for one viewer.
// Runs: called on the tracker's owner; must not block.
type Writer interface {
	WritePacket(p packet.Packet)
}

// State is what the engine reports for an actor each tick.
type State struct {
	RuntimeID uint64
	UniqueID  int64
	Position  mgl32.Vec3
	Motion    mgl32.Vec3
	Pitch, Yaw, HeadYaw float32
	OnGround  bool
	Mob       bool // has head yaw and equipment
	Flags     Flags // AlwaysSend, KeepWhileAlive, ...
	Bandwidth *Bandwidth // nil: score is always 0
}

// Spawner builds the parts of a spawn the library cannot know: metadata,
// attributes, links and equipment items. The library fills position and
// rotation from the last-sent state.
type Spawner interface {
	AddActor(id int64, p *packet.AddActor) bool
	Equipment(id int64, e *Equipment) bool
}

// Tracker holds the network state of one dimension.
// Runs: owner (the dimension or region that ticks these actors).
type Tracker struct{ /* ... */ }

func New(cfg Config) *Tracker

func (t *Tracker) SetActor(s State)          // add or update
func (t *Tracker) RemoveActor(uniqueID int64)
func (t *Tracker) SetViewer(uniqueID int64, pos mgl32.Vec3, w Writer)
func (t *Tracker) RemoveViewer(uniqueID int64)

func (t *Tracker) Teleport(uniqueID int64, forceMoveLocal bool)
func (t *Tracker) Broadcast(uniqueID int64, p packet.Packet) // ActorEvent, Animate, equipment, data
func (t *Tracker) QueueSound(p *packet.LevelSoundEvent)

// Tick runs the passes in BDS order for the given server tick.
func (t *Tracker) Tick(tick uint64, sp Spawner)
```

Every rule is also a separate exported function or interface, so an engine
with its own loop can use only the pieces it wants (goal 1), and anything can
be replaced (goal 3):

```go
func RotationByte(deg float32) byte
func ByteDegrees(b byte) float32
func Significant(last, cur Sent, motion mgl32.Vec3) bool
func DeltaHeader(last, cur Sent, onGround, teleported bool) uint16
func Score(dist float32, b Bandwidth) int64

type ViewArea interface {
	Chunks(center ChunkPos, dst []ChunkPos) []ChunkPos
}

type Config struct {
	View      ViewArea  // DefaultView(): the 57-chunk disc
	Bandwidth Bandwidth // DefaultBandwidth(): 80 blocks, 10 ticks
	Throttle  bool      // false: send every significant delta to every viewer
}

func DefaultConfig() Config
```

The library knows chunks only as coordinates; the engine says which chunks
are loaded through `SetActor` (an actor in an unloaded chunk is not
reported).

### How it meets the goals

1. **Expose everything.** Every rule is an exported function or an interface
   in `Config`; nothing is internal.
2. **Vanilla is just modules.** Helix's vanilla entity module uses the library
   through this public API, and a minigame can skip it and write packets
   itself.
3. **Replaceable layers.** View area, throttle, and spawner are swappable;
   the defaults are BDS.
6. **Pay only for what you use.** No reflection, no events. Steady-state
   ticks do not allocate: packets come from the protocol pool, per-viewer
   maps and slices are reused. Benchmarks with `b.ReportAllocs()` for
   `Tick` with 1,000 actors and 50 viewers.
8. **Multi-core.** One `Tracker` per dimension or region, owned by it
   (`Runs: owner`). No shared state, no locks inside. Packet order follows
   the actor order the engine passes in, so 1 core and many cores give the
   same bytes. Crossing regions (an actor moving to another region) is the
   engine's job: it calls `RemoveActor` on one tracker and `SetActor` on the
   other, and the library offers a hand-over that carries last-sent state so
   no extra AddActor is sent.

Errors from bad input (unknown ids, NaN positions) are ignored or returned;
the library never panics.

### Testing

- Unit tests from the captured BDS values: rotation bytes, delta headers,
  the 4,990 throttle decisions, and the view-set changes after a teleport.
  The captured values are exported without BDS addresses into test fixtures,
  with a `source` line in `CREDITS.md`.
- A replay test: feed the actor states of a capture tick by tick and compare
  the packets with what BDS sent.
- In-game testing by a human, per [TESTING.md](../TESTING.md): walk around
  mobs at 5, 30 and 70 blocks, fly out of and back into range, teleport,
  change equipment, two players seeing each other; compare with BDS in the
  same world.

## Open questions

1. Where the view scan and the player tick run relative to the data,
   movement and sound passes. Needed for exact packet order across types.
2. The second per-actor value that forces score 0, and the component that
   gives a player a second view centre.
3. `AddPlayer` and player-to-player view: needs a two-player capture.
4. Which code builds the equipment packets for spawns and changes (only
   observed, not read).
5. Sound queue: flush order is known; hearing range and target rules are
   not read yet.
6. Whether the 57-chunk table depends on any setting. `view-distance` was
   not changed in tests.
7. Repository name, and whether the default `Spawner` (metadata, attributes)
   belongs here or in Helix.

## Prior art

- **Dragonfly** (Go, checked in `helix-new`): `MoveActorAbsolute` with float
  rotations on every movement, `SetActorMotion` for velocity, entities shown
  by the chunk loader's radius. No significance check, no throttle, no
  rotation rounding.
- **PowerNukkitX, Allay, Pumpkin**: not compared yet. To be done before this
  RFC asks for acceptance.
