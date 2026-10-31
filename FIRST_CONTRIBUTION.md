# Your first contribution, step by step

A short walk-through from an empty computer to a merged change. Each step links to the full rules
when you need more detail.

## 1. Install the tools

- **Go** (latest stable), **Git** and the **GitHub CLI** (`gh`). See the tool table in
  [`SETUP.md`](SETUP.md#tools).
- **An editor for Go:** [VS Code](https://code.visualstudio.com/) with the Go extension, or
  [GoLand](https://www.jetbrains.com/go/).
- **golangci-lint**, the same linter CI runs.
- **A decompiler** to read how BDS works: [Ghidra](https://ghidra-sre.org/) (free) or
  [IDA Pro](https://hex-rays.com/ida-pro).
- **Minecraft: Bedrock Edition**, the same version as BDS, to test in game.

## 2. Set up signed commits

Every commit must be signed and signed off. Set it up once with the steps in
[`SETUP.md`](SETUP.md#signing-commits); a test commit should show **Verified** on GitHub.

## 3. Clone the workspace

Clone the repos side by side into one folder, always with `.github`:

```bash
gh repo clone helix-bedrock/.github Helix/.github
gh repo clone helix-bedrock/helix Helix/helix
```

Add the libraries you change (`protocol`, `world`, `worldgen`, ...) the same way, and link them
with a `go work init` in the workspace folder ([`SETUP.md`](SETUP.md#linking-local-libraries-while-developing)).

## 4. Get BDS

Download the Linux Bedrock Dedicated Server from
[minecraft.net](https://www.minecraft.net/en-us/download/server/bedrock), accepting Mojang's terms
yourself. Keep it **outside** the workspace and never commit any of its files: its license does not
allow redistribution.

## 5. Pick something to work on

Look for issues labelled `good first issue` or `help wanted`, or ask in
[Discussions](https://github.com/orgs/helix-bedrock/discussions). For anything bigger than a small
fix, open an issue first and wait for a maintainer to agree on the approach.

## 6. Find out how BDS does it

Helix matches BDS one-to-one, so never work from memory or the wiki.

1. **Check Mojang's data first:** [bedrock-samples](https://github.com/Mojang/bedrock-samples) has
   blocks, items, entities, loot, recipes and spawn rules.
2. **Decompile BDS** for the rest: open the `bedrock_server` binary in Ghidra or IDA and let the
   analysis finish (it takes a while). Find the functions for your system by names and strings, and
   read what they do: the order of steps, the numbers, the random draws.
3. **Write down what you learned in your own words.** Never paste decompiled Mojang code into
   Helix; use it only to understand the behaviour.
4. **Compare with other servers** (PocketMine-MP, PowerNukkitX, Allay) if it helps, but the design
   comes from BDS. Ported code follows [`PORTING.md`](PORTING.md).

## 7. Write the change

Read the repo's `AGENTS.md` and [`CONTRIBUTING.md`](CONTRIBUTING.md#quality-bar) for its rules:
no panics, errors wrapped with context, no new dependencies without approval, tests for new
behaviour. Make the branch `type/short-description`, for example `fix/nbt-long-array`.

## 8. Check everything

1. **The code:**

   ```bash
   gofmt -l .
   go vet ./...
   golangci-lint run
   go test -race ./...
   ```

2. **Against BDS:** run the same scenario on BDS and on Helix, with the same world or seed, and
   compare the result.
3. **In the game:** start the server, join it with the Bedrock client and test the normal case, the
   edge cases and, where it matters, with more than one player. See [`TESTING.md`](TESTING.md).

## 9. Open the pull request

Commit with `git commit -S -s` and a [conventional commit](https://www.conventionalcommits.org/)
message, such as `fix(world): keep empty sub chunks`. Push, open a PR against `main` and fill in the
template: what you changed, how you checked it against BDS and what you tested in game. CI must
pass; answer review comments with new commits.

## 10. Get credited

When your PR is merged, a maintainer adds you to [`CONTRIBUTORS.md`](CONTRIBUTORS.md). Welcome to
Helix.
