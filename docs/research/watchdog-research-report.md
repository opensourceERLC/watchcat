# Research report: an open-source alternative to Melonly Watchdog

*Project: **Watchcat**, an independent, open-source, self-hosted reimplementation of the ideas described in the Watchdog paper. Research date: 24 October 2026. Primary reference: [Watchdog: Real-Time Rule Enforcement for ER:LC Roleplay Servers](https://melonly.com/research/watchdog.pdf), Melonly Ltd technical report, 21 October 2026.*

**Evidence labels used throughout:**

- **[P]** stated in the paper
- **[V]** verified against an external primary source
- **[I]** inference by the author of this report
- **[?]** unknown

---

## 0. The paper in brief

- *Watchdog: Real-Time Rule Enforcement for ER:LC Roleplay Servers*, Melonly Ltd technical report, 21 October 2026, 9 pages, no appendices. [P]
- Headline results [P]:
  - Combat incident AP 0.707, which is 2.7× the base rate.
  - Combat precision is 98.0% in the top 100 incidents and 96.3% in the top 2,500.
  - Staff-evasion lift is 25×.
  - Avatar model: ROC AUC 0.856 against a blind manual audit, and 90.0% audited precision.
  - The autonomous-action gate was right 97.2% of the time when replayed against history.
- Closing statement [P]: *"Feature definitions, thresholds, label horizons, concept prompts, model weights and training data are proprietary to Melonly Ltd and are not included in this report."*
- The paper's statistics were recomputed and are internally consistent:
  - The Wilson intervals check out: 72/80 gives 81.5–94.8%, and 37/40 gives 80.1–97.4%.
  - The lifts check out: 0.687/0.255 = 2.69 and 0.458/0.0183 = 25.0.
  - The error-rate reduction checks out: 8.5% to 2.8%.

---

## 1. System architecture

**Components [P] (Fig. 3):**

1. **Game telemetry**: kills, joins and departures, staff commands, moderator calls, and periodic world snapshots.
2. **Behaviour state**: a causal per-player history, plus a player history ledger `L` and community-norm statistics `N_s`.
3. **Detectors**: "nine offence classes".
   - Only these are named: combat (kill trigger), staff RDM, mass RDM, staff evasion (departure trigger), avatar (join trigger) and "peacetime" (snapshot trigger).
   - The remaining classes are never listed. [?]
4. **Case engine**: merges detections about the same player into a case and assigns it to a staff member.
5. **Outputs**: a web staff panel popup, in-game PMs and replies, and gated autonomous action.
6. **Melonly log**: the sanction of record.
   - Only human sanctions flow back into player history.

**Data flow [P]:**

- An event arrives. If it is a trigger, features are computed from the history before time `t` only, and the model scores it.
- Triggers are merged into an incident, then into a case.
- The case is routed to the nearest free staff member, or acted on automatically if the gate passes.
- A staff member accepts the proposed sanction, which runs through the community's normal pipeline and audit log.
- The sanction updates player history, but only if a human issued it.

**Case states [P]:** `detected → escalated → left → resolved`. A case can go from any state to resolved, and a sanction resolves it.

**Discrepancy between the paper and the product docs [V]:**

- Melonly's public documentation describes admins configuring free-text rules, "notes" about the server's vibe, and a strictness slider. It warns that more than five rules reduce accuracy, and says detection takes 5–10 minutes ([Melonly docs](https://docs.melonly.xyz/docs/Dashboard/watchdog), seen via a search snippet; the page returned 404 at its redirect target when fetched).
- The paper never mentions free-text rules or a language model interpreting them. Either part of the product is undocumented, or the docs and the paper describe different versions. [I]

---

## 2. Mathematical methods

All of this is from the paper [P]. Implementation notes are marked [I].

**Eq. 1, event stream.**

```
E_s = {(t_i, k_i, a_i, b_i, z_i)},  k ∈ {join, leave, kill, command, modcall, snapshot}
```

- Each event has a time, type, actor, target and payload (positions, teams, permissions).
- Implementation: one append-only event table, keyed by server, ordered by time. [I]

**Eq. 2–3, causal scoring.**

```
x_e = Φ(E_s,<t, L_<t, N_s)
p̂_e = σ(F(x_e)) = 1 / (1 + e^(−F(x_e)))
```

- `Φ` is a feature function that may only see the past. That rule is what prevents label leakage.
- Implementation: compute features as of time `t`, using the same code in training and at serving time. [I]

**Eq. 4–5, labels.**

- `y_e = 1[e violates a rule in family f(e)]`
- Training keeps only `K = {e : y_e is unambiguous within [t, t+H]}`.
- The horizon `H` is **not disclosed**.
- The paper never says where labels come from. [?]
  - Staff sanctions are the obvious candidate. [I]
  - If so, two problems follow:
    - **Hidden positives.** The paper's own introduction says most violations "go unseen". Unpunished violations would be labelled as negatives, which makes this a positive-unlabelled learning problem ([Elkan & Noto 2008](https://dl.acm.org/doi/10.1145/1401890.1401920)).
    - **Easy-case bias.** Keeping only "unambiguous" events selects easier cases, so every reported metric is conditional on that set.

**Eq. 6, incidents.**

- Consecutive triggers by one player less than `g` apart form an incident.
- `S(I) = max p̂_e` and `Y(I) = max y_e`.
- The gap `g` is **not disclosed**.
- Implementation: session-window grouping per actor. [I]

**Eq. 7, leak-free split.**

- Group keys are contiguous slices of one community's history.
- Each key goes to exactly one partition: test if `h(g) mod 100 < α`, validation if `< α+β`, otherwise train.
- `α` and `β` are not given.
- Implementation: hash on (server, time block). [I]

**Eq. 8–10, gradient boosting** (Friedman; XGBoost's second-order objective; LightGBM's histogram, leaf-wise growth):

```
F(x) = η Σ_m f_m(x)
J = −Σ_{e∈K}[y log p̂ + (1−y) log(1−p̂)] + Σ_m (γT_m + ½λ‖w_m‖²)
g_e = p̂_e − y_e,   h_e = p̂_e(1−p̂_e)
w_j* = −G_j/(H_j+λ)
Gain = ½[G_L²/(H_L+λ) + G_R²/(H_R+λ) − (G_L+G_R)²/(H_L+H_R+λ)] − γ
```

- This is the standard [XGBoost](https://arxiv.org/abs/1603.02754) and [LightGBM](https://papers.nips.cc/paper/6907-lightgbm-a-highly-efficient-gradient-boosting-decision-tree) mathematics. It never needs hand-writing: `objective="binary"` in either library is exactly this.
- Each detector is an **ensemble of two models with averaged probabilities**.
  - Which two is not stated. Likely one XGBoost and one LightGBM, given the citations. [I]
- Training uses early stopping on the validation partition.
- Sizes [P]:
  - Combat (settled): 109 features, 805 trees, 143 s.
  - Combat (immediate): 104 features, 955 trees, 156 s.
  - Staff evasion: 40 features, 1,193 trees, 173 s.
- All of these train in minutes on a single workstation, so compute is not the bottleneck.

**Eq. 11, history feature with a feedback guard.**

```
R_p^(w)(t) = Σ_{ℓ∈L_p} 1[t − w ≤ t_ℓ < t] · 1[author(ℓ) ∉ A_auto]
```

- Counts a player's prior sanctions inside a time window, **leaving out the system's own sanctions**.
- This prevents a loop where each automated ban makes the next automated ban more likely.
- Adopt this guard as is. It is the most transferable safety idea in the paper.

**Settled vs immediate scoring [P].**

- The settled model waits a short, undisclosed interval and adds evidence that arrives after the kill (the victim calls a moderator, the killer leaves).
- Gain: +0.010 event AP and +0.009 incident AP.
- That gain is small. An immediate-only model is adequate for a first version. [I]

**Eq. 12, avatar features.**

```
z_p = g(r_p)/‖g(r_p)‖
c_{p,k} = ⟨z_p, u_k⟩
x_p = [z_p ; c_{p,1..K}] ∈ R^(d+K)
```

- `g` is a frozen contrastive image encoder ([CLIP](https://arxiv.org/abs/2103.00020)-style), and `u_k` are text embeddings of concept prompts.
- A boosted classifier is trained on top.
- The encoder, `K` and the prompts are **undisclosed**.

**Eq. 13, Wilson score interval** ([Wilson 1927](https://doi.org/10.1080/01621459.1927.10502953)):

```
[p̂ + z²/2n ± z·sqrt(p̂(1−p̂)/n + z²/4n²)] / (1 + z²/n),   z = 1.96
```

**Eq. 14, metrics.**

- `P@k` = mean label of the top `k` incidents.
- `AP = Σ(R_n − R_{n−1})P_n`.
- `Lift = AP / π`, where π is the base rate. A random ranking has AP ≈ π.
- `sklearn.metrics.average_precision_score` computes AP directly.

**Eq. 16, routing.**

- `s* = argmin_{s∈S_t} ‖x_s(t) − x_o(t)‖²`: the nearest staff member who is present, free, and not contacted within a cooldown `q`.
- Positions older than a freshness bound count as unknown.
- **One-sanction lock**: an atomic set-if-absent, so two moderators pressing Accept at once still produce one sanction.

**Eq. 17, autonomous gate.**

```
A(e) = 1[c(e) ∈ C*] · 1[p̂_e ≥ θ_a] · 1[R_p(t) ∈ B]
```

- `C*` = staff RDM and mass RDM.
- `θ_a` is above the alerting threshold.
- `B` is a region of recent history associated with repeat offenders.
- `θ_a` and `B` are **undisclosed**.

**Statistics the paper does not report:**

- **Calibration.** No reliability diagram, no Brier score, no ECE. `p̂` is called a probability but its calibration is never shown.
  - For avatars, the paper sidesteps this by showing staff the audited precision of the score band.
- **Recall.** No recall or alert volume at the operating threshold.
- **Figure 4.** No sample sizes for the four policies.
- **The "new encoder is better" claim** rests on 36/40 vs 32/40 confirmations.
  - A two-proportion z-test gives z ≈ 1.25, p ≈ 0.2 (this report's calculation). Suggestive, not significant.
  - The AUC increase from 0.795 to 0.856 has no confidence interval.
- **Audit AUC.** The audit sampled within score bands, so a ROC AUC computed on that sample needs reweighting. Whether it was reweighted is not stated. [?]
- **Why a blind audit.** The paper cites [Menon et al. 2015](https://proceedings.mlr.press/v37/menon15.html): under label noise, a ranking metric measured on noisy labels is an affine function of the clean one. The ranking survives noise but the absolute numbers are distorted.

---

## 3. Data requirements

**Telemetry used [P] (Table 1):**

- Kills
- Joins and departures
- Staff commands
- Moderator calls
- Player records
- Avatar renders

**Scale [P]:** six months of 2026 from "the communities Watchdog monitors". Event, server and label counts are not given.

**What a comparable system needs [I]:**

| Need | Source in an independent build |
|---|---|
| Event stream | Poll the ER:LC v2 API (section 4) |
| Positions and teams | `Players` snapshots taken at polling time |
| Player history ledger | Own sanction records, plus the community's existing logs if it permits import |
| Community norms `N_s` | Sanction rate per offence per server. Needs **many servers** to be meaningful. |
| Labels | Staff Accept/Reject decisions on alerts, plus sanctions issued independently. Add a periodic blind audit for clean evaluation labels. |
| Avatar labels | Renders from the Roblox thumbnails API, labelled against each community's avatar policy |

**Cold start [I]:**

- No labelled data exists on day one, so start with rules.
- The staff UI doubles as the labelling tool: every Accept or Reject is a label. That label set is biased, because only alerted events get labelled, and the bias must be corrected for in evaluation.

---

## 4. ER:LC integration

### Verified against official documentation [V]

**API basics**

- Base URL: `https://api.erlc.gg/v2`. The old `policeroleplay.community` domain now redirects to `erlc.gg`.
- Authentication: a `server-key` header, obtained from in-game server settings. Using the API requires the paid "ERLC API" server pack.
- A public application additionally sends an `Authorization` global key. ([intro](https://apidocs.erlc.gg/), [OpenAPI spec](https://api.erlc.gg/internal/docs/apispec.v2.json))

**`GET /v2/server` (base fields always returned)**

- Base fields: `Name`, `OwnerId`, `CoOwnerIds`, player counts, `JoinKey`, `AccVerifiedReq`, `TeamBalance`.
- Optional boolean includes, each adding one field:
  - **Players**: `Team`, `Player` ("Name:Id"), `Callsign`, `Location{LocationX, LocationZ, PostalCode, StreetName, BuildingNumber}`, `Permission`, `WantedStars`.
  - **Staff**: Admins, Mods and Helpers maps, keyed by Roblox user ID.
  - **JoinLogs**: `Join`, `Timestamp`, `Player`.
  - **Queue**: a list of user IDs.
  - **KillLogs**: `Killer`, `Killed`, `Timestamp` only.
  - **CommandLogs**: `Player`, `Timestamp`, `Command`.
  - **ModCalls**: `Caller`, `Moderator` (nullable), `Timestamp`.
  - **EmergencyCalls**: team, caller, players, position, description and related fields.
  - **Vehicles**: name, owner, plate, colour and related fields.

**`POST /v2/server/command`**

- Body: `{"command": ":h Hey everyone!"}`.
- Returns 422 if the server is offline or empty.
- Rate limit: 1 request per 5 seconds (the "command" bucket, marked subject to change). ([rate limits](https://apidocs.erlc.gg/rate-limits.md))

**Location data**

- 2D only: `LocationX` and `LocationZ`, origin at map centre, Z increasing downward.
- No height axis, no units, no freshness guarantee. ([location doc](https://apidocs.erlc.gg/how-do-i-interpret-the-xyz.md))

**Event webhooks**

- Only two event types: in-game messages starting with `;`, and emergency calls.
- Signed with Ed25519 (headers `X-Signature-Ed25519` and `X-Signature-Timestamp`).
- Payload schema, retries and ordering are not documented.
- The docs state explicitly that webhooks do **not** replace polling. ([webhooks](https://apidocs.erlc.gg/event-webhooks.md))
- Still useful: the `;` messages provide a signed channel for staff to reply in game, for example `;accept`. [I]

**Rate limits**

- The global limit varies per application. The example header shows 35.
- Using many servers without authentication, or rotating IPs, returns 403 with code 2005.
- Repeated 4xx errors lead to temporary blocks.
- Shared-IP hosts such as BotGhost are not supported.
- "Large application" status requires 100+ servers. It grants the right to send commands to any server sharing its key, but no numeric limits are stated. ([large apps](https://apidocs.erlc.gg/large-applications.md))

**Acceptable use policy** ([AUP](https://apidocs.erlc.gg/policies/aup.md), [large-app AUP](https://apidocs.erlc.gg/policies/large-app-aup.md))

- Prohibited: spam, impersonation, raiding, proxies or IP rotation.
- Automation must stop when the application is removed from a server.
- Keys must never appear in public repositories, which matters for an open-source repository.
- Large applications must process data only "as strictly necessary".
- **The AUP says nothing about ML use, cross-server player records, or data retention.** [?] Ask PRC before building cross-community history.

### Gaps between the public API and the paper's features [I]

- **"Victim had been shooting first."** The public API has **no gunshot or damage events**.
  - Possible proxies: the victim recently killed someone, victim wanted stars, team pairing, or a recent emergency call.
  - Melonly may have access to non-public data, or may infer this. [?]
- **"Distance at kill" and "killer speed".** Kill logs carry **no positions**.
  - Each kill must be joined to the nearest-in-time `Players` snapshot. Accuracy depends on the polling rate, which the rate limit caps.
  - Speed comes from snapshot differences and is coarse.
- **Spawn zones.** Map units and bounds are undocumented, so spawn polygons must be calibrated by hand from observed spawn positions.
- **Log windows.** How many log entries each poll returns is undocumented. Polls overlap, so events must be deduplicated on (type, timestamp, actor, target).
- **Identity.** `Name:Id` gives a stable Roblox user ID.
  - Linking Discord staff accounts to Roblox accounts needs a separate step: Roblox OAuth, or a third-party linker.

### Moderation actions [I]

- The docs only show `:h`. In-game admin commands (`:kick`, `:ban`, `:pm`, `:jail` and so on) are community knowledge and must be verified against the current game version.
- The 1-command-per-5-seconds limit makes a command queue mandatory.
- A 422 response means the offender has left, which maps onto the paper's "hold interval, then fall back to staff" behaviour.

---

## 5. Automatic moderation

### Detecting offences with public data only [I]

All thresholds below are design parameters to be tuned on local data. The paper does not provide them.

**RDM (single random kill)**

- Scoring signals:
  - No prior interaction between killer and victim (no recent kills between them, no emergency call involving both).
  - The team pairing is illegitimate under the server's rules, for example police killing a civilian with 0 wanted stars.
  - The victim is near spawn or has just joined.
  - A victim moderator call within Δ after the kill.
  - The killer leaves soon after.

**Mass RDM**

- Count distinct victims per killer in sliding windows. Distinct victims is a better signal than raw kill count.
- Use incident grouping (Eq. 6).
- Rules alone are strong here, so this is the best first target for automation, as the paper also found.

**Staff RDM and staff abuse**

- The same signals, filtered to actors whose `Permission` is not "Normal", or who appear in the `Staff` map.
- Add staff `CommandLogs` targeting players with no open moderator call or case: kick, ban, kill, jail.
- Caveat: the paper **exempts** moderators and above from detection [P]. A staff-abuse detector therefore cannot be automated and should only notify owners.

**Staff evasion**

- Trigger: a departure while the player has an open case, an unanswered moderator call naming them, a recent staff command targeting them, or wanted stars.
- The base rate is very low (1.83% in the paper [P]), so precision collapses without good features. Treat this as a ranking problem, not a pass/fail rule.

**Peacetime violations**

- Snapshot-triggered, for example weapons or crime during an announced "peacetime".
- Requires the server's peacetime state, probably parsed from `:m` or `:h` announcements in command logs. The API does not expose weapon state. Weak. [?]

### Reducing false positives

- Per-community norms (the paper's `N_s` idea). A strict server and a relaxed server need different thresholds.
- Merge kills into incidents and score the incident, not each kill.
- Exclude staff-sanctioned events, for example a kill immediately after a staff `:kill` command.
- Keep the Eq. 11 guard so automated sanctions never feed back into history.
- Report incident-level precision, because staff experience incidents, not individual events.

### Calibration

- Fit isotonic or Platt calibration on the validation partition, never on test ([Niculescu-Mizil & Caruana 2005](https://dl.acm.org/doi/10.1145/1102351.1102430), [Guo et al. 2017](https://arxiv.org/abs/1706.04599)).
- Show staff the audited score-band precision rather than raw probabilities, as the paper does for avatars.
- Recalibrate per community, or at minimum check calibration per community.

### When automation is safe

Automate only when all of the following hold:

1. The offence class has stable, high audited precision. Start with mass RDM only.
2. Gate conditions like Eq. 17 hold. Add a minimum players-online and data-freshness condition. [I]
3. Shadow mode has run first: log what the system *would* have done and compare with staff outcomes.
4. The community has opted in, and the executing service re-checks the opt-in itself (the paper's safeguard [P]).
5. The sanction runs before the log entry is written.
6. The mildest sanction comes first: a kick or warning, never a permanent ban.
7. Appeals and a kill switch exist.

---

## 6. Avatar moderation

**Paper [P]:**

- Full-body render, a frozen contrastive vision-language encoder, `K` concept-probe similarities concatenated with the embedding, then a boosted classifier.
- Evaluated out of fold, then validated by a blind audit sampled within score bands.
- Staff see the audited band precision.
- The previous version used a smaller encoder and scored AUC 0.795.

**Obtaining renders [V]:**

- Roblox thumbnails API, `GET thumbnails.roblox.com/v1/users/avatar?userIds=…` for full-body images ([Roblox docs](https://create.roblox.com/docs/cloud/reference/features/thumbnails)).
- Check Roblox's terms regarding bulk use. [?]

**Open-source encoders** (licences checked only via secondary listings; confirm each LICENSE file before use):

| Encoder | Licence | Notes |
|---|---|---|
| OpenAI CLIP | MIT ([repo](https://github.com/openai/CLIP)) | Older and weaker |
| OpenCLIP | Code MIT ([repo](https://github.com/mlfoundations/open_clip)) | LAION-trained weights. LAION-5B had a 2023 dataset-safety withdrawal before being re-released as Re-LAION, a reputational consideration. [I] |
| SigLIP 2 | Apache-2.0 per third-party listings ([Voxel51](https://docs.voxel51.com/model_zoo/models/google_siglip2_base_patch16_224.html)) | Strong zero-shot performance. Base sizes run acceptably on CPU. **Recommended.** [I] |
| Generative vision-language models (Qwen-VL family, etc.) | Licence **varies by model size** | Too slow and costly for every join. Use only for staff-facing explanations, if at all. [I] |

**Build ladder [I]:**

1. Zero-shot: cosine similarity to prompts such as "a giant meme character" or "a headless avatar". No labels needed. Measure it with an audit.
2. Linear or logistic probe on the embedding plus concept similarities, trained on a few hundred to a few thousand labelled renders.
3. Boosted classifier, as in the paper.

**Limitations [I]:**

- Each community has its own avatar policy, so labels do not transfer across servers.
- Roblox renders are stylised and differ from the photographs these encoders were trained on.
- In-game team uniforms can override the avatar's appearance, so the profile render is not always what other players see.
- Players change avatars mid-session, and the thumbnail cache may lag.
- Renders depict accounts that frequently belong to minors, which is a privacy consideration (section 10).

---

## 7. Open-source technology stack

**Recommended for a small team or solo developer [I]:**

- **Language:** Python 3.12+ throughout. The ML ecosystem decides this.
- **Ingestion:** an `asyncio` + `httpx` poller, one task per server. It respects `X-RateLimit-*` headers and backs off on `Retry-After`.
- **Webhook receiver:** FastAPI (MIT), with Ed25519 verification via `cryptography` or PyNaCl (Apache/BSD).
- **Storage:** PostgreSQL (PostgreSQL License).
  - Event table partitioned by month.
  - The one-sanction lock is a **unique constraint** or `INSERT … ON CONFLICT DO NOTHING`. Redis is not needed for it.
- **Features and training:** Polars or DuckDB (MIT) offline, scikit-learn (BSD-3), LightGBM (MIT) and/or XGBoost (Apache-2.0).
- **Serving:** in-process LightGBM, or ONNX Runtime (MIT).
- **Avatar:** `transformers` (Apache-2.0) + SigLIP 2, on CPU.
- **Staff UI:** a Discord bot (discord.py MIT, or discord.js Apache-2.0) with Accept/Reject buttons. A web panel can follow later.
- **Operations:** a Docker Compose stack on one VPS. Prometheus + Grafana optional.

**Alternatives compared [I]:**

| Choice | Options | Verdict |
|---|---|---|
| Event store | **Postgres** / TimescaleDB (core Apache-2.0, some features under the separate TSL licence) / ClickHouse (Apache-2.0) | Postgres until hundreds of servers. ClickHouse is overkill for one developer. |
| Locks and cache | Postgres / Valkey (BSD-3) / Redis | Redis has moved to source-available licences, with an AGPL option from Redis 8. Prefer Valkey if a cache is needed. |
| Boosting | **LightGBM** / XGBoost / CatBoost (Apache-2.0) | All three are adequate. LightGBM is fastest. CatBoost handles categorical features natively. |
| Ingestion language | Python / Go / TypeScript | Go is more efficient at large scale, but splitting languages costs a solo developer more than it gains. |
| Bot language | discord.py / discord.js | Match the rest of the codebase. |
| Experiment tracking | none / MLflow (Apache-2.0) | Start with none: version datasets and models in object storage. |

**Running cost [I]:** one small VPS handles dozens of servers. CPU inference is sufficient. GPU spend is approximately zero unless a generative vision-language model is adopted.

**Licence for the project's own code [I]:** Apache-2.0 for its patent grant, or AGPL-3.0 to prevent closed SaaS forks. Decide before accepting outside contributions.

---

## 8. Reproducibility gap

**Documented in the paper [P]:**

- The formulation: event triggers, causal features, incident max-pooling.
- The model family: GBDT, a two-model average, early stopping.
- Feature *groups*: kill dynamics, victim context, spatial and kinematic, staff interaction, community norms, player history.
- The history-ledger formula with the automation-exclusion guard.
- The grouped hash split.
- The avatar architecture.
- The audit method and Wilson intervals.
- The evaluation metrics.
- Nearest-staff routing, the atomic lock, the case states.
- The form of the three-condition gate, and the four safeguards.
- Headline results.

**Undisclosed or proprietary [P]:**

- The 109, 104 and 40 feature definitions.
- `H`, `g`, the settled-wait interval, the cooldown `q`, the freshness bound, the hold interval `h`, the split fractions `α` and `β`, `θ_a` and the alerting threshold, the history region `B`.
- The concept prompts and `K`.
- The encoder identity.
- Which boosting libraries make up the pair.
- How labels are produced.
- Dataset size and which communities.
- Six of the nine offence classes.
- Model weights and training data.

**Not resolvable from the paper alone [I]:**

- Whether the reported numbers transfer to other communities.
- Whether Melonly has telemetry beyond the public API. The "shooting first" feature suggests it might.
- How the free-text rules and strictness slider in the product docs relate to these models.

**Matching the paper's numbers:** no claim of equal performance is warranted without comparable labelled data and an independent evaluation.

---

## 9. Evaluation methodology [I]

1. **Splits**
   - Group by server and time block, as in the paper.
   - Better still, **forward-chain by time**: train on months 1–4, test on month 6. Deployment always predicts the future.
   - Also hold out entire servers to measure transfer to new communities.
2. **Labels**
   - Build a gold set from a **blind audit**: sample stratified by score band *and* include random unalerted events to estimate recall.
   - Use at least two judges and measure agreement (Cohen's κ). Reweight by sampling probabilities when estimating overall precision, recall and AUC.
3. **Metrics**
   - Incident-level PR-AUC and lift.
   - Precision@k.
   - **Recall and alert volume per server-hour at the operating threshold**, which the paper omits.
   - Confidence intervals on every metric (Wilson, or bootstrap clustered by group).
4. **Calibration**
   - Reliability diagrams, Brier score and ECE on held-out data, checked per community.
5. **False punishment**
   - Measure (wrong automated sanctions) / (automated sanctions), wrong sanctions per 1,000 player-hours, and the appeal overturn rate.
   - A zero-error run of `n` cases gives a 95% upper bound of about `3/n` (the "rule of three"). Claiming ≤1% error requires about 300 consecutive correct audited actions per class.
6. **Before deployment**
   - Several weeks of shadow mode.
   - Then automated action on a small opted-in group, under a pre-registered stop rule, for example stop if the upper Wilson bound on error exceeds a set percentage.
   - Then widen gradually.
7. **Subgroup checks**
   - New accounts vs veterans, per team, per server size.
   - The system must not systematically over-punish one role, for example criminals.

---

## 10. Legal and licensing considerations [I]

*This section is not legal advice.*

**Copyright in the paper**

- The text, figures and tables belong to Melonly Ltd, and no licence is granted. Publishing a paper does not make its implementation open source.
- Copyright protects expression, not ideas, methods or mathematics (US [17 U.S.C. §102(b)](https://www.law.cornell.edu/uscode/text/17/102); UK law follows the same principle).
- Independently implementing GBDT, CLIP probes, incident max-pooling or a gated policy is permissible. Those methods mostly come from the cited literature.
- Do not copy the paper's prose, figures, or the Figure 1 avatar images into project documentation.
- Cite the paper as prior art and inspiration.

**Patents**

- No patent is mentioned in the paper. Patent databases were not searched for this report, so this is unknown. [?]
- Search Espacenet and Google Patents for Melonly filings before release.

**Trademark and naming**

- Do not use "Melonly" or "Watchdog" in branding, and do not imply affiliation with Melonly or PRC. The large-app AUP explicitly requires a third-party disclaimer.
- The project name is **Watchcat**. It echoes "Watchdog" but is a distinct mark; pair it with a clear non-affiliation disclaimer (section 13).

**Third-party licences**

- LightGBM MIT, XGBoost Apache-2.0, CLIP MIT, OpenCLIP code MIT.
- SigLIP 2 weights reportedly Apache-2.0; confirm from Google's model card.
- Generative vision-language model licences vary by size.
- Maintain a NOTICE file with Apache-2.0 attributions.

**Data protection**

- Roblox user IDs, usernames, positions and behaviour histories are personal data. Many ER:LC players are children, which brings in the UK GDPR, the ICO Children's Code, and COPPA for US users.
- Automated sanctions engage the rules on solely automated decision-making (UK GDPR Art. 22, as amended by the Data (Use and Access) Act 2025; [ICO guidance](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/automated-decision-making-and-profiling/)).
  - Whether a game kick or ban counts as "significant" is debatable. Build in notice, human review and appeals regardless.
- Data minimisation also follows from PRC's large-app rule.
- Cross-community player records are the riskiest feature: profiling across data controllers, with the AUP silent on the matter. Obtain PRC's written approval and community consent first.

**API terms**

- Each community must supply its own server key and pay for the API pack.
- Keys belong in a secrets store, never in the repository.

---

## 11. Development roadmap

| Phase | Deliverable | Depends on | Main risk |
|---|---|---|---|
| **0. Ingestion** | Rate-limit-aware poller, event normalisation and deduplication, Postgres store, identity (`Name:Id`), webhook receiver | Server keys from 1–3 partner communities | Undocumented log windows; rate-limit blocks |
| **1. Rules + cases** | Mass-RDM and RDM heuristics, incident merging, cases, nearest-staff routing, Discord alerts with Accept/Reject, one-sanction lock, per-community toggles and exemptions | Phase 0; Discord-to-Roblox account linking | Alert fatigue; per-server threshold tuning |
| **2. Labels + evaluation** | Persist every decision; blind-audit tool; offline evaluation harness (grouped and temporal splits, PR-AUC, P@k, calibration) | Several weeks of Phase 1 traffic | Feedback bias (only alerts get labelled); judge disagreement |
| **3. Learned behaviour models** | Causal feature pipeline shared between training and serving; LightGBM for combat and evasion; isotonic calibration; shadow comparison against rules | Thousands of labelled incidents across several servers | Leakage; poor transfer across communities; noisy snapshot-joined positions |
| **4. Avatar** | Zero-shot SigLIP, then a linear probe; band-precision display | Thumbnails API; a few hundred labelled renders per policy | Per-server policy variation; stylised images; minors |
| **5. Gated autonomy** | Eq. 17-style gate for mass RDM only, opt-in, the service re-checks consent, sanction before log, hold-interval fallback, automated sanctions excluded from history, kill switch, appeals | Phases 2–3 with audited precision at or above target and enough audited cases | A wrongful automated ban; legal and AUP exposure |
| **6. Expansion** | Staff evasion, staff abuse (owner notifications only), community-norm features, settled model | Many servers | Multi-server operation requires a public or large-application key |

**Hardest components, in order:**

1. Obtaining enough honest labels.
2. Features the public API cannot supply: shooting first, positions at kill time.
3. Community-norm features, which require breadth across many servers.
4. Statistically validating autonomous action.

---

## 12. Feasibility assessment

**Realistic for a solo developer:**

- Phases 0, 1, 4 (zero-shot) and the safety *framework* of Phase 5, within a few months of part-time work.
- The ML code is straightforward. The paper's mathematics is library-standard, and training takes minutes on a single workstation.

**Requires substantial data collection:**

- Learned combat and evasion models, community norms and calibrated probabilities. These need months of telemetry from many servers plus an audit programme.
- Melonly had six months of data across its customer base [P]. An independent project starts at zero.

**Blocked or uncertain:**

- Victim-aggression features (no shot events in the public API).
- Precise kill geometry.
- Multi-server operation without a public-application key.
- Cross-community history without PRC's approval.

**Priorities:**

1. Ingestion + mass-RDM rules + routing + Discord Accept/Reject, which also starts label collection.
2. Evaluation harness and blind audit.
3. Zero-shot avatar screening.
4. LightGBM combat model in shadow mode.
5. Narrowly gated automation: mass RDM only, kicks only.

**Bottom line:** the architecture and safety design can be reproduced from the paper. The performance cannot, and it should not be claimed until an independent held-out audit supports it.

---

## 13. Watchcat: open-source distribution model

*Decision: Watchcat is published as open source. The maintainers host nothing; each community self-hosts its own instance.*

### Scope of the remake

- Watchcat reimplements the **ideas** in the Watchdog paper: causal event scoring, incident grouping, case routing, the one-sanction lock, the automation-exclusion history guard, and the gated autonomous-action policy.
- All code, documentation, features, thresholds and prompts are written independently. Nothing is copied from the paper's prose, figures or tables, and nothing is derived from Melonly's code, weights or data (none of which is public).
- The paper is cited as prior art and inspiration.
- Name and branding: **Watchcat** only. No use of "Melonly" or "Watchdog" in branding.

### What self-hosting changes

**Unchanged**

- Copyright and trademark obligations apply regardless of the project being free and open source.
- A patent search (Google Patents, Espacenet) is still advisable before the first release. [?]
- All undisclosed components (section 8) still have to be designed from scratch.

**Easier**

- **Data protection responsibility moves to operators.** With no hosted service and no telemetry, each community running an instance is the controller of its own players' data. Watchcat should:
  - send no telemetry or usage data back to the maintainers;
  - ship privacy-friendly defaults and a configurable data-retention period;
  - document operators' responsibilities (children's data, automated decisions, appeals).
- **API terms are simpler.** Each operator supplies its own ER:LC server key and API pack.
  - The repository contains only a `.env.example`, never real keys.
  - The default deployment is one instance per server. Operators running one instance for many servers must register a PRC public application, because unauthenticated multi-server use is blocked (403, code 2005).
- **Cross-community player records largely disappear**, since each instance is isolated.

**Harder**

- **No pooled data.** Instances start empty and stay small, so learned models must be trained per instance, or not used. This matches the roadmap: rules first, per-instance learning later.
- **Pretrained weights.** Publishing a model trained on players' behaviour raises consent questions. The safer distribution is:
  - rules, feature pipeline, training tools and audit tools, with **no behavioural weights**;
  - avatar screening as zero-shot SigLIP 2, which needs no player data;
  - optionally, later, an explicitly opt-in donated dataset with documented consent.
- **Unsafe forks and misconfiguration.** Anyone can enable automatic sanctions with weak settings. Mitigations built into the defaults:
  - autonomous action **off by default**;
  - when enabled, limited to mass RDM and to kicks;
  - the gate refuses to activate until a minimum number of audited cases with sufficient precision exist on that instance;
  - prominent documentation of the risks.

### Required repository notices

- README disclaimer: *Watchcat is an independent open-source project. It is not affiliated with, endorsed by or derived from the code of Melonly Ltd, Police Roleplay Community (PRC) or Roblox Corporation.*
- The licence's warranty disclaimer covers maintainer liability.
- A NOTICE file listing third-party components and their licences.

### Licence choice (to decide before accepting outside contributions)

| Licence | Effect |
|---|---|
| **AGPL-3.0** | Anyone offering a modified Watchcat as a hosted service must publish their changes. Prevents closed SaaS forks. |
| **Apache-2.0** | Permissive, with an explicit patent grant. Allows closed forks. |

---

## Sources

- Primary reference: [Watchdog paper](https://melonly.com/research/watchdog.pdf)
- ER:LC API: [Docs](https://apidocs.erlc.gg/) · [v2 OpenAPI spec](https://api.erlc.gg/internal/docs/apispec.v2.json) · [Rate limits](https://apidocs.erlc.gg/rate-limits.md) · [Event webhooks](https://apidocs.erlc.gg/event-webhooks.md) · [Location data](https://apidocs.erlc.gg/how-do-i-interpret-the-xyz.md) · [Large applications](https://apidocs.erlc.gg/large-applications.md) · [AUP](https://apidocs.erlc.gg/policies/aup.md) · [Large-app AUP](https://apidocs.erlc.gg/policies/large-app-aup.md)
- Product: [Melonly Watchdog docs](https://docs.melonly.xyz/docs/Dashboard/watchdog)
- Roblox: [Thumbnails API](https://create.roblox.com/docs/cloud/reference/features/thumbnails)
- Models and libraries: [SigLIP 2 listing (Voxel51)](https://docs.voxel51.com/model_zoo/models/google_siglip2_base_patch16_224.html) · [OpenCLIP](https://github.com/mlfoundations/open_clip) · [CLIP](https://github.com/openai/CLIP)
- Methods: [XGBoost](https://arxiv.org/abs/1603.02754) · [LightGBM](https://papers.nips.cc/paper/6907-lightgbm-a-highly-efficient-gradient-boosting-decision-tree) · [CLIP paper](https://arxiv.org/abs/2103.00020) · [Menon et al. 2015](https://proceedings.mlr.press/v37/menon15.html) · [Guo et al. 2017](https://arxiv.org/abs/1706.04599) · [Niculescu-Mizil & Caruana 2005](https://dl.acm.org/doi/10.1145/1102351.1102430) · [Elkan & Noto 2008](https://dl.acm.org/doi/10.1145/1401890.1401920) · [Wilson 1927](https://doi.org/10.1080/01621459.1927.10502953)
- Legal: [ICO automated decision-making guidance](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/automated-decision-making-and-profiling/) · [17 U.S.C. §102](https://www.law.cornell.edu/uscode/text/17/102)
