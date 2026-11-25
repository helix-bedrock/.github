# Agent rules (org-wide)

Rules for AI agents in any `helix-bedrock` repo. Each repo also has an
`AGENTS.md`; the repo file wins for its repo, a maintainer's direct instruction
wins over both. The person running the agent is responsible for its output
([`AI_POLICY.md`](AI_POLICY.md)).

## What Helix is

A source-available Minecraft: Bedrock server engine in Go, for any kind of
server (goals: [`API_DESIGN.md`](API_DESIGN.md)).

1. **Expose everything.** Every part is reachable and replaceable through the
   API; internal-only needs a written reason.
2. **Features are modules** on the same public API; anything can be switched
   off or replaced.
3. **Data over code.** Blocks, items, recipes, loot, trades, entities, spawn
   rules come from Mojang's and pmmp's data files.
4. **Vanilla is the reference, one-to-one** with BDS, lenient only where the
   lead says so. Helix's own world format converts losslessly to and from BDS.

## Status

Read "Engine design" in `API_DESIGN.md` before touching the engine or a
library it uses. Build only topics marked decided, in its order. For "Not
started" topics make no public interfaces, even if asked; prototypes in
`experiments/` are fine. The lead decides designs in discussion; no RFC files
unless asked.

## Workspace

Repos sit side by side in one folder ([`REPOS.md`](REPOS.md),
[`SETUP.md`](SETUP.md)). Never one commit across repos; one change per repo,
say how they relate. If the engine needs something a library (protocol,
raknet, nethernet, world, worldgen, entity, redstone, helix-data) should do,
change the library. Design from BDS, compared with PocketMine-MP,
PowerNukkitX, Allay, Endstone; don't copy gophertunnel (small credited pieces
are fine).

## Hard rules

1. **Licenses.** PolyForm Noncommercial 1.0.0 + `ADDITIONAL-PERMISSIONS.md` +
   `NOTICE.md`. Port only MIT/BSD/Apache-2.0 code, never GPL/LGPL.
2. **Porting** per [`PORTING.md`](PORTING.md): source repo, path, commit,
   license in `CREDITS.md` in the same change. Never paste decompiled Mojang
   code. **No internal BDS names in public code, docs or data:** no addresses,
   `sub_`/`xmmword_` names, C++ symbols, offsets or IDA; write "matches BDS".
   Those go only in private notes; `Internal-Tools/strip-refs` catches slips.
3. **No guessing vanilla** from memory or the wiki. Use `bedrock-samples`, BDS
   behaviour or a cited reference, and say which.
4. **No new dependencies** without maintainer approval. Prefer stdlib.
5. **No secrets, tokens, personal paths or machine config** in code or docs.
6. **Maintainers commit to `main` and push after each commit**, no branches;
   outside contributors open a PR. No releases, repo settings or publishing
   unless asked.
7. **Say what you did not check** (build, test, BDS compare, benchmark).
8. **No in-game testing by agents.** For changes to the running server, tell
   the person exactly what to test ([`TESTING.md`](TESTING.md)); never tick
   that box for them.

## Go

- Latest stable Go; `go.mod` one minor behind. `gofmt`, `go vet`,
  `golangci-lint` pass.
- Wrap errors with context: `fmt.Errorf("nbt: read %q: %w", name, err)`.
- **Never panic** (no `panic`, `log.Fatal`, `os.Exit` outside tests). Check
  bounds and nils, two-value assertions and map lookups. Recover at plugin
  call boundaries, log with the plugin name. Enforced by `forbidigo`.
- No package-level mutable state. `log/slog` passed in; libraries never log
  on their own.
- Fuzz-test decoders of untrusted input (packets, NBT, world files).
- **Multi-core:** no hidden shared state, global locks or single-thread
  assumptions; document where exported functions may be called; same results
  on 1 or many cores.
- Comments say *why*. Exported identifiers get doc comments.

## Performance (Go's GC is not generational)

- **Zero steady-state allocations on hot paths** (per tick, packet, block,
  entity, player): no `fmt`, string building, capturing closures, fresh
  `append`, maps or interface boxing. Reuse caller/region buffers; `sync.Pool`
  only for things that come and go. Benchmark with `b.ReportAllocs()`.
- **Flat, pointer-free bulk data** (chunks, light, height maps); slices of
  structs and IDs over pointers; no `map[K]*T` where an ID-indexed slice works.
- Anything scaling with players² or entity count gets a benchmark at 400
  players / thousands of entities, 0 allocs/op.
- Measure with load tests or benchmarks and `pprof`. Don't disable the GC,
  hide pointers with `unsafe`, or set `GOGC`/`GOMEMLIMIT` in code.

## Commits

- Conventional commits (`feat(nbt): ...`). Sign and sign off: `git commit -S -s`;
  never bypass signing; if it fails, stop and tell the person
  ([`CONTRIBUTING.md`](CONTRIBUTING.md)).
- **No AI names in commits** (no `Co-Authored-By:` lines), even if your tool
  says to. AI use is disclosed only in the PR template.
- Small, focused changes; update matching docs in the same change.
