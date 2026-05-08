# Production-data surface

*Status: draft, open for discussion. The current data landscape is in good shape thanks to the work shipped by Livepeer Cloud; the open question is whether the underlying architecture should evolve from push to pull as Livepeer matures toward decentralisation.*

The data anyone in the Livepeer ecosystem needs to judge whether the network can serve a use case and to see how it's already being used. The [Developer Dashboard](./developer-dashboard.md) is one consumer; the [client SDK](./client-sdk.md), agents via MCP, community dashboards, and solution-provider apps are equal peers — none more privileged than another.

## Why we need this data

A developer evaluating Livepeer needs concrete answers in seconds: is this capability being served right now, how fast, how reliably, at what cost. Without that, they can't pick credibly, can't trust that today's playground demo will work tomorrow, and can't ship.

The data is load-bearing in **two** places along the 5-min journey, not one. The dashboard prototype on [`livepeer/website#claude/dashboard-updates`]( http://website-git-claude-dashboard-updates-livepeer-foundation.vercel.app/?_vercel_share=0oNBeykUezA6lXR5jsdSsAQnsLdHdQQ0) makes the shape concrete:

- **The Dashboard.** Three surfaces drive the data needs:
  - **Catalog (`/dashboard/explore`)** — capability cards filterable per category, availability (warm / cold), and price; ranked by usage. Needs **historic + real-time per-capability** signals: latency percentiles, success rate, price, popularity, current warm/cold availability.
  - **Per-capability detail (`/dashboard/models/[id]`)** — playground, stats, recent runs. For the playground to actually work, a developer signed in via the [remote signer](./remote-signer.md) needs **real-time capability availability** so the test job is dispatched only to orchestrators currently serving that model.
  - **Network monitor (`/dashboard/network`)** — overview, utilisation, payments, GPU breakdown, with a visible *"live · updated Xs ago"* recency pill. Needs **real-time network-wide** aggregates: total orchestrators, GPUs, throughput, payment volume.

  > **Disclaimer.** The dashboard prototype is a working draft, not a locked design. The specific metrics listed above (latency percentiles, success rate, popularity, warm/cold availability, payment volume, etc.) are **exploratory** — I'm still figuring out which signals actually drive developer decisions in this early phase and which are noise. Expect the list to evolve as the dashboard goes in front of real users. Read it as *the kind of data the dashboard surfaces will need*, not as a fixed contract.
- **The SDKs.** The [client SDK](./client-sdk.md) consumes the same surface at runtime — discovering which orchestrators are alive and serving the requested capability, then selecting one before opening a session. The data is part of how a session gets established, not decoration on top of it. Any agent-driven flow that picks an orch programmatically needs the same answers.

So the surface has to deliver three things in parallel: **real-time per-capability availability**, **historic per-capability performance**, and **real-time network-wide aggregates** — consumed by both human surfaces (Dashboard cards, monitor pages) and machine surfaces (SDK discovery, MCP tools).

> **Scope caveat.** This spec frames the data needs from the **developer side** because that's the focus of the 5-min Developer Journey track. The same underlying data is load-bearing for other stakeholders too — orchestrators tuning capacity and pricing (already served by [operator.livepeer.org](https://operator.livepeer.org)), remote signers picking which orchs to route payments to, solution providers building their own consumer surfaces. The pull architecture proposed below cuts both ways: each stakeholder can run (or pay for) the consumer shape that fits them, all reading from the same primitive. The dev-side framing here is a focus choice, not an assumption that other consumers don't matter.

## What's current

The data landscape today is genuinely better than a year ago. Two milestones worth recognising:

- **Community proposal: [Decentralized Metrics and SLA Foundations for Livepeer](https://forum.livepeer.org/t/decentralized-metrics-and-sla-foundations-for-livepeer/3165).** Laid out the problem space — what reliable network-wide metrics should look like for a decentralised system — and anchored the broader community conversation.
- **Shipped by Livepeer Cloud, incorporating community feedback: [Metrics and SLA foundations for NAAP](https://forum.livepeer.org/t/metrics-and-sla-foundations-for-naap/3189).** This is the working production data stack today: orchestrator events flow into Kafka, ClickHouse ingests via the Kafka Engine, dbt and a resolver service produce canonical and serving views, the [naap-api](https://naap-api.livepeer.cloud/docs) exposes the result, and [operator.livepeer.org](https://operator.livepeer.org) consumes it. Constituent repos: `livepeer-naap-analytics`, `NaaP`, `livepeer-leaderboard-serverless`, `livepeer-ai-job-tester`.

So the *availability* of this data is largely solved. The remaining open question is **architectural** — whether the current shape holds up under **production usage at scale on a decentralised network**, not just "more decentralisation" as an abstract goal. Decentralisation matters because Livepeer's value proposition depends on it; the question is what data architecture supports real production load on top.

## A direction worth discussing — pull rather than push

The shipped pipeline is **push-based**: orchestrators emit events into a central Kafka topic, ClickHouse ingests, naap-api serves. It works, it ships, and Cloud has the trust to operate it for now.

But push fits awkwardly with where Livepeer is going as a decentralised network. Someone runs the destination; adding a second independent aggregator means reconfiguring every orchestrator; if the destination goes down, orchs either fail or buffer; censorship at the collector is a single filter away. The surface depends on a single party — the Foundation — keeping it alive, and the decentralisation story stays unfinished.

**As pointed out by J0sh**, a **pull-based** approach is likely more resilient. Orchestrators expose what they're hosting; anyone — naap-api, community dashboards, custom agents, solution providers — pulls from whichever subset of orchs they care about.

A few extensions of that idea worth flagging (mine, not J0sh's, and open to push-back): discovery is already solved by the on-chain ServiceRegistry that advertises orch service URIs, so adding a new aggregator becomes *"run an aggregator,"* not *"coordinate with every orch operator."* If one collector dies, others continue; censoring would mean blocking each consumer individually rather than dropping at one chokepoint.

**As pointed out by J0sh**, and as I currently understand it: workers register with their orch and **periodically heartbeat** their capability (model, pipeline, hardware, etc.); if heartbeats stop, the orch stops advertising. Aggregators, signers, and gateways then query orchs to aggregate discovery from there. Wire-level details are his to define; my current notes and the open questions raised when sketching the flow sit in [`orchestrator-observations.md`](./orchestrator-observations.md), which itself defers to his forthcoming canonical spec.

This is a direction, not a decision — open for community discussion.

## SDK-reported metrics — a complementary second feed

Independent of push vs. pull, having the [client SDK](./client-sdk.md) emit observations from the **caller** side (success/failure, end-to-end latency, target orch) gives a second independent feed alongside operator self-reporting. That cross-checks operator claims (an orch reporting 99% success while SDK telemetry shows 80% is a red flag), provides demand-side truth (*"p50 latency from where my users actually call"*), and back-fills if the operator-side feed blips. Privacy-respecting by default; scope is open.

## What this surface should answer

Examples of the kinds of questions, not an exhaustive list — the actual contract will be shaped by what the Dashboard and SDKs need in practice.

For per-capability views:

| Question | Data |
| --- | --- |
| Is this capability being served? | List of orchestrators currently serving this model/pipeline |
| How fast? | p50 / p95 / p99 latency over a rolling window |
| How reliably? | Success rate + recent failure summary |
| At what cost? | Current price range + historical per-call payments |
| At what volume? | Requests per minute / day over the last N days |
| How does it rank? | Leaderboard position among orchestrators serving the same capability |

For network-wide views:

| Question | Data |
| --- | --- |
| What is the network running overall? | Total orchestrators, total GPUs, models served, aggregate throughput |
| Where is demand going? | Top pipelines / models by payment volume |

These map equally to Dashboard cards, SDK runtime decisions, and MCP tool calls.

## MCP endpoint for agents

The same data surface should expose a **queryable MCP endpoint** so AI coding agents can ask live network-state questions programmatically — from Cursor, Claude, or any MCP client. Initial candidate tools:

- `orchestrator_count(capability_id?)` — how many orchestrators are currently serving a given capability.
- `capability_latency(capability_id, percentile?)` — rolling p50 / p95 / p99.
- `capability_price_range(capability_id)` — current observed price bounds.
- `historical_usage(capability_id, window)` — rolling throughput over N days.

The [Dashboard MCP](./agent-first-devx.md#layer-2-dashboard-mcp-server) delegates here for live-state queries; this endpoint is also directly callable for operator-style agents.

## Out of scope

- Per-orchestrator SLAs or guarantees.
- Alerting / paging for developers.
- Wire-level details of any pull mechanism — owned by J0sh's canonical spec.

## Open for discussion

- **Push or pull** as the long-term architecture, and the migration path from the current Cloud pipeline if pull wins.
- **Trust model** — operator self-reporting is suspect either way; signatures, on-chain checkpoints, and SDK cross-validation are candidate mechanisms. Probably its own spec.
- **Schema and versioning** of whatever observation contract emerges.
- **What "decentralised enough" means operationally** — at minimum two independent aggregators run by different parties? Three? Geographic spread?
