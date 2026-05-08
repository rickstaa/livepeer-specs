# Remote-signer clearinghouse (PymtHouse)

*Status: stub — first payment-abstraction work shipped, OIDC/device-flow integration in flight, multi-signer redundancy still to design.*

A third-party actor — a **payment clearinghouse**, aka **PymtHouse** — that signs and pays for a developer's network calls on their behalf, so the developer does **not** need to run a local gateway.

## Problem

To use the Livepeer network today, a developer has to run a local gateway that holds ETH, manages probabilistic-payment tickets, and signs every call. For a developer who just wants to make their first API call, that's a two-hour detour — and most will bounce before getting there.

## What's shipped

**`go-livepeer` remote signer actor.** Landed at [`server/remote_signer.go`](https://github.com/livepeer/go-livepeer/blob/main/server/remote_signer.go). Exposes HTTP handlers (e.g. `SignOrchestratorInfo`) that create a `Broadcaster` and return signatures + addresses as hex/JSON; wired into `ai_mediaserver.go`, `ai_process.go`, and `live_payment.go`; configurable via flags in `cmd/livepeer/starter/flags.go`.

**Client-side SDK counterpart.** The [gateway SDK](./client-sdk.md) at `livepeer_gateway/remote_signer.py` supports an **offchain mode** (no signer) and an **on-chain mode** (`--signer <host:port>`). End-to-end, a developer can already use the network via the SDK without running a local gateway — as long as a signer is reachable.

**PymtHouse implementation in flight.** John (`eliteprox`) is building a concrete clearinghouse across two linked artefacts:

- **[`pymthouse`](https://deepwiki.com/eliteprox/pymthouse)** — the clearinghouse itself. Next.js + PostgreSQL (Drizzle ORM). Acts as an OIDC provider (`oidc-provider`), a multi-tenant billing platform (`developer_app` = tenant), and a secure proxy fronting a `go-livepeer` signer sidecar. Admin auth via `next-auth` (Google / GitHub / Bearer).
- **[livepeer/website PR #50](https://github.com/livepeer/website/pull/50)** — integrates the above into the Developer Dashboard. Uses **OAuth 2.0 Device Authorization Grant (RFC 8628)** for headless SDK clients. A public OIDC client (`app_*`) drives browser login; a confidential **M2M client** (`m2m_*`) lets the Dashboard server-side `upsertAppUser`, `mintUserAccessToken` (scope `sign:job`), and `completeDeviceApproval`. User JWTs authorise the bearer to *request* signatures; the PymtHouse holds the wallet.

## How this enables the Developer Studio

The Developer Dashboard's payment story (see [developer-dashboard.md § Remote-signer clearinghouse](./developer-dashboard.md#remote-signer-clearinghouse--payment-without-a-gateway)) requires:

- **One API key** the developer sees; **no local gateway** to run.
- At session start, the SDK exchanges the key for a **short-lived token bundle** — one entry per connected clearinghouse, each with currencies, priority, and refresh credentials.
- All per-call traffic flows **SDK ↔ clearinghouse** directly; the Dashboard is in the path only at session start and token refresh.
- Day one: a **single community-hosted clearinghouse**, auto-connected on login.
- Users can connect additional clearinghouses — Foundation-, solution-provider-, or user-run — via **OIDC + device flow**; each is a full peer, with **transparent SDK-side failover** across them.
- A **direct-ETH escape hatch** for users who want zero intermediaries.

The shipped primitives map onto this cleanly:

| Dashboard requirement                   | Shipped / in-flight substrate                                   |
| --------------------------------------- | --------------------------------------------------------------- |
| No local gateway                        | `go-livepeer` remote signer actor + SDK `--signer` mode         |
| One API key → token bundle              | `pymthouse` OIDC provider + `mintUserAccessToken` (`sign:job`)  |
| Headless SDK auth                       | OAuth 2.0 Device Authorization Grant in PR #50                  |
| Community-hosted default clearinghouse  | `pymthouse` multi-tenant instance (Foundation-hosted)           |
| Connecting additional clearinghouses    | `pymthouse` as OIDC issuer; `app_*` public client per Dashboard |
| Per-tenant billing / usage              | `developer_app` tenant model in `pymthouse`                     |
| Direct-ETH escape hatch                 | SIWE + `PaymentTab.tsx` prototype in the Dashboard              |

## Full implementation is not yet settled

The primitives are in place, but the **PymtHouse Protocol** — the published spec that makes *any* conformant clearinghouse connectable, not just the Foundation's instance — is not yet drawn. Several integration choices between the Dashboard and the PymtHouse are still open: the M2M vs front-channel-OIDC trade-off that determines how permissionless "bring your own PymtHouse" actually is, the custody-disclosure format, the admin-security posture, the multi-tenancy / single-tenant split, multi-signer redundancy, and the token-storage split between Dashboard and SDK. The design sketch below is where we currently think each of these lands; the questions at the end are where they still need to be pinned down.

## Design sketch — keep apps working under failure

Each move preserves a specific failure mode.

### 1. OIDC + device flow *as a published spec*, not just an implementation

Publish a **PymtHouse Protocol** document in this repo or alongside `pymthouse`. It names:

- OIDC issuer metadata requirements.
- Device-flow endpoints and timing semantics.
- The `sign:job` scope (and any other scopes — e.g. `read:usage`, `admin:tenant`).
- A custody-disclosure format.
- The conformance tests (anyone implementing the spec runs these and publishes the result).

Anyone who implements the spec and publishes a passing conformance result is a valid PymtHouse. Foundation instance = one implementation.

### 2. Dashboard as routing-info provider, not per-call runtime

The Dashboard's job ends at "here are the PymtHouses you've connected, and here are the tokens for each." Beyond that, the SDK talks directly to PymtHouses.

- On connect, the Dashboard gives the SDK a bundle: `{ issuers: [{issuer_url, access_token, refresh_token?, priority, scopes}, ...] }`.
- The SDK caches the bundle client-side (in the developer's app).
- The Dashboard is consulted *only* when the user changes connected providers.

**Result: if the Dashboard goes down, apps keep working** on cached tokens.

### 3. SDK does client-side failover

Per-call, the SDK picks a PymtHouse by a policy:

- **Priority-ordered** (user-configurable; price-weighted default).
- **Health-aware** (rolling success-rate / latency window per PymtHouse; de-prioritise degraded).
- **Transparent retry** — on transport error or 5xx, retry the next PymtHouse before surfacing to the caller.

**Result: if any single PymtHouse goes down, apps keep working** as long as the user has connected ≥ 1 other.

### 4. Free-tier = one instance of the spec, with HA

Foundation-hosted free-tier PymtHouse:

- Publishes the same OIDC issuer metadata as any third-party.
- Auto-connected on first Dashboard login (every new user has ≥ 1 working signer immediately).
- Runs as ≥ 2 instances behind a load balancer.
- Publicly rate-limited — discovery/experimentation path, not a production backend.

### 5. Direct-ETH escape hatch stays

For users who want zero intermediaries: SIWE-style wallet connect → pay orchestrators directly on-chain → no PymtHouses in the loop. Already prototyped in the Dashboard's `PaymentTab.tsx`. Keep it.

### Token storage at the Dashboard (hybrid)

Of the three realistic options:

- *Server-side only* — Dashboard is a credential custodian; compromise is high-impact.
- *Client-side only* — Dashboard is not a custodian, but UX is harder (no cross-device sync, manual re-auth per browser).
- *Hybrid — refresh tokens at Dashboard, short-lived access tokens cached client-side by SDK* — Dashboard compromise gives attacker refresh-token access to PymtHouses, but PymtHouses can revoke per-user. Access tokens in flight stay valid briefly; signing authority does not leak.

**Hybrid is the right default.** Maps to *minimum viable centralisation* in [developer-dashboard.md](./developer-dashboard.md#not-a-single-point-of-failure) — Dashboard holds enough to be useful, not enough to be catastrophic.

## Failure-mode checklist

Design decisions must preserve these rows. Any regression needs a mitigation before shipping.

| Failure                              | What should keep working                                                                                                                   |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Single PymtHouse goes down           | Apps with ≥ 2 connected signers: everything. Apps with default-only: auto-reconnect tries another on next session.                         |
| Dashboard goes down                  | All in-flight apps keep working on cached tokens. New onboarding breaks until Dashboard returns.                                           |
| Free-tier PymtHouse goes down        | HA pair minimises outage window. Users with any other connected PymtHouse unaffected.                                                      |
| Dashboard credentials leak (hybrid)  | Attacker can request new access tokens via refresh, but cannot sign directly. PymtHouses revoke per-user refresh tokens out-of-band.       |
| Single PymtHouse compromised         | Only users connected to *that* PymtHouse are exposed. Scope `sign:job` limits blast radius to signing on their behalf, not draining wallets — provided custody disclosure honours per-user wallet or equivalent. |
| Every PymtHouse is offline           | Direct-ETH users unaffected. Everyone else fails — this is the fallback layer's job, not the signer layer's.                               |

## Out of scope

- **Enterprise-grade PymtHouse.** Operators building enterprise offerings run their own single-tenant instance — per [developer-dashboard.md](./developer-dashboard.md#not-a-commercial-saas-product).
- **Ticket UX at the protocol level** — e.g. ERC-20 stablecoin settlement. Parallel protocol track; simplifies every PymtHouse when it lands, but not gating here.
- **On-chain reputation / slashing for misbehaving signers.** Defer until there are enough signers to care.
- **Cross-PymtHouse settlement / arbitrage.** Each connected PymtHouse is its own billing boundary; the Dashboard surfaces usage per-provider rather than aggregating.

## Open questions — how the current implementation relates to the Dashboard

### Spec-shape questions

- **M2M vs front-channel OIDC.** Today, the Dashboard holds an `m2m_*` client secret per PymtHouse it integrates with, used server-side to upsert users and mint tokens on their behalf. An alternative is pure front-channel OIDC: the user clicks "Connect *this PymtHouse*" → redirected to the PymtHouse's consent page → redirected back with an authorisation code, no shared secret between Dashboard and PymtHouse. Front-channel makes "bring your own PymtHouse" permissionless — any spec-conformant PymtHouse is immediately connectable by any Dashboard. The trade-off is a one-time explicit signup at each PymtHouse instead of Dashboard-transparent account creation. Which trade do we want? Is the explicit signup the *correct* friction that signals "separate entity, not Dashboard extension"?
- **Can the device flow complete without M2M at all?** This is the sharpest form of the previous question. If yes, we get permissionless PymtHouse registration for free.
- **PymtHouse spec conformance test-suite shape.** Where does the spec live, who runs the conformance tests, how are results published so the Dashboard can decide who to list?

### Custody + trust questions

- **Custody disclosure on the Dashboard card.** A PymtHouse is custodial by definition — it holds funds and signs on users' behalf. The pymthouse wiki today doesn't disclose the wallet custody model (per-user vs per-tenant vs pooled; HSM vs KMS vs encrypted-at-rest DB; admin access model; max balance exposure; incident response). The Dashboard has a natural slot on each PymtHouse card to render this. Should the spec require every conformant PymtHouse to publish a signed custody disclosure, and should the Dashboard render it on the connect flow?
- **Custody-disclosure verification.** The operator signs its own disclosure — who (if anyone) independently verifies it stays accurate over time? Audit cadence, on-chain attestation, community watchdog, or pure observable-behaviour enforcement?
- **Admin-security posture as a user-facing signal.** Pymthouse admin auth today is `next-auth` (Google / GitHub / Bearer). For the Foundation-hosted default that the Dashboard auto-connects every new user to, what's the minimum bar — hardware-backed 2FA (WebAuthn), named admin roster, quorum approval for key-touching operations? And should the Dashboard surface admin-security posture on third-party PymtHouse cards so developers can choose?

### Tenancy + surface questions

- **Community vs enterprise PymtHouse in the Dashboard UI.** The free-tier community PymtHouse is multi-tenant (many developers share one instance; admin costs amortise). An enterprise is more naturally a single-tenant PymtHouse they run themselves. How should the Dashboard distinguish these — same card with a tier indicator, a different section, or surfaced only via the ecosystem page (Frameworks, Daydream, Blue Claw, Streamplace)?
- **Per-user revocation UX.** Device lost, credential leaked — what's the path from "oh no" to "cleanly revoked," who drives it (Dashboard? PymtHouse?), and how long does it take?

### SDK-policy questions

- **Routing policy defaults.** Price-weighted, priority-first, round-robin? Default matters because most users won't configure.
- **SDK failover ergonomics.** How aggressively to retry across PymtHouses? Too few retries → transient blips become user-visible; too many → real failures get masked.
- **Free-tier rate-limit dimension.** Per user, per IP, per API token? Different abuse/UX profiles.
