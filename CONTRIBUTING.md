# Contributing to Watchcat

Thanks for your interest in Watchcat. This guide explains how to propose changes and what we expect from contributions.

By participating you agree to follow the [Code of Conduct](CODE_OF_CONDUCT.md).

## Ways to contribute

- **Report bugs** using the bug report issue form.
- **Report detection problems** (false positives or missed offences) using the detection feedback form.
- **Suggest features** using the feature request form, or start a discussion for larger ideas.
- **Improve documentation**, including the research report in `docs/research/`.
- **Write code** once the relevant roadmap phase is open. Check open issues labelled `good first issue` or `help wanted`.

For anything larger than a small fix, please open an issue first so the approach can be agreed before you invest time.

## Ground rules

These rules exist because Watchcat deals with moderation, real players and other people's intellectual property. Pull requests that break them will not be merged.

### 1. Independent work only

- Do not contribute code, text, figures, prompts, model weights or data copied from Melonly, Watchdog or any other proprietary or non-permissively licensed source.
- Ideas and published methods may be reimplemented in your own words and code. Cite papers where relevant.
- Third-party code must be under a licence compatible with MIT (for example MIT, BSD or Apache-2.0) and must keep its licence notice.

### 2. Protect player privacy

- Never include real Roblox usernames, user IDs, avatar images, chat logs or server telemetry in issues, pull requests, test fixtures or documentation.
- Use synthetic or clearly anonymised data in tests and examples, for example `Player1:1000001`.
- Do not contribute data collected from a server unless you have the permission of that server's owner, and even then only in anonymised form.

### 3. Never commit secrets

- ER:LC server keys, global API keys, Discord bot tokens and similar credentials must never appear in the repository, in issues or in CI logs.
- Use environment variables and the `.env` file, which is git-ignored.
- If you accidentally expose a key, rotate it immediately and tell the maintainers privately (see [SECURITY.md](SECURITY.md)).

### 4. Safety-sensitive changes need extra care

Changes that affect **automatic sanctions**, detection thresholds or the autonomous-action gate must:

- keep autonomous action **off by default**;
- include tests covering the gate's failure modes;
- explain the expected effect on false positives in the pull request description;
- keep the rule that automated sanctions are excluded from player-history features.

### 5. Respect the ER:LC API terms

Code must follow the [ER:LC API acceptable use policy](https://apidocs.erlc.gg/policies/aup.md) and [rate limits](https://apidocs.erlc.gg/rate-limits.md): respect rate-limit headers, back off on `429`, never rotate IPs or use proxies to evade limits.

### 6. Contributor anonymity

You may contribute under a pseudonym. Do not try to identify other contributors or maintainers, and do not post anyone's personal information.

## Development workflow

1. Fork the repository and create a branch from `main` with a descriptive name, for example `fix/incident-merge-gap`.
2. Make focused commits with clear messages. [Conventional Commits](https://www.conventionalcommits.org/) prefixes (`feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`) are encouraged.
3. Add or update tests and documentation alongside code changes.
4. Run the linters and tests locally (commands will be documented here once code lands).
5. Open a pull request using the template and fill in every section.

To keep your email private, enable **Keep my email addresses private** in your GitHub email settings and commit with your `@users.noreply.github.com` address.

## Pull request review

- A maintainer will review your pull request. Please be patient; this is a volunteer project.
- Reviews may ask for changes. Push follow-up commits to the same branch.
- Pull requests are squash-merged into `main`.
- CI must pass before merge.

## Licence of contributions

Watchcat is licensed under the [MIT License](LICENSE). By submitting a contribution you confirm that you have the right to submit it and agree that it is licensed under the same terms ("inbound = outbound").
