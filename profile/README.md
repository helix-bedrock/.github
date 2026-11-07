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

> **Status: building** (as of 2026-11-07). The core design is decided and the engine is being
> built. Full checklist: [helix/STATUS.md](https://github.com/helix-bedrock/helix/blob/main/STATUS.md).
> Follow along in [Discussions](https://github.com/orgs/helix-bedrock/discussions).

### What works (checked in game)

- Joining (NetherNet, Xbox Live login, encryption, resource packs), BDS worlds opened and saved in
  BDS's own format, the registry, chunks, and regions ticking on many cores.
- The inventory, chests, shulker boxes, furnaces, crafting and smithing tables, all from BDS's
  code. Their basic use is checked; not every case yet.
- Player combat, health, hunger and totems.
- Redstone and rails basically fully work (not yet compared with BDS side by side).
- Bows and arrows.

### Built, not finished or not checked yet

- Block ticks: many blocks' tick behaviour is in, not all.
- Snowballs, eggs and ender pearls are built but do not work in game yet.
- Vanilla terrain generation is hooked up for the overworld; all three dimensions are generated
  in [worldgen](https://github.com/helix-bedrock/worldgen), with a refinement pass still to come.
- The command system and `/give`, riding, the day cycle, beds, banners, signs, fire and more are
  written from BDS's code and waiting for in-game tests.

### Not started

- Mobs: [entity](https://github.com/helix-bedrock/entity) has most of BDS's components mapped, but
  the engine does not run mobs yet.
- Weather, potions, crossbows, tridents, fishing, brewing, most vanilla commands, forms and UI.

### Repositories

| Repo | What it is |
|---|---|
| [helix](https://github.com/helix-bedrock/helix) | The server engine |
| [protocol](https://github.com/helix-bedrock/protocol) | Packets, NBT, login and connections |
| [raknet](https://github.com/helix-bedrock/raknet), [nethernet](https://github.com/helix-bedrock/nethernet) | The two network transports, with full low-level access |
| [world](https://github.com/helix-bedrock/world) | Worlds read and written byte for byte as BDS does |
| [worldgen](https://github.com/helix-bedrock/worldgen) | Vanilla world generation, one-to-one with BDS |
| [entity](https://github.com/helix-bedrock/entity) | Mob AI and pathfinding, one-to-one with BDS |
| [redstone](https://github.com/helix-bedrock/redstone) | Redstone circuits, one-to-one with BDS |
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

- [Your first contribution, step by step](https://github.com/helix-bedrock/.github/blob/main/FIRST_CONTRIBUTION.md)
- [Contributing guide](https://github.com/helix-bedrock/.github/blob/main/CONTRIBUTING.md)
- [API design status](https://github.com/helix-bedrock/.github/blob/main/API_DESIGN.md)
- [Code of conduct](https://github.com/helix-bedrock/.github/blob/main/CODE_OF_CONDUCT.md)
