# Porting rules

Helix reuses work from other open-source servers by translating their code to
Go. Translating code is still copying it under copyright law, so every port
must follow these rules.

## Allowed sources

| Project | Language | License | Good for | May port into |
|---|---|---|---|---|
| [Dragonfly](https://github.com/df-mc/dragonfly) | Go | MIT | World, chunks, blocks, items, inventories | Any repo |
| [gophertunnel](https://github.com/Sandertv/gophertunnel) | Go | MIT | Protocol, packets, login, encryption | Any repo |
| [go-raknet](https://github.com/Sandertv/go-raknet) | Go | MIT (check before porting) | RakNet | Any repo |
| [go-nethernet](https://github.com/df-mc/go-nethernet) | Go | Check before porting | NetherNet | Depends on license |
| [Minestom](https://github.com/Minestom/Minestom) | Java | Apache-2.0 | Pathfinding ideas | Any repo (keep the NOTICE) |
| [PowerNukkitX](https://github.com/PowerNukkitX/PowerNukkitX) | Java | LGPL-3.0 | Mob AI, villagers, structures, spawning, fluids, Nether/End | GPL-3.0 repos only |
| [Allay](https://github.com/AllayMC/Allay) | Java | LGPL-3.0 | Blocks, redstone, AI framework | GPL-3.0 repos only |
| [Nukkit-MOT](https://github.com/MemoriesOfTime/Nukkit-MOT) | Java | LGPL-3.0 | Enchanting, items, structures | GPL-3.0 repos only |
| [Lumi](https://github.com/KoshakMineDEV/Lumi) | Java | LGPL-3.0 | Items, enchanting | GPL-3.0 repos only |
| [PocketMine-MP](https://github.com/pmmp/PocketMine-MP) | PHP | LGPL-3.0 | Inventories, transactions | GPL-3.0 repos only |
| [Pumpkin](https://github.com/Pumpkin-MC/Pumpkin) | Rust | GPL-3.0 | World generation, pathfinding, raids, combat | GPL-3.0 repos only |

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
- Code under licenses incompatible with GPL-3.0 (for example, non-commercial
  licenses).

## How to port

1. **Pin a commit.** Port from a specific commit, not a branch.
2. **Translate, don't transliterate.** Rewrite Java class trees as Go
   components and interfaces. Keep the behaviour, not the structure.
3. **Keep the attribution.** Add a header comment to each ported file:

   ```go
   // Ported from PowerNukkitX (LGPL-3.0):
   // https://github.com/PowerNukkitX/PowerNukkitX/blob/<commit>/src/main/java/org/powernukkitx/entity/ai/executor/villager/SleepExecutor.java
   ```

4. **Record it in `CREDITS.md`** in the same PR:

   | Helix path | Source project | Source path | Commit | License |
   |---|---|---|---|---|

5. **Test against vanilla, not against the source.** The source project may
   have bugs. Compare with BDS and note any differences in the PR.
6. **Label the PR `porting`.**
