# Watchcat

Open-source, self-hosted rule enforcement for **Emergency Response: Liberty County (ER:LC)** private servers.

Watchcat reads live server telemetry from the official ER:LC API, flags likely rule breaks (random deathmatch, mass RDM, staff evasion, inappropriate avatars and more), groups related events into incidents, and routes each case to the staff member best placed to handle it. Automatic action is strictly opt-in, narrowly gated, and off by default.

> **Status: pre-alpha / research stage.** There is no runnable code yet. The design is being worked out in the open. See the [research report](docs/research/watchdog-research-report.md) and the [roadmap](#roadmap).

## Goals

- **Self-hosted.** Each community runs its own instance with its own ER:LC server key. The project hosts nothing and collects no telemetry.
- **Human-first.** Detections go to staff with a proposed sanction. Staff decide.
- **Safe automation.** Autonomous action is opt-in, limited to the clearest offence classes, and never feeds back into the history it is judged on.
- **Transparent.** Every rule, feature and threshold lives in this repository and is configurable per community.
- **Privacy-respecting.** Minimal data collection, configurable retention, no cross-community player records.

## Roadmap

| Phase | Focus |
|---|---|
| 0 | Ingestion: rate-limit-aware ER:LC API poller, event store, webhook receiver |
| 1 | Rule-based detection, incident merging, case routing, Discord alerts with Accept/Reject |
| 2 | Label capture, blind-audit tooling, offline evaluation harness |
| 3 | Learned behaviour models (gradient boosting) trained per instance, in shadow mode |
| 4 | Avatar screening with an open vision-language encoder |
| 5 | Carefully gated autonomous moderation |

Details, risks and trade-offs are in [section 11 of the research report](docs/research/watchdog-research-report.md#11-development-roadmap).

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) and the [Code of Conduct](CODE_OF_CONDUCT.md) first. For help, see [SUPPORT.md](SUPPORT.md). To report a vulnerability, follow [SECURITY.md](SECURITY.md). Do not open a public issue.

## Disclaimer

Watchcat is an independent open-source project. It is not affiliated with, endorsed by, or derived from the code of Melonly Ltd, Police Roleplay Community (PRC) or Roblox Corporation. "ER:LC" and "Roblox" are the property of their respective owners.

The design draws on ideas described publicly in Melonly's Watchdog technical report, which is cited as prior art. No proprietary code, text, models or data have been used.

Watchcat is provided without warranty. Server operators are responsible for how they configure and use it, including any automated sanctions and their handling of players' personal data.

## Licence

[MIT](LICENSE) © 2026 Watchcat contributors
