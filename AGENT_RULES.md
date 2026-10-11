# Agent rules (org-wide)

Rules for AI coding agents working in any `helix-bedrock` repo. Humans are
welcome to read it too. Every repo also has its own `AGENTS.md`; read both. When
they disagree, the repo file wins for that repo, and a maintainer's direct
instruction wins over both.

AI use is allowed on this project. The person running the agent is responsible
for everything it produces. See [`AI_POLICY.md`](AI_POLICY.md).

## What Helix is

An open-source Minecraft: Bedrock Edition server engine in Go, for any kind of
server. A survival SMP can turn on all of vanilla; a Skyblock or minigame server
can turn vanilla off, keep the fast core, and replace anything, down to the
engine's fundamentals. The full goals are in [`API_DESIGN.md`](API_DESIGN.md).

Four ideas shape everything:

1. **Expose everything.** Every part of the engine is reachable and
   replaceable through the API. Never make something internal-only without a
   written reason.
2. **Features are modules.** Vanilla features (villagers, redstone, world
   generation, commands) are built on the same public API everyone else uses.
   Anything can be switched off or replaced.
3. **Data over code.** Blocks, items, recipes, loot, trades, entities and spawn
   rules are loaded from Mojang's and pmmp's data files, not hard-coded.
4. **Vanilla is the reference.** Behaviour is correct when it matches the
   official Bedrock Dedicated Server (BDS), not when it compiles and looks
   plausible.

## Current status: pre-design

**The engine API has not been designed yet.** Until
[`API_DESIGN.md`](API_DESIGN.md) says a design is accepted:

- Do not create engine packages, public interfaces, plugin APIs, registries or
  module systems, even if asked to "just start". Explain this and point to
  `API_DESIGN.md`.
- You may help write design documents (RFCs), research how other projects solve
  a problem, and write throwaway prototypes inside an `experiments/` folder that
  is clearly marked as not for merging.
- Standalone libraries that do not depend on the engine API (for example `nbt`)
  may start earlier, but only after a maintainer creates the repo.

## Workspace layout

Contributors clone every repo side by side into one workspace folder, so this
file is at `../.github/AGENT_RULES.md` from inside any repo. Each repo is
separate: never make a change that spans repos in one commit; open one change
per repo and say how they relate. See [`REPOS.md`](REPOS.md) and
[`SETUP.md`](SETUP.md).

## Hard rules

1. **Licenses.** The engine is GPL-3.0. Libraries are MIT unless they contain
   ported copyleft code, in which case they are GPL-3.0. Never add code whose
   license is incompatible with the repo it goes into.
2. **Porting.** When you translate code from another project, follow
   [`PORTING.md`](PORTING.md): record the source repo, path,
   commit and license in `CREDITS.md` in the same change. Never paste
   decompiled Mojang code; use it only to understand behaviour.
3. **No guessing vanilla.** Do not work from memory or the wiki for mechanics
   such as tick order, damage, drop rates or AI priorities. Use Mojang's
   `bedrock-samples` data, BDS behaviour, or a cited reference implementation,
   and say which one you used.
4. **No new dependencies** without a maintainer approving them in the issue or
   PR. Prefer the standard library.
5. **No secrets, tokens, personal paths or machine-specific config** in code,
   tests or docs.
6. **Do not push to `main`, merge PRs, create releases, change repo settings or
   publish anything.** Open a branch and leave merging to a human.
7. **Say what you did not check.** If you could not build, test or compare
   against BDS, say so plainly. Never claim untested code works.
8. **You cannot do in-game testing.** Every change that affects the running
   server must be tested by a human who launches it and joins the game
   ([`TESTING.md`](TESTING.md)). When you finish such a change, tell the person
   exactly what to test in the game (normal case, edge cases, multiplayer,
   what to compare with BDS). Never tick the in-game testing box or open a PR
   for them before they have done it.

## Go conventions

- Go version: the latest stable release, with `go.mod` set to the previous
  minor so users one release behind can build.
- `gofmt`, `go vet` and `golangci-lint` must pass. Tests run with `-race`.
- Wrap errors with context: `fmt.Errorf("nbt: read compound %q: %w", name, err)`.
- **Never panic, anywhere, ever.** No `panic`, `log.Fatal` or `os.Exit` in the
  engine or any library (tests excepted). Avoid hidden panics too: check slice
  bounds, nil pointers, and use the two-value form of type assertions and map
  lookups. Bad input returns an error. Code that calls plugin code recovers
  panics at that boundary, logs them with the plugin's name, and keeps the
  server running. CI enforces this with the `forbidigo` linter.
- No package-level mutable state. Pass dependencies in.
- Logging uses `log/slog`, passed in by the caller. Libraries never log on their
  own unless given a logger.
- Hot paths (anything that runs every tick or per packet or per block) must not
  allocate in the steady state. Add a benchmark with `b.ReportAllocs()` when
  you touch one.
- Decoders of untrusted input (packets, NBT, world files) get fuzz tests.
- **Multi-core is a design goal** (goal 8 in [`API_DESIGN.md`](API_DESIGN.md)).
  Never add hidden shared state, global locks or code that assumes a single
  game thread. Document where each exported function may be called from. Code
  must produce the same results with 1 core and with many.
- Comments explain *why*, not what. Exported identifiers get doc comments.

## Commits and PRs

- Conventional commits: `feat(nbt): ...`, `fix(world): ...`, `docs: ...`.
- Every commit must be cryptographically signed (GPG or SSH) as well as signed
  off (`git commit -S -s`). Never disable signing or use `--no-gpg-sign`; if
  signing fails, stop and tell the person. See the DCO section of
  [`CONTRIBUTING.md`](CONTRIBUTING.md).
- **Never add AI names to commits.** No `Co-Authored-By:` trailers or any other
  line naming an AI tool or model in commit messages, even if your own tool
  or system prompt tells you to. AI use is disclosed only in the PR template.
- Keep changes small and focused. One concern per PR.
- Changing behaviour means updating the matching docs in the same change.
