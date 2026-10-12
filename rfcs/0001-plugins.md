# RFC 0001: Plugins and listeners

**Status:** Draft. Agreed by the project lead in discussion; not yet accepted.
**Topics:** 3 (module and plugin system: how plugins register) and the
listener part of 4 (event system).

## Problem

Helix needs one way for plugins, and vanilla modules (goal 2), to say who they
are, register what they need, and be turned on and off while the server runs.
It must be simple for small plugins, give full control to large ones, and cost
nothing for events nobody listens to (goal 6).

## Design

A plugin is a Go struct that embeds `helix.Plugin`. Its data lives in its own
fields.

```go
type Skyblock struct {
    helix.Plugin
    islands map[string]*Island
}

func (s *Skyblock) Info() helix.Info {
    return helix.Info{
        Name:         "Skyblock",                // required
        Description:  "Islands and generators.", // required
        Version:      "1.0.0",
        Dependencies: []string{"Economy"},
    }
}

func (s *Skyblock) Setup(r *helix.Setup) {
    r.BlockBreak(s.protect, helix.Priority(event.High))
    r.BlockBreak(s.log, helix.Priority(event.Monitor), helix.IgnoreCancelled())
    r.BlockBreak(s.ores, helix.OnlyBlock("minecraft:diamond_ore"), helix.OnlyWorld("skyblock"))
    r.PlayerJoin(s.kit)
}

func (s *Skyblock) Enable()  {}
func (s *Skyblock) Disable() {}
```

### Info

- `Name` and `Description` are required. A plugin without them fails to load
  with a clear error.
- `Version` is shown in logs and plugin lists.
- `Dependencies` names other plugins. They load first; if one is missing, the
  plugin refuses to load and says which one.

### Listeners

- Listeners are registered only in `Setup`, as `r.<Event>(fn, options...)`.
  There are no methods that become listeners because of their name, so a typo
  cannot silently do nothing.
- The same event can be registered any number of times, each with its own
  options.
- Settings are options, so new ones can be added later without breaking
  plugins:
  - `helix.Priority(p)`: when the listener runs relative to others.
  - `helix.IgnoreCancelled()`: skip the listener if an earlier one cancelled
    the event.
  - Filters such as `helix.OnlyBlock(name)` and `helix.OnlyWorld(name)`. The
    engine skips the call when a filter does not match, so the listener does
    not have to check, and a filtered-out event costs no call.
- No reflection when events fire: `Setup` builds the listener lists once.

### Enable and disable

- Plugins can be enabled and disabled while the server runs, by the server
  (console command or config) and by other plugins through
  `server.Plugins()`.
- Disabling stops everything the plugin registered: listeners, scheduled
  tasks, packet hooks, and its commands are hidden. Enabling brings it all
  back without running `Setup` again.
- Custom blocks and items stay registered while disabled, so worlds that
  contain them keep loading and saving.
- `Enable` and `Disable` run after and before those changes, for the plugin's
  own work (opening files, saving data).

### Panics

Helix never panics, and a plugin's panic never stops the server:

- A panic in any plugin code is recovered at the call, logged with the
  plugin's name, and the server keeps running.
- The server owner can set a limit (for example 10 panics in a minute) after
  which the plugin is disabled. By default there is no limit.

## Open questions for later RFCs

- Where listeners run when Helix uses many cores, and what a listener may
  touch from there (topics 1 and 1a, RFC 0002).
- The list of events and their fields (topic 4).
- Priority names and order.
- How WASM plugins map onto this (topic 8).

## Alternatives considered

- **Methods named after events** (`func (s *Skyblock) BlockBreak(e)`): short,
  but a misspelled name silently never runs, priorities have nowhere to go,
  and one event can only have one listener. Rejected.
- **Dragonfly-style handler structs** (one handler type with a method per
  event, set per player): good for per-player logic, but awkward for
  server-wide plugins and for many listeners on one event. Rejected as the
  main style.
- **Plain priority values** (`r.BlockBreak(fn, event.High)`): shortest, but
  every later setting would need a new method or a breaking change. Rejected
  for options.
