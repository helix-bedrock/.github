## What and why

<!-- What does this change, and why? Link the issue: "Closes #123". -->

## Go tests

<!-- Tests added, commands run, benchmarks (before/after with benchstat). -->

## In-game testing (required)

<!-- A human must launch the server, join with a real Bedrock client and test the change
     BEFORE opening this PR. See https://github.com/helix-bedrock/.github/blob/main/TESTING.md
     Write "N/A: <reason>" only if the change cannot affect the running server. -->

- Tested by: <!-- GitHub username of the person who joined -->
- Client version and platform: <!-- e.g. 1.26.60, Windows -->
- Preset: <!-- vanilla / minimal / custom -->
- What I tried (normal case and edge cases):
- Multiplayer tested: <!-- yes / no / not relevant -->
- Server log clean: <!-- yes / no -->
- Video or screenshots: <!-- optional, strongly encouraged -->

## Vanilla parity

<!-- For gameplay changes: how did you confirm it matches Bedrock?
     BDS comparison / bedrock-samples data / cited reference. Write "N/A" otherwise. -->

## Performance

<!-- Required for every new feature and every change to hot paths, ticking, networking, chunks, entities or AI.
     Show how your feature performs: load-test scenario and results (tick time, memory, GC, CPU per core),
     compared to main. Write "N/A: <reason>" only for docs, CI or pure refactors with no runtime change. -->

## Checklist

- [ ] Every commit is signed (shows **Verified**) and signed off (`git commit -S -s`), and follows Conventional Commits
- [ ] `gofmt`, `go vet`, `golangci-lint` and `go test -race ./...` pass
- [ ] Tests added or updated
- [ ] **A human launched the server, joined the game and tested this change** (or it is N/A, with the reason above)
- [ ] Performance of the new feature is shown above (load test compared to `main`)
- [ ] Docs updated if behaviour changed
- [ ] Ported code is recorded in `CREDITS.md` ([porting rules](https://github.com/helix-bedrock/.github/blob/main/PORTING.md)), or there is none
- [ ] No new dependencies, or a maintainer approved them in the issue

## AI assistance

- [ ] This PR used AI tools. Tool(s): <!-- e.g. Claude Code -->
- [ ] I have read and understand every line, and I can explain it in review ([AI policy](https://github.com/helix-bedrock/.github/blob/main/AI_POLICY.md))
