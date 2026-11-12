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

> **Status: building** (as of 2026-11-12). Live progress, by feature: [helixbedrock.com/progress](https://helixbedrock.com/progress/).
> Follow along in [Discussions](https://github.com/orgs/helix-bedrock/discussions).

Helix is **in early development and not ready for real servers yet**. The road to a stable release:

1. ✅ Engine core: networking, world storage, regions, terrain (done, Oct to Nov 2026).
2. 🟡 Every vanilla mechanic, one-to-one with BDS (in progress, aiming for summer 2027).
3. ⬜ Plugin API and custom content (2027 to 2028).
4. ⬜ Testing on real servers, then a stable release (around 2028).

### What works

- Joining (NetherNet, Xbox Live login, encryption, resource packs), BDS worlds in BDS's own format,
  vanilla terrain, the Nether and the End, light, weather, and regions ticking on many cores.
- Players: movement, health, hunger, combat, shields, totems, elytra, portals, beds.
- Inventory and every common container and work block: chests, furnaces, hoppers, crafting,
  smithing, enchanting, anvils.
- Redstone, pistons, observers, TNT, rails and minecarts.
- Farming and plants, falling blocks, fire, fishing, bundles, fireworks, goat horns.
- Mobs spawn, fight back, wear equipment and drop loot.

### Built, being tested in game

- Potions, status effects, brewing, crossbows, tridents, the mace, wind charges.
- Breeding, leads, mob spawners, spawn eggs, name tags, mobs saved with the world.
- Armour stands, candles, cakes, campfires, barrels, stonecutters.

### Still to build

- Villages, villagers and trading; raids; the Ender Dragon and the Wither; structure loot; maps,
  beacons, conduits; most vanilla commands, forms and UI.
- Every mob behaviour checked one-to-one against BDS, and load testing at 400 players.
- The plugin API.
- These lists only cover what we know about: there are surely mechanics we have not found yet that still need to be added.

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
