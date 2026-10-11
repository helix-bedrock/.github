# AI policy

Helix allows AI tools (coding assistants, agents, chat models) for code, tests,
docs and research. These rules keep that safe for the project and fair to
reviewers.

AI agents themselves follow [`AGENT_RULES.md`](AGENT_RULES.md)
and the `AGENTS.md` in each repo.

## The rules

1. **You are responsible.** Whoever opens the PR owns every line in it, exactly
   as if they had typed it. "The AI wrote it" is never an excuse for a bug, a
   license problem or a security issue.
2. **Understand what you submit.** You must be able to explain and defend every
   change in review. If you cannot explain it, do not submit it.
3. **Disclose it, in the PR only.** Tick the AI box in the PR template and name
   the tool. **Never** add AI names to commits: no `Co-Authored-By:` trailers or
   other lines naming an AI tool or model. Commits are authored and signed by
   the human who is responsible for them. AI tools are credited in
   [`CONTRIBUTORS.md`](CONTRIBUTORS.md) instead.
4. **Verify it.** Build it, run the tests, then launch the server, join the game
   and test it yourself ([`TESTING.md`](TESTING.md)). Check gameplay against vanilla.
   AI is often confidently wrong about Minecraft mechanics; never trust its
   numbers without a source.
5. **Keep it small.** No large AI-generated PRs that nobody has reviewed line
   by line. Split big work into reviewable pieces.
6. **Respect licenses.** AI tools can reproduce code from their training data.
   If output looks like it came from another project, find the source and
   follow [`PORTING.md`](PORTING.md), or rewrite it. Never ask an AI to
   reproduce decompiled Mojang code.
7. **Protect secrets.** Do not paste tokens, private keys, server credentials
   or other people's personal data into AI tools.
8. **No AI spam.** Mass-generated issues, PRs or review comments will be closed,
   and repeat offenders blocked.

## What reviewers do

Reviewers may ask any contributor to explain a change in their own words.
Reviewers may close AI-assisted PRs that the author clearly has not read or
tested.

## Agents running on their own

Autonomous agents may open branches and draft PRs only when a human maintainer
started them and will review the result. Agents never merge, release, change
settings or publish.
