<p align="center">
  <img src="https://raw.githubusercontent.com/helix-bedrock/brand/main/helix-wordmark-light.svg#gh-light-mode-only" alt="Helix" width="420">
  <img src="https://raw.githubusercontent.com/helix-bedrock/brand/main/helix-wordmark-dark.svg#gh-dark-mode-only" alt="Helix" width="420">
</p>

<p align="center"><b>An open-source Minecraft: Bedrock Edition server engine, written in Go.</b></p>

Helix is an engine for any kind of server.

- **Running a survival SMP?** Turn on vanilla: mobs with real AI, villages and
  trading, redstone, world generation, the Nether and the End.
- **Building Skyblock or minigames?** Turn vanilla off, keep the fast core, and
  write your own rules.

Everything is exposed. Every feature is a module on the same API you use, and
even the engine's fundamentals can be replaced.

> **Status: early design.** We're writing the architecture before the code.
> Follow along in [Discussions](https://github.com/orgs/helix-bedrock/discussions).

### Principles

- **Fast on every core:** worlds, regions, mob AI and chunk work run in parallel,
  and you choose how many cores Helix uses, from one to all of them.
- **Stable:** Go, with a strict no-allocation rule on hot paths.
- **Modular:** vanilla features and plugins use the same API.
- **Data-driven:** game content loads from Mojang's own data files.
- **Vanilla-accurate:** behaviour is checked against the official server.
- **Open:** GPL-3.0 engine, MIT libraries, AI-assisted contributions welcome
  under our [AI policy](https://github.com/helix-bedrock/.github/blob/main/AI_POLICY.md).

### Get involved

- [Contributing guide](https://github.com/helix-bedrock/.github/blob/main/CONTRIBUTING.md)
- [API design status](https://github.com/helix-bedrock/.github/blob/main/API_DESIGN.md)
- [Code of conduct](https://github.com/helix-bedrock/.github/blob/main/CODE_OF_CONDUCT.md)
