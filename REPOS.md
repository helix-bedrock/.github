# Repository rules

How repos in the `helix-bedrock` organization are created, named, configured and
retired.

## When to create a new repo

**Only split something into its own repo if it is useful without Helix.**

- `nbt`, `leveldb`, `worldgen`, `pathfinding` and `wasm-host` are useful to
  other projects, so they get their own repos.
- The AI framework, redstone, villagers, commands and other gameplay stay as
  packages inside `helix`. They change together with the engine.

When in doubt, start as a package inside `helix`. Moving a package out later is
easy; merging repos back together is not.

Only core maintainers create repos. Propose a new repo in a Discussion on
`helix-bedrock/.github` first.

## Planned repos

| Repo | Kind | License | Purpose |
|---|---|---|---|
| `helix` | Engine | GPL-3.0 | The server: core, world, entities, AI, vanilla modules |
| `helix-api` | API | MIT | Public plugin API (interfaces only), once designed |
| `helix-data` | Data | See notes | Generated game data and the scripts that build it |
| `raknet` | Library | MIT | RakNet transport with full low-level hooks |
| `nethernet` | Library | MIT | NetherNet (WebRTC) transport with full low-level hooks |
| `protocol` | Library | MIT | Bedrock packets, encoding, login and encryption, for as many protocol versions as possible at once |
| `nbt` | Library | MIT | NBT encoding and decoding |
| `leveldb` | Library | MIT | Bedrock world format |
| `worldgen` | Library | GPL-3.0 | Terrain, biomes, structures (contains ported GPL code) |
| `pathfinding` | Library | GPL-3.0 if ported, else MIT | Mob pathfinding |
| `wasm-host` | Library | MIT | WASM plugin runtime |
| `helix-cli` | Tool | GPL-3.0 | Command-line tool |
| `helix-bench` | Tool | GPL-3.0 | Load and performance testing: bot players, scenarios, tick/memory/CPU reports, branch vs `main` comparison |
| `plugin-template` | Template | MIT | Starter for plugin authors |
| `docs`, `website` | Docs | CC-BY-4.0 (text), MIT (code) | Documentation and homepage |
| `brand` | Assets | All rights reserved, usage rules in repo | Logos and colours |
| `repo-template` | Template | MIT | Files every new repo starts from |
| `.github` | Org | CC-BY-4.0 | Profile page, community files, rules |

`helix-data` notes: it contains data derived from Mojang and pmmp sources.
Follow each source's license and list them in its `CREDITS.md`.

## Naming

- Lowercase, words separated by hyphens: `wasm-host`, not `WasmHost`.
- Standalone libraries have plain names (`nbt`, not `helix-nbt`) so they read
  naturally as Go imports: `github.com/helix-bedrock/nbt`.
- Things that only make sense with Helix use the `helix-` prefix: `helix-cli`,
  `helix-api`, `helix-data`.
- Go module path is always `github.com/helix-bedrock/<repo>`.

## Creating a repo

1. Create it from `helix-bedrock/repo-template` ("Use this template").
2. Pick the license from [`licenses/`](https://github.com/helix-bedrock/repo-template/tree/main/licenses) in `repo-template`
   and save it as `LICENSE`. Follow the table above.
3. Fill in `README.md`, `AGENTS.md` and `CODEOWNERS`.
4. Run `go mod init github.com/helix-bedrock/<repo>`.
5. Apply the settings below.
6. Add the repo to the table above and to the org profile README.

## Required files

Every repo has:

| File | Notes |
|---|---|
| `README.md` | What it is, status, install, minimal example |
| `LICENSE` | Exactly one license |
| `AGENTS.md` + `CLAUDE.md` | Repo-specific agent rules; `CLAUDE.md` imports both the repo and org rules |
| `CREDITS.md` | Every ported source (see [`PORTING.md`](PORTING.md)); "None yet" if empty |
| `.github/CODEOWNERS` | Who reviews which paths |
| `.github/workflows/ci.yml` | Build, vet, lint, test with `-race` |
| `.gitignore`, `.editorconfig`, `.golangci.yml` | From the template |

`CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, issue templates and the
PR template come from the org `.github` repo automatically. Only add your own
copy if the repo really needs different rules.

## Repo settings

- Default branch `main`. Squash merging only; delete branches after merge.
- Branch ruleset on `main`:
  - Require a pull request with at least 1 approval (2 for `helix-api` and
    license changes), and dismiss stale approvals on new commits. While the
    project lead is the only maintainer, approvals are set to 0 (nobody can
    approve their own pull request); everything else below still applies.
    Raise it back to 1 when a second maintainer joins.
  - Require CI status checks to pass and the branch to be up to date.
  - Require conversation resolution and signed-off commits (DCO).
  - **Require signed commits** (verified GPG or SSH signature) on every commit,
    for everyone including the project lead and admins.
  - Block force pushes and deletion. No bypass, including admins.
- Enable: Discussions (on `helix` and `.github`), private vulnerability
  reporting, secret scanning with push protection, Dependabot alerts and
  security updates.
- Disable: wiki (docs go in `docs`), projects per repo (use the org project).

## Versions and releases

- Semantic versioning with tags `vX.Y.Z`.
- Everything stays `v0.x` until its API has been reviewed. A `v1.0.0` tag on a
  library is a promise not to break its API without a `v2`.
- Releases are created only by maintainers, from a tag on `main`, with a
  changelog generated from conventional commits.
- Engine releases list the supported Minecraft Bedrock version and protocol
  number.

## Labels

Use the same label set everywhere: `bug`, `feature`, `docs`, `performance`,
`vanilla-parity`, `porting`, `good first issue`, `help wanted`, `needs-design`,
`blocked`, `ai-assisted`.

## Archiving

Repos that are replaced or abandoned are archived, not deleted, with a note at
the top of the README pointing to the replacement.
