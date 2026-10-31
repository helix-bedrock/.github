<p align="center">
  <img src="https://raw.githubusercontent.com/helix-bedrock/brand/main/helix-wordmark-light.svg#gh-light-mode-only" alt="Helix" width="420">
  <img src="https://raw.githubusercontent.com/helix-bedrock/brand/main/helix-wordmark-dark.svg#gh-dark-mode-only" alt="Helix" width="420">
</p>

<p align="center"><b>An open-source Minecraft: Bedrock Edition server engine, written in Go.</b><br>
BDS, but better: the same mechanics, with access to everything from the lowest level to the highest.</p>

No Bedrock server software matches the official Bedrock Dedicated Server (BDS) one-to-one yet.
Helix aims to be the first, and to run any kind of server.

- **Running a survival SMP?** Turn on vanilla: world generation, mobs, villages, redstone, the
  Nether and the End, all working exactly as in BDS.
- **Building Skyblock or minigames?** Turn vanilla off, keep the fast core, and write your own
  rules.

Every mechanic is BDS's by default and can be replaced. Every feature is a module on the same API
plugins use, and even the engine's fundamentals can be replaced.

> **Status: building.** The core design is decided and the engine is being built. Players can
> already join, spawn in a BDS world, see each other and use their inventory.
> Follow along in [Discussions](https://github.com/orgs/helix-bedrock/discussions).

### What we're working on

- **Core player gameplay** in [helix](https://github.com/helix-bedrock/helix): breaking, placing
  and using blocks, item drops, water and lava flow, and containers. Networking, login, world
  storage, the registry and multi-core regions are done.
- **World generation** in [worldgen](https://github.com/helix-bedrock/worldgen): all three
  dimensions are generated as BDS does; a refinement pass for the last small differences is next.
- **Mob AI** in [entity](https://github.com/helix-bedrock/entity): researching BDS's components,
  goals and pathfinding to port them one-to-one.

### Repositories

| Repo | What it is |
|---|---|
| [helix](https://github.com/helix-bedrock/helix) | The server engine |
| [protocol](https://github.com/helix-bedrock/protocol) | Packets, NBT, login and connections |
| [raknet](https://github.com/helix-bedrock/raknet), [nethernet](https://github.com/helix-bedrock/nethernet) | The two network transports, with full low-level access |
| [world](https://github.com/helix-bedrock/world) | Worlds read and written byte for byte as BDS does |
| [worldgen](https://github.com/helix-bedrock/worldgen) | Vanilla world generation, one-to-one with BDS |
| [entity](https://github.com/helix-bedrock/entity) | Mob AI and pathfinding, one-to-one with BDS |
| [helix-data](https://github.com/helix-bedrock/helix-data) | Game data from Mojang's files and the scripts that build it |
| [multiversion](https://github.com/helix-bedrock/multiversion) | Support for older Minecraft versions |

### Principles

- **Vanilla one-to-one:** behaviour matches BDS exactly, checked against the official server.
- **Rich API:** everything is reachable and replaceable, from packets and chunks to gameplay.
- **Fast on every core:** regions tick in parallel, and you choose how many cores Helix uses.
- **Stable:** Go, no panics, and a strict no-allocation rule on hot paths.
- **Data-driven:** game content loads from Mojang's own data files.
- **Open:** GPL-3.0 engine, MIT libraries, AI-assisted contributions welcome under our
  [AI policy](https://github.com/helix-bedrock/.github/blob/main/AI_POLICY.md).

### Get involved

- [Contributing guide](https://github.com/helix-bedrock/.github/blob/main/CONTRIBUTING.md)
- [API design status](https://github.com/helix-bedrock/.github/blob/main/API_DESIGN.md)
- [Code of conduct](https://github.com/helix-bedrock/.github/blob/main/CODE_OF_CONDUCT.md)
