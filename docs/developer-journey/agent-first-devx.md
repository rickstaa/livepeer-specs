# Agent-first DevX

*Status: stub — design principle established, layered architecture sketched. Layers 1-3 are the MVP scope for the H1 outcome; Layer 3 is an MVP dependency because Layer 2's `network_state` delegates into it.*

Every capability in the Developer Dashboard — its card, schema, playground, snippet, docs — is designed to be legible to AI coding agents as much as to humans. An agent using Claude, Cursor, or any MCP client should be able to discover a Livepeer capability, generate working code, and integrate it, without a human in the loop.

## Problem

The H1 2026 outcome says, in so many words, that a developer using Claude or Cursor or any MCP-compatible tool should be able to make a first Video AI call in under 5 minutes. That only works if the **entire surface** is designed for agent consumption from day one — docs, schemas, SDK, network data, auth flows.

Livepeer has the window to get this right on the first pass because the Dashboard is still being built. If we design each component with agents as a first-class reader, every part of the funnel benefits — *and* humans benefit too, because agent-legible design is almost always more consistent, better documented, and more rigorously typed than human-only design.

## Proposed shape — a three-layer architecture

Agents hit Livepeer from several angles — looking up a fact from docs, asking about live network state, interacting with the user's own signer connections. No single endpoint can cover all of them. Three layers, each for a different kind of interaction.

### Layer 1: Agent-legible docs (Mintlify — `llms.txt` + docs MCP)

Entry point for agents that have never heard of Livepeer, plus the look-up surface for agents that want to find a specific doc page mid-session. Mintlify (which hosts `docs.livepeer.org`) provides **both** surfaces automatically:

- **`docs.livepeer.org/llms.txt`** (plus `/llms-full.txt`) — static markdown pointers for crawl-time ingestion (model training, RAG pipelines, coarse bootstrapping).
- **`docs.livepeer.org/mcp`** — live MCP server with `search` (full-text search across docs) and `query filesystem` (shell-style navigation) tools. Works out-of-the-box with Claude, Cursor, Claude Code, and VS Code via standard MCP client config. Rate-limited at Mintlify defaults (5k requests/user/hour; 10k/site/hour per tool). See [Mintlify's MCP docs](https://www.mintlify.com/docs/ai/model-context-protocol).

Both are generated from the `docs.json` navigation structure — no infrastructure work on our side. The open work is **content quality**: making sure the canonical pages (overview, quickstart, capability catalog, API reference) are organised, named, and cross-linked so both surfaces return the right thing. The [documentation-restructure RFP](https://forum.livepeer.org/t/rfp-documentation-restructure/3071) is the vehicle for that.

### Layer 2: Dashboard MCP server

Active-work interface for **dynamic, stateful** operations — catalog, live network state, usage, snippet generation, PymtHouse connection. Docs search and navigation are covered by Mintlify's MCP (Layer 1), so this layer specifically carries the things static docs can't: anything that depends on the current state of the network, the user's keys, or the user's connected signers.

The natural home is the Developer Dashboard — that's where the aggregated developer experience already lives, per [developer-dashboard.md](./developer-dashboard.md) — but **the hosting location is not yet fixed.** If the MCP server is on the Dashboard and the Dashboard goes down, agents lose network-state queries at the same time. Hosting it separately (e.g. `mcp.livepeer.org` backed by its own infra), or running it as an HA pair independent of the Dashboard's own HA, are live alternatives. This is a redundancy question worth resolving explicitly; see *Open questions* below.

Initial tool surface (all read-only or side-effect-with-explicit-consent):

- `list_capabilities()` — browse the catalog.
- `describe_capability(id)` — OpenAPI schema for one capability. An agent that holds this can deterministically write code to call the capability; no separate snippet tool is needed.
- `network_state(capability_id?)` — live orchestrator count, p50 latency, price. *Delegates to the production-data MCP (Layer 3).*
- `check_usage()` — against the authenticated user's key.
- `connect_clearinghouse(issuer_url)` — *starts* the PymtHouse OIDC device flow and returns both a user-facing **approval URL** and a short-lived **polling handle**. The user opens the URL and approves consent at the PymtHouse directly; the agent polls the handle to detect completion. No part of the approval happens silently through the MCP.

Auth is the **Dashboard API key** the developer already has — the MCP client presents it in tool calls. This key authenticates the user to the Dashboard only; it does *not* unlock PymtHouse signing authority, because MCP calls here do not spend money (the only tool with side effects, `connect_clearinghouse`, kicks off OIDC at the PymtHouse and requires the user's direct consent there). **The MCP does not issue keys.** Key issuance stays in the web Dashboard, behind an OAuth + human-consent flow. Agents receive keys from the user's environment; they do not request them.

### Layer 3: Production-data MCP (delegated-to)

Raw network-state queries for agents doing data-style exploration rather than developer-flow work. Exposed by the production-data surface itself, not the Dashboard. **MVP-required** because Layer 2's `network_state` delegates here — it has to be reachable for that tool to work.

The Dashboard MCP (Layer 2) calls into this for live state. It is also directly callable for operator-style agents that want raw telemetry rather than developer-framed answers.

See [production-data.md](./production-data.md) for the data surface and the MCP tool list.

## Design principles

These govern every layer.

### 1. Schema is the contract

OpenAPI schemas from the [Pipeline SDK](./pipeline-sdk.md) flow through every layer — linked from `llms.txt`, exposed as MCP `describe_capability`, consumed by the client SDK. An agent that reads the schema should be able to write correct integration code. If a capability can't be described accurately in its schema, that's a spec bug, not a documentation gap.

### 2. Error messages self-correct

Every error returned from the SDK or MCP server should contain enough context that an LLM reading it can fix the code. Avoid opaque error codes with separate lookup tables. Include the actual parameter that was wrong, the valid range, and a suggestion where possible. Good error messages help humans too.

## Out of scope

- **Building Livepeer-specific agents.** We're making our surface agent-friendly; we're not shipping our own agent.
- **On-protocol attestation** that "the agent that made this call was authorised." Standard API-key auth with consent UX is enough for MVP.
- **Full MCP client conformance testing** across every MCP-compatible client. Target Claude and Cursor as primary; others come along because the protocol is standard.

## Open questions

- **Hosting URL and redundancy posture** — `mcp.livepeer.org`? `studio.livepeer.org/mcp`? `.well-known/mcp` on the Dashboard? This is not only a URL choice — it's a redundancy decision. If the MCP server is coupled to the Dashboard's lifecycle, they go down together. Running the MCP server on separate infrastructure (or as an HA pair independent of the Dashboard's own HA) avoids that coupling but costs operationally. Decide before anyone wires it up.
- **Token scoping** — do MCP clients present the same API key as direct SDK calls, or a scoped-down MCP-only credential? The latter is safer but adds an issuance UX.
- **Rate-limiting agent traffic** — agents burst differently than humans (a Cursor autocomplete session can fire many exploratory MCP calls per second). A separate quota bucket for MCP clients, standard rate-limit headers, or both? Defaults matter since most callers won't configure.
- **Decentralised-auth variant** — if we ever move away from Dashboard-issued API keys toward a PymtHouse-issued-token or wallet-based auth model, what does the MCP auth surface look like then? Specifically, how does an agent authenticate against the MCP server when there is no Dashboard root credential?
- **SDK-local MCP mode** — do we eventually ship a `livepeer-sdk mcp-server` that exposes the installed client SDK as MCP tools to an agent in the developer's IDE, or is the SDK's own type-rich ergonomics enough on its own? Post-MVP either way.
- **`llms.txt` maintenance model** — who owns the file, how does it stay in sync with the Mintlify docs, what lives in `llms-full.txt` vs the pointer file?
- **Landscape signal** — as of early 2026, Replicate, Fal, and Chutes don't appear to publish hosted MCP servers. First-mover advantage for a demand-facing protocol, or a signal that it's premature? Worth a second pass on [landscape.md](./landscape.md) once this spec firms up and once those platforms' current state is re-verified.
