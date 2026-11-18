# Porting rules

Helix may reuse work from other servers by translating their code to
Go. Translating code is still copying it under copyright law, so every port
must follow these rules.

## Allowed sources

| Project | Language | License | Good for | May port into |
|---|---|---|---|---|
| [Dragonfly](https://github.com/df-mc/dragonfly) | Go | MIT | World, chunks, blocks, items, inventories | Any repo |
| [gophertunnel](https://github.com/Sandertv/gophertunnel) | Go | MIT | Protocol, packets, login, encryption | Any repo |
| [go-raknet](https://github.com/Sandertv/go-raknet) | Go | MIT (check before porting) | RakNet | Any repo |
| [go-nethernet](https://github.com/df-mc/go-nethernet) | Go | MIT (check before porting) | NetherNet | Any repo |
| [Minestom](https://github.com/Minestom/Minestom) | Java | Apache-2.0 | Pathfinding ideas | Any repo (keep the NOTICE) |

**Data sources** (loaded or converted, not ported as code):
[Mojang/bedrock-samples](https://github.com/Mojang/bedrock-samples),
[pmmp/BedrockData](https://github.com/pmmp/BedrockData),
[CloudburstMC/Data](https://github.com/CloudburstMC/Data),
[PrismarineJS/minecraft-data](https://github.com/PrismarineJS/minecraft-data).
Check each one's license file before adding it to `helix-data`.

Any project not in this table needs approval from 2 core maintainers before
porting. Licenses change: check the source's current `LICENSE` at the commit you
port from.

## Never port

- **Decompiled Mojang code** (Java Edition or Bedrock). You may read it to
  understand behaviour, then write your own implementation. Do not paste it,
  translate it line by line, or ask an AI to reproduce it.
- Code with no license (that means "all rights reserved").
- GPL, LGPL, AGPL or any other copyleft code (PowerNukkitX, Allay,
  Nukkit-MOT, Lumi, PocketMine-MP, Pumpkin and the like). Copyleft licenses
  forbid Helix's noncommercial terms. You may read them to understand behaviour,
  never copy or translate them.

## How to port

1. **Pin a commit.** Port from a specific commit, not a branch.
2. **Translate, don't transliterate.** Rewrite Java class trees as Go
   components and interfaces. Keep the behaviour, not the structure.
3. **Keep the attribution.** Add a header comment to each ported file:

   ```go
   // Ported from Dragonfly (MIT):
   // https://github.com/df-mc/dragonfly/blob/<commit>/server/world/chunk/chunk.go
   ```

4. **Record it in `CREDITS.md`** in the same PR:

   | Helix path | Source project | Source path | Commit | License |
   |---|---|---|---|---|

5. **Test against vanilla, not against the source.** The source project may
   have bugs. Compare with BDS and note any differences in the PR.
6. **Label the PR `porting`.**
