# Governance

How decisions are made in the `helix-bedrock` organization.

## Roles

| Role | Who | Can do |
|---|---|---|
| **Project lead** | See [`MAINTAINERS.md`](MAINTAINERS.md) | Final say on direction, API design, releases, licenses and disputes. Appoints and removes maintainers |
| **Core maintainers** | See [`MAINTAINERS.md`](MAINTAINERS.md) | Approve API, license and dependency changes; review and merge anywhere; create repos |
| **Maintainers** | See [`MAINTAINERS.md`](MAINTAINERS.md) | Review and merge in the areas they own (`CODEOWNERS`) |
| **Contributors** | Everyone who has had work merged | Open issues and PRs; listed in [`CONTRIBUTORS.md`](CONTRIBUTORS.md) |

## How decisions are made

1. Most decisions happen in PR review, following [`CONTRIBUTING.md`](CONTRIBUTING.md).
2. Bigger changes (API, architecture, new repos) go through an RFC, as in
   [`API_DESIGN.md`](API_DESIGN.md).
3. When people disagree, maintainers discuss it in the open and try to reach
   agreement. If they cannot, the project lead decides.

## Becoming a maintainer

Contributors who have had steady, good-quality work merged in an area, review
other people's PRs well, and follow the project's rules can be invited by the
project lead to become a maintainer of that area. Core maintainers are chosen
from maintainers the same way.

## Stepping down

Anyone can step down at any time; they move to the "Emeritus" section of
`MAINTAINERS.md`. Maintainers who have been inactive for 6 months may be moved
there by the project lead, after asking them first.
