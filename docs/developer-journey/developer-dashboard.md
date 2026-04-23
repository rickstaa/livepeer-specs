# Developer Dashboard

An opinionated, community-owned surface that takes a developer from **discovering** what the Livepeer network can do, to **experimenting** with a capability in a playground, to **integrating** it into their app — all without direct hand-holding from the Livepeer community.

The Developer Dashboard is not a new piece of infrastructure. It is the one surface that ties together the components Livepeer engineering has shipped over the last months — the Livepeer Cloud data work, the remote signer, the client SDK, upcoming runner SDK — into a single funnel a developer can walk end-to-end.

> See also: [Prior art — how other ecosystems do this](./prior-art.md). Replicate, Modal, Chutes, Prime Intellect, Fal.ai.

## Agent-first by default

Every capability in the Dashboard — its card, schema, playground, snippet, and docs — is designed to be legible to **AI coding agents** as much as to humans. An agent using Claude, Cursor, or any MCP client should be able to discover a capability, generate working code, and integrate it, without a human in the loop. The schema is the contract.

This is a design principle, not a feature. It affects every layer below — the docs, the Pipeline SDK schema, the client SDK's error messages, and a hosted MCP server on the Dashboard. See [agent-first-devx.md](./agent-first-devx.md) for the full four-layer architecture.

## Why this exists

Getting from *"I heard of Livepeer"* to *"my code just made a Video AI call"* takes **15-30 minutes today** and asks a developer to navigate orchestrators, gateways, tickets, and fragmented docs. Each of the components needed to collapse that path now exists — but they're scattered, and the developer is expected to assemble them. The Developer Dashboard is the one place where they come together.

The goal is not completeness. It is **an opinionated shortest path** from discovery to a working call. One catalog. One key. One playground. One SDK snippet. Copy, paste, ship.

## What the Developer Dashboard is

A **capabilities-first** developer portal — in the lineage of Replicate, Chutes, Fal — specialised for real-time AI video. A developer lands, browses the capabilities the network serves, tries one in an auto-generated playground, and walks away with an API key and a working snippet.

Concretely, the Developer Dashboard offers:

- A **public catalog** of capabilities the network can serve today (one card per capability).
- A **playground** per capability, auto-generated from the capability's OpenAPI schema.
- **API-key issuance**, usage history, and lightweight billing.
- **Network-state surfacing** woven into capability cards and the home page — real orchestrator counts, real latencies, real throughput.

The live skeleton of this is in flight on [PR #57 of `livepeer/website`](https://github.com/livepeer/website/pull/57) — explore, model detail + playground, home page, header, auth context, and visual prototypes of keys, usage, settings, and provider pages.

## How the Developer Dashboard uses each component

The Dashboard is the umbrella. Each component below is substrate it reads from, routes to, or surfaces.

### Production-data surface — *network visibility*

For the stats a developer cares about (*"is this capable enough for my use case? how is it already being used? what latencies and costs?"*) the Dashboard reads from the Cloud team's real-time metrics and the upgraded subgraph (gateway tracking, treasury tracking, delegator-income tracking). The same data feeds the naap operator app at [operator.livepeer.org](https://operator.livepeer.org); the Dashboard reads the same endpoint. Over time, **SDK-reported metrics** add a second, caller-side feed that cross-validates operator self-reporting and keeps the surface useful if the operator-side feed blips.

Today this endpoint is centralised (naap + Cloud data infrastructure). One possible longer-term improvement would be a small aggregator container anyone can run, gossiping with peers — so the data surface doesn't disappear if any one operator does. Not committed; an illustrative direction. See [production-data.md](./production-data.md).

### Client SDK — *playground + snippets*

Every capability page exposes a playground that calls the network via the client SDK. Crucially, **the call the developer runs in the playground is the call they copy into their app** — the playground is the SDK, rendered as a form. No separate playground-only runtime, no drift between playground and production. See [client-sdk.md](./client-sdk.md).

### Remote-signer clearinghouse — *payment without a gateway*

A developer does not need to run a gateway to use the Dashboard. A **payment clearinghouse** (a remote signer) holds the wallet, signs tickets, and pays orchestrators on the developer's behalf — the client SDK partners with it.

The developer sees **one API key**. At session start the SDK exchanges it for a short-lived **token bundle** — one entry per connected clearinghouse, each with currencies, priority, and refresh credentials. All per-call traffic then flows SDK ↔ clearinghouse directly, with transparent failover. The Dashboard is in the path only at session start and token refresh, so apps keep working even if it goes offline mid-session.

Day one: a single community-hosted clearinghouse, auto-connected at login. Users can connect more — Foundation-, solution-provider-, or user-run, each with their own currency mix — via **OIDC + device flow**. Each is a full peer, not a subordinate; connecting one adds an entry to the bundle.

For users who want zero intermediaries, a **direct ETH** path bypasses clearinghouses entirely and pays orchestrators on-chain.

See [remote-signer.md](./remote-signer.md) for the protocol, architecture trade-offs, and failure-mode checklist.

### Pipeline SDK — *adding capabilities to the catalog*

New capabilities land in the catalog through the **Pipeline SDK**: a builder writes a Python class, packages a container compatible with the BYOC communication contract. The container carries **metadata** (model info, pricing hints) and exposes an **OpenAPI-compatible endpoint** — which the Dashboard reads to auto-generate the capability's card, playground form, and SDK snippet.

Three job shapes are supported, with transport auto-selected from the Python class shape: **batch** (one-shot image/audio over HTTP), **SSE** (LLM token streaming), and **real-time** (video/audio over trickle transport).

The container is the registry. No separate model registry is required for MVP. A richer model/metadata registry layer, as is [already done in this naap PR](https://github.com/livepeer/naap/pull/269), is a possible later addition — not an MVP requirement — added only if the opinionated-container pattern proves insufficient. See [pipeline-sdk.md](./pipeline-sdk.md).

### Docs — *entry point for humans and agents*

For humans, the docs at [docs.livepeer.org](https://docs.livepeer.org) are the entry point — they direct readers toward the Dashboard. Significant work has already landed via an earlier [documentation-restructure RFP](https://forum.livepeer.org/t/rfp-documentation-restructure/3071); upcoming work simplifies the stakeholder journeys further.

For agents, Mintlify already provides two automatic surfaces on top of the same content: `docs.livepeer.org/llms.txt` for crawl-time ingestion, and `docs.livepeer.org/mcp` for live search and navigation in Claude, Cursor, or any MCP client. Both are free with Mintlify; the work is content quality. The Dashboard's own MCP server handles the dynamic, stateful surface (catalog, network state, usage, connecting signers) — not docs search, which Mintlify already covers. See [agent-first-devx.md](./agent-first-devx.md).

## What the Developer Dashboard is not

### Not a commercial SaaS product

The Foundation is not a commercial company. The Dashboard exists to **amplify discovery** of what Livepeer can do — not to be a managed service with SLAs, account reps, or enterprise contracts.

SLAs are a commercial-provider job. Solution providers on [livepeer.org/ecosystem](https://livepeer.org/ecosystem) — Frameworks, Blue Claw, Streamplace, Daydream — can offer the SLAs, support, and enterprise contracts the Foundation won't. The Dashboard points developers to the ecosystem page when their needs outgrow the community surface.

Enterprise adoption still runs through the adopting company or its chosen provider — they run their own payment signer (protocol work — e.g. accepting ERC-20 stablecoin tickets — can simplify this meaningfully over time).

### Not a single point of failure

The Dashboard is allowed to **bootstrap with centralised pieces** to ship fast.

Doing this fully decentralised from day one is possible — but materially harder. The plumbing to do it well (gossip, aggregation, permissionless registration, redundancy, reputation) is real engineering work, and rushing it tends to produce brittle systems that erode trust rather than build it. So the operating principle is **minimum viable centralisation**: start from what genuinely has to be centralised to ship, keep the full decentralised picture visible in every design, and replace centralised components with decentralised or redundant alternatives **where it makes sense** — piece by piece, as the plumbing matures. Not every component needs full decentralisation; some are fine as coordination points provided they have visible redundancy paths.

Concrete current bets:

- **Data endpoint** is naap + Cloud infrastructure today. Path: a small aggregator container anyone can run, maybe gossip like style synchronisation between aggregators (exact protocol undecided).
- **Payment clearinghouse** is one community-hosted signer today. Path: John's work → multiple registered signers + Dashboard token-routing → full permissionless signer registration.
- **Catalog sorting** uses observable signals — reliability, request volume, latency — rather than a Foundation-curated tier. Any capability reachable through a conformant Pipeline SDK container is eligible; the signals do the ranking.

The design test for any new Dashboard feature: **if our centralised bootstrap goes offline, what degrades? If the answer is "the whole network experience," rethink.**

### Not a solutions catalog

The Dashboard lists **capabilities** (individual models, pipelines, network primitives). **Solutions** (pre-built products like Daydream, Streamplace, Embody) live on [livepeer.org/ecosystem](https://livepeer.org/ecosystem).

The split matters. If the Dashboard wrapped third-party solution APIs via OAuth or similar, it would:

1. Duplicate the ecosystem page's job.
2. Compete with the Foundation's own design-partner solutions.
3. Blur the contract a capability card offers — a network-served capability has observable reliability and request-volume signals; a wrapped solution API does not.

Payment providers are the one permitted cross-over, and only through the remote-signer model: a solution provider can pay for a user's network calls, but **the capability being called is still a network capability, not a wrapped solution API.**

### Not Modal.com-style "run any code"

The Pipeline SDK's current shape is *package a container that orchestrators opt into running*. It is not one-command, fully-automated, run-any-code deployment. Closing that gap is a later track, not this one.

### Not the Explorer

The [Explorer](https://explorer.livepeer.org) is the sibling surface — the permissionless **participation portal** for operators (Extend) and delegators (Invest). That's where the network's supply side and token-holders do their ongoing work: staking, monitoring orchestration, participating in governance, observing network state. Same design language, separate information architecture, separate roadmap, separate audiences.

The test for overlap is *audience and job*. **If it's a build task** (making an application using the network), it belongs on the Dashboard. **If it's an extend-or-invest task** (operating the network, delegating LPT, voting on LIPs), it belongs on the Explorer. Neither absorbs the other.

### Not a provider-management dashboard

The Dashboard is for developers **discovering and calling** capabilities — not for providers managing their own pipelines through a UI. Providers publish capabilities via the [Pipeline SDK](./pipeline-sdk.md): write a class, push a container. Operational management of adapters and inference servers runs through the existing [`livepeer-inference-adapter`](https://github.com/livepeer/naap/tree/main/containers/livepeer-inference-adapter) container and the naap [operator app](https://operator.livepeer.org).

A provider-facing UI inside the Dashboard is out of scope for this track. A future track could add it if the Pipeline SDK plus operator-side tooling proves insufficient.

## Open questions

There are more but here are some open questions:

- The exact token-routing interface between the SDK and the Dashboard API once multiple clearinghouses exist — especially failure-mode semantics when a registered signer goes offline mid-session.
- How the Pipeline SDK's OpenAPI metadata reaches the Dashboard — push-on-publish, pull-from-orchestrator, or a dedicated registry? The opinionated-container model pushes this toward pull, but the tradeoff isn't decided.
- Which observable signals are load-bearing for catalog ranking — reliability, request volume, latency, user-ratings? What's the weighting, and how do we avoid new capabilities being invisible by definition (cold-start problem)?
- Exact shape of the aggregator-container/gossip protocol for the data surface. Out of scope for MVP; needs a deliberate design pass before building.
- Whether the Dashboard ever hosts a first-party model/metadata registry layer, or keeps the container-is-registry model forever.
