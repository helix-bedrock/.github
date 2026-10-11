# Testing

Every change to Helix is tested at three levels **before a PR is opened**.
Passing Go tests is not enough: a person has to launch the server, join the
game and use the feature.

| Level | Who | Required for |
|---|---|---|
| 1. Go tests | Author (or their AI tool) | Every change |
| 2. In-game testing | **A human**, by launching and joining | Every change that affects the running server |
| 3. Load and performance testing | Author, with the Helix load tool | **Every new feature**, and every change to hot paths, ticking, networking, chunks, entities or AI |

## 1. Go tests

```bash
go vet ./...
golangci-lint run
go test -race ./...
```

- New behaviour has tests. Bug fixes include a test that fails without the fix.
- Hot paths have benchmarks (`b.ReportAllocs()`), with before and after numbers
  in the PR (`benchstat`).
- Decoders of untrusted input have fuzz tests.
- Multi-core code is tested with 1 core and with many, and must give the same
  results (goal 8 in [`API_DESIGN.md`](API_DESIGN.md)).

## 2. In-game testing (required, done by a human)

**Do not open a PR until a person has launched the server, joined with a real
Minecraft Bedrock client, and tested the change in the game.** AI tools cannot
do this step and cannot tick its box.

1. Build and start the server from your branch.
2. Join with a Bedrock client on the version Helix supports.
3. Test the new feature or fix the way a player would use it:
   - the normal case;
   - edge cases (inventory full, chunk borders, dying mid-action, relogging,
     server restart and reload of saved data);
   - with more than one player when the feature involves other players.
4. Play around it for a few minutes and check that nothing nearby broke.
5. For gameplay, compare with the same scenario on Bedrock Dedicated Server
   (BDS) and note any difference.
6. Check the server log has no new errors or warnings.
7. Write down what you tested in the PR. A short video or screenshots are
   strongly encouraged for visible features.

Changes that do not affect the running server (docs, CI, a library function not
yet used by the engine) can mark this section "N/A" with a reason.

## 3. Load and performance testing

Helix will have its own load tool (planned repo: `helix-bench`) that:

- connects many simulated players (bots) that move, build, fight and trade;
- runs repeatable scenarios (for example "200 players spread over a world",
  "5,000 mobs in loaded chunks", "redstone clock farm");
- records tick time (average and worst), memory, garbage collection pauses,
  CPU use per core, and network bandwidth;
- compares a branch against `main` and reports the difference.

Until the tool exists, use Go's `pprof` and benchmarks and describe your manual
load test in the PR.

A PR that makes a scenario measurably slower needs a maintainer to approve the
trade-off explicitly.

## Release testing

Before every release, maintainers run a full play session on the release
candidate: survival from a new world, the Nether and the End, villages and
trading, redstone, and a load test, on both the `vanilla` and `minimal`
presets.
