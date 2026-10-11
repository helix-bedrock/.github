# Setup

Part 1 is for every contributor. Part 2 is a one-time checklist for org owners.

## Part 1: Developer setup

### Tools

| Tool | Version | Why |
|---|---|---|
| [Go](https://go.dev/dl/) | Latest stable (1.27 at the time of writing) | Everything is Go |
| Git | 2.40+ | Version control |
| [golangci-lint](https://golangci-lint.run/) | Latest | Linting, same config as CI |
| [GitHub CLI](https://cli.github.com/) (`gh`) | Latest | Cloning, PRs |
| `benchstat` | `go install golang.org/x/perf/cmd/benchstat@latest` | Comparing benchmarks |
| Bedrock Dedicated Server | Latest release | Checking behaviour against vanilla |
| Minecraft Bedrock client | Same version as BDS | Joining test servers |

### Workspace layout

Clone the repos you work on side by side into one folder. Agent rules and
cross-repo links rely on this layout.

```text
Helix/
├── .github/        helix-bedrock/.github  (rules, always clone this)
├── brand/          helix-bedrock/brand
├── repo-template/  helix-bedrock/repo-template
├── helix/          helix-bedrock/helix    (once it exists)
└── nbt/ ...        libraries
```

```bash
gh repo clone helix-bedrock/.github Helix/.github
```

Repeat for each repo you need.

### Linking local libraries while developing

When you change a library and the engine together, use a Go workspace so the
engine builds against your local copy. Create it in the workspace folder; it is
never committed.

```bash
go work init ./helix ./nbt
```

### Signing commits

Every commit must be signed (see `CONTRIBUTING.md`). The simplest way is with an
SSH key:

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

Then add the same public key on GitHub under **Settings → SSH and GPG keys →
New SSH key**, with key type **Signing key**. Check it works: push a commit
and GitHub should show **Verified** next to it.

### Before every PR

```bash
gofmt -l .
go vet ./...
golangci-lint run
go test -race ./...
```

### Vanilla parity testing

Keep a BDS install outside the workspace. Compare behaviour by running the same
scenario on BDS and on Helix with the same seed, and note the result in your
PR. Never commit BDS files; its license does not allow redistribution.

### AI tools

Agents read `AGENTS.md` in the repo and `../.github/AGENT_RULES.md`. Claude
Code reads `CLAUDE.md`, which imports both. Follow
[`AI_POLICY.md`](AI_POLICY.md).

## Part 2: Organization setup (owners, one time)

### Organization settings

- [ ] Profile: avatar `brand/helix-avatar` exported as PNG (at least 500×500),
      display name "Helix", description, website, Discord link.
- [ ] Require two-factor authentication for all members.
- [ ] Base member permission: **Read**.
- [ ] Members cannot create repositories, change visibility, delete or transfer
      repos.
- [ ] Enable for all new repos: Dependabot alerts and security updates, secret
      scanning with push protection, private vulnerability reporting.
- [ ] Turn on GitHub Discussions on `.github` (org-wide discussions).
- [ ] Install the [DCO GitHub App](https://github.com/apps/dco) on all repos so
      unsigned commits fail the check, and add it as a required status check.
- [ ] Create an org-level GitHub Project for the roadmap.

### Teams

| Team | Permission | Who |
|---|---|---|
| `owners` | Org owner | 2–3 founders. Never just one person |
| `core` | Admin on all repos | Core maintainers; approve API, license and dependency changes |
| `maintainers` | Maintain on assigned repos | Area owners named in `CODEOWNERS` |
| `contributors` | Triage | Regular contributors who label and close issues |

### Repos to create first

1. `.github`, public, from this folder. Gives the org its profile page and
   default community files.
2. `repo-template`, public, then mark it as a **template repository** in its
   settings.
3. `brand`, public.

Then apply the repo settings in [`REPOS.md`](REPOS.md) to each one and create
the shared labels.

### Org-wide rulesets

Prefer organization rulesets (Settings → Repository → Rulesets) over per-repo
branch protection, so new repos get the `main` rules from
[`REPOS.md`](REPOS.md) automatically.

### Fill in the TODOs

Search this repo for `TODO` (for example, the conduct contact email in
`CODE_OF_CONDUCT.md`) and replace them before announcing the project.
