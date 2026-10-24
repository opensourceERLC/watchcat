# Governance

Watchcat is a small, volunteer-run project.

## Roles

- **Maintainers** review and merge pull requests, triage issues, manage releases, and enforce the [Code of Conduct](CODE_OF_CONDUCT.md). Maintainers are listed in [`.github/CODEOWNERS`](.github/CODEOWNERS).
- **Contributors** are anyone who opens an issue, joins a discussion, or submits a pull request.

## Decision making

- Day-to-day decisions are made by maintainers through pull request review.
- Significant changes (new offence classes, changes to the autonomous-action gate, data collection, licensing, or project scope) are proposed in an issue or discussion and left open for comment for at least **7 days** before a decision.
- Maintainers aim for consensus. If consensus cannot be reached, the maintainers decide by majority, and a tie keeps the status quo.

## Non-negotiables

The following can only be changed with a documented public discussion:

- autonomous action remains **off by default**;
- automated sanctions are never fed back into player-history features;
- the project does not collect telemetry from self-hosted instances;
- the project does not host a service on behalf of communities.

## Becoming a maintainer

Contributors with a sustained record of high-quality contributions and reviews may be invited by the existing maintainers. Maintainers may participate under a pseudonym.

## Stepping down

Maintainers may step down at any time by opening a pull request that removes them from `CODEOWNERS`. Inactive maintainers (no activity for 12 months) may be moved to emeritus status.
