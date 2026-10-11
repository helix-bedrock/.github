# Contributing to Helix

Thanks for helping. These rules apply to every repo in the `helix-bedrock`
organization unless a repo's own `CONTRIBUTING.md` says otherwise.

## Before you start

- **Small fixes** (typos, obvious bugs, missing tests): open a PR directly.
- **Anything bigger** (new feature, new package, behaviour change, new
  dependency): open an issue first and wait for a maintainer to agree on the
  approach. This saves you from writing code that cannot be merged.
- **Engine API work** is on hold until the design is accepted. See
  [`API_DESIGN.md`](API_DESIGN.md).
- **Ported code** must follow [`PORTING.md`](PORTING.md).
- **AI-assisted work** is welcome and must follow [`AI_POLICY.md`](AI_POLICY.md).

## Workflow

1. Fork the repo (or create a branch if you are a member).
2. Name the branch `type/short-description`, for example `fix/nbt-long-array`
   or `feat/villager-trading`.
3. Make focused commits using
   [Conventional Commits](https://www.conventionalcommits.org/):
   `feat(scope): ...`, `fix(scope): ...`, `docs: ...`, `test: ...`,
   `refactor: ...`, `perf: ...`, `chore: ...`.
4. Sign off **and** cryptographically sign every commit (see "Signed commits"
   and DCO below). This applies to everyone, including the project lead.
5. **Test it in the game.** A person launches the server, joins with a real
   Bedrock client and tests the change. New features and performance-sensitive
   changes are also load-tested. Do this before opening the PR, not after. See
   [`TESTING.md`](TESTING.md).
6. Open a PR against `main` and fill in the template, including what you tested
   in the game.
7. CI must pass. Address review comments with new commits; do not force-push
   during review unless asked.
8. A maintainer squash-merges when approved and adds or updates the author in
   [`CONTRIBUTORS.md`](CONTRIBUTORS.md). Reviewers may also join a test
   server to check the change themselves.

PRs without in-game testing are closed until it has been done.

## Review requirements

| Change | Approvals needed |
|---|---|
| Docs, tests, small fixes | 1 maintainer |
| New features, behaviour changes | 1 maintainer + the area owner (`CODEOWNERS`) |
| Public API, plugin API, data formats, licenses, new dependencies | 2 core maintainers |

## Quality bar

- Code is formatted (`gofmt`), vetted (`go vet`) and lint-clean
  (`golangci-lint run`).
- New behaviour has tests. Bug fixes include a test that fails without the fix.
- Gameplay behaviour matches vanilla Bedrock. Say in the PR how you checked
  (BDS comparison, `bedrock-samples` data, or a cited reference
  implementation).
- A human has launched the server, joined the game and tested the change.
- Every new feature shows its performance: benchmarks and a load test compared
  to `main`.
- Docs are updated in the same PR when behaviour changes.

## Signed commits

Every commit in every `helix-bedrock` repo must carry a verified GPG or SSH
signature, so GitHub shows it as **Verified**. Unsigned commits cannot be merged.
There are no exceptions, including for the project lead and maintainers. Set it
up once with the steps in [`SETUP.md`](SETUP.md#signing-commits), then commit with:

```bash
git commit -S -s -m "fix(nbt): handle empty lists"
```

`-S` signs the commit; `-s` adds the DCO sign-off below. With signing turned on
in your Git config, `-S` happens automatically.

## Developer Certificate of Origin (DCO)

Every commit must be signed off:

```bash
git commit -s -m "fix(nbt): handle empty lists"
```

This adds `Signed-off-by: Your Name <you@example.com>` and certifies that you
have the right to submit the code under the project's license, as described at
<https://developercertificate.org/>. For AI-assisted commits, the human signing
off is certifying this for the whole commit.

## Licensing

- The engine (`helix`) and anything containing ported GPL/LGPL code is
  **GPL-3.0**.
- Standalone libraries are **MIT** unless they contain ported copyleft code.
- By contributing you agree your contribution is licensed under the license of
  the repo you contribute to.

## Getting help

Ask in GitHub Discussions on the relevant repo. Please do not open issues for
questions.
