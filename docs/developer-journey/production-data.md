# Production-data surface

*Status: stub — most of the data already exists (subgraph + Cloud aggregator + naap-api + remote signer); the structure of the developer-facing surface is not yet decided.*

The data a developer needs to judge whether Livepeer can serve their use case, and to see how it's already being used — without needing to know who an orchestrator is. Consumed by the [Developer Dashboard](./developer-dashboard.md), and potentially by agents via MCP.

## Problem

A developer evaluating Livepeer needs concrete answers to questions like:

- *Is the network serving this capability right now, and at what latency?*
- *How many orchestrators are serving it?*
- *What does throughput look like at the times I care about?*
- *What have callers paid per call historically?*
- *How reliable has it been?*

Those answers already exist in one form or another — spread across the subgraph, the Cloud team's metrics stack, the naap operator API, and the remote signer. None of them are framed for a developer evaluating fit.

**This spec is about making sure that data reaches the Dashboard in the shape the Dashboard needs. The exact source-of-truth, aggregation point, and caching architecture is an open decision, not a conclusion.**

## What the Dashboard needs

For per-capability pages in the Dashboard:

| Question | Data |
|---|---|
| Is this capability being served? | List of orchestrators currently serving this model/pipeline |
| How fast? | p50 / p95 / p99 latency over a rolling window |
| How reliably? | Success rate + recent failure summary |
| At what cost? | Current price range + historical per-call payments |
| At what volume? | Requests per minute / day over the last N days |
| How does it rank? | Leaderboard position among orchestrators serving the same capability |

For the Dashboard home / catalog:

| Question | Data |
|---|---|
| What is the network running overall? | Total orchestrators, total GPUs, models served, aggregate throughput |
| Where is demand going? | Top pipelines / models by payment volume |

## Candidate data sources

Three layers already exist; any of them — or a combination — can feed the surface. The choice is open.

### naap-api (operator layer, cached + realtime)

The [naap-api](https://naap-api.livepeer.cloud/docs) already exposes most of what the Dashboard would ask for, backed by the Cloud aggregator with a caching + realtime layer on top. Relevant endpoints:

- **Network inventory:** `/v1/net/summary`, `/v1/net/orchestrators`, `/v1/net/gpu`, `/v1/net/models`, `/v1/net/gateways` (+ per-address profiles).
- **Streams:** `/v1/streams/active`, `/v1/streams/summary`, `/v1/streams/history`, `/v1/streams/samples`.
- **Performance:** `/v1/perf/fps` (+ history), `/v1/perf/latency`, `/v1/perf/webrtc`.
- **Payments:** `/v1/payments/summary`, `/v1/payments/history`, `/v1/payments/by-pipeline`, `/v1/payments/by-orch`.
- **Reliability:** `/v1/reliability/summary`, `/v1/reliability/orchs`, `/v1/reliability/history`, `/v1/failures`.
- **Leaderboard:** `/v1/leaderboard` (+ per-orch profile).

If the Dashboard reads from naap-api, MVP is thin: the endpoints exist and the shapes are usable. Today's production consumer of the same API is the [operator app](https://operator.livepeer.org).

### Cloud data aggregator (direct)

See [Metrics and SLA foundations for NAAP](https://forum.livepeer.org/t/metrics-and-sla-foundations-for-naap/3189/13). The aggregator is the upstream source for naap's realtime metrics. Reading it directly means fewer hops and no caching layer, at the cost of re-doing whatever shaping naap does today.

### Remote signer / PymtHouse

Relevant for **per-user** usage and billing — "what has *this* developer called, what did they pay, what is their quota left?" Not a substitute for network-wide state; a complement to it. See [remote-signer.md](./remote-signer.md).

### SDK-reported metrics (demand-side)

A second, independent feed emitted by the client SDK. Important for cross-validation and resilience — detailed below.

## What already shipped

- **Subgraph data upgrades** — gateway tracking, treasury tracking, delegator-income tracking (coming). Richer historical data per orchestrator and per gateway.
- **Cloud team real-time metrics** — see [Metrics and SLA foundations for NAAP](https://forum.livepeer.org/t/metrics-and-sla-foundations-for-naap/3189/13).
- **naap-api** at [naap-api.livepeer.cloud](https://naap-api.livepeer.cloud/docs) — the operator-facing API with caching + realtime; already powers [operator.livepeer.org](https://operator.livepeer.org).

## Not a single point of failure

Whichever source is picked, today it's centralised (naap + Cloud). Acceptable bootstrap, not the final state. Possible decentralisation directions — none committed; each needs a dedicated design pass:

- An **aggregator container** anyone can run, pulling from the subgraph + metrics feed and gossiping with peers (exact protocol undecided).
- **SDK-reported metrics** as an independent second feed (see extension below) — already useful even without the aggregator-container path.
- Reading **directly from the subgraph** for anything on-chain-derivable — the subgraph is decentralised infrastructure today.

Design principle: *minimum viable centralisation*. See [developer-dashboard.md](./developer-dashboard.md#not-a-single-point-of-failure).

## Extension: SDK-reported metrics (demand-side feed)

The [client SDK](./client-sdk.md) can emit metrics back as a **second, independent feed** complementing the operator-side data above. What shipped so far is operator-reported (orchestrators self-reporting through the metrics stack). What this adds is *caller-reported* — the SDK observing actual request outcomes from the developer's side.

Why this matters for the surface:

- **Cross-validation.** Two independent feeds let the aggregator catch operator misreporting — an orchestrator claiming 99% success while SDK-side telemetry shows 80% is a red flag.
- **Resilience.** If the operator-side Cloud/naap feed blips, SDK-reported metrics partially back-fill. The surface degrades gracefully instead of going dark.
- **Demand-side truth.** "P50 latency at my calling location" is a different question from "P50 latency reported by orchestrator X." Developers care about the former.

Scope of what the SDK reports is the open design question — at minimum: success/failure, end-to-end latency, orchestrator ID (if known), capability ID, approximate client region. Privacy-respecting by default.

## Extension: MCP endpoint for agents

The same data surface should expose a **queryable MCP endpoint** so AI coding agents can ask live network-state questions the way a developer would, but programmatically — from Cursor, Claude, or any MCP client.

Initial tools:

- `orchestrator_count(capability_id?)` — how many orchestrators are serving a given capability right now.
- `capability_latency(capability_id, percentile?)` — p50 / p95 / p99 over the last N minutes.
- `capability_price_range(capability_id)` — current observed price bounds.
- `historical_usage(capability_id, window)` — rolling throughput over N days.

Relationship to the [Dashboard MCP](./agent-first-devx.md#layer-2-dashboard-mcp-server): the Dashboard MCP delegates to this one for live-state queries, so developer-facing tools like `network_state()` are thin wrappers that package the raw answers into developer-framed responses. This MCP is also directly callable — for operator-style agents who want raw telemetry rather than a developer-framed answer.

## Out of scope

- Per-orchestrator SLA contracts or guarantees.
- Alerting / paging for developers (that's a monitoring product; this is a data surface).
- Historical replay / time-travel debugging.
- **Picking the primary data source** — this spec lists candidates; the source decision is deferred to implementation.

## Open questions

- **Primary data source** — naap-api, direct Cloud aggregator, subgraph-first, or a hybrid? Each has different latency/caching/ownership trade-offs.
- **Shape of the Dashboard-facing response** — per-capability aggregates, per-orchestrator breakouts, or both; pre-joined server-side or Dashboard composes from multiple calls.
- **Which metrics are load-bearing** for a developer's fit decision, vs. operator-facing ones that should stay in the [operator app](https://operator.livepeer.org).
- **Rate-limiting and abuse controls** — if the data endpoint is exposed publicly, what limits; if it's proxied through the Dashboard only, does the Dashboard inherit naap's limits?
- **Decentralisation path** — small aggregator container + gossip? Direct subgraph reads? SDK-reported back-fill as primary? Deliberate design pass needed.
- **Per-user vs. network-wide split** — when does the Dashboard need the remote-signer's billing/usage data vs. the naap network view?
