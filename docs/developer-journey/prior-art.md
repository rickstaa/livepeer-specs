# Prior Art: Discovery → Playground → Integrate

> [!WARNING]
> **Draft — AI-assisted, not fully verified.** This document was produced with AI assistance and has not been fully fact-checked. Details (pricing, SDK shapes, URLs, behaviour) may be inaccurate or out of date. Verify against the linked sources before relying on any specific claim.

## Thesis

Every serious inference portal of the last five years has converged on the same three-step funnel: a browsable **catalogue of capabilities**, an **in-browser playground** that runs before or immediately after signup, and a **single-call SDK** that maps one-to-one onto what the playground just did. The differences sit in two axes the Developer Dashboard has to take a position on: (1) whether the unit of discovery is a *capability* (a model, a pipeline) or a *solution* (a hosted app, an endpoint bundle), and (2) who owns billing, compute, and the publish flow. Replicate is the canonical capabilities-first portal; Modal is the counter-example that optimises for "deploy arbitrary code" instead of "browse the menu"; Chutes and Prime Intellect are the two most visible decentralised-GPU attempts at the same pattern; Fal.ai is the real-time/video specialist. The Developer Dashboard's bets — capabilities-first, community-owned, remote-signer billing, opinionated Pipeline SDK, minimum viable centralisation — each map cleanly to a "adopt" or "deliberately avoid" from one of these five.

## At-a-glance

| Platform | Unit | Playground before signup | Decentralised compute | Authors self-serve |
|---|---|---|---|---|
| Replicate | Capability (model) | Yes | No (single operator) | Yes, via Cog push |
| Modal | Code / function | No (deploy-first) | No (single operator) | Yes, by deploying |
| Chutes | Capability (model/chute) | Partial (public chats) | Yes (Bittensor SN64) | Yes, `chutes deploy` |
| Prime Intellect | Compute + inference endpoint | Limited | Yes (marketplace) | Limited / curated |
| Fal.ai | Capability (model) | Yes | No (single operator) | Curated + some self-serve |

---

## Replicate

**Discovery / catalog.** Replicate's home is a grid of models grouped by task (text-to-image, video, audio, language, etc.) with an "Explore" page, featured collections, and per-creator profiles at `replicate.com/<user>/<model>`. Search is model-name and tag based; the URL shape itself communicates the capabilities-first stance. Every model page is the atomic unit — one capability, one README, one playground, one "API" tab. ([replicate.com/explore](https://replicate.com/explore))

**Playground.** Visible without an account. The input form is **auto-generated from the model's OpenAPI schema** exposed by Cog, which is why every Replicate model has a consistent UI regardless of author. You can run at least one prediction before being asked to authenticate (behaviour varies by model/traffic — unclear from public docs where the exact gate sits today). ([replicate.com/docs](https://replicate.com/docs))

**Auth & key issuance.** GitHub OAuth is the dominant path; email also works. API tokens are issued immediately from `replicate.com/account/api-tokens` after signup — order of minutes from landing to a working `curl`. ([replicate.com/account/api-tokens](https://replicate.com/account/api-tokens))

**Pricing / billing.** Per-second GPU billing, rate published per hardware SKU (CPU, T4, A40, A100, H100 tiers). A handful of high-traffic models are priced per-run or per-token instead. Credits are prepaid; the pricing page surfaces the rate on every model page in the "Run time and cost" panel. ([replicate.com/pricing](https://replicate.com/pricing))

**SDK shape.** Single-call: `replicate.run("owner/model:version", input={...})` in Python and JS, with a matching raw HTTP API. Streaming, webhooks, and prediction objects are available but optional — the 90% path is one function. Typed clients are community-built; the first-party SDK stays loosely typed to match arbitrary model schemas. ([github.com/replicate/replicate-python](https://github.com/replicate/replicate-python))

**Provider / publisher side.** Authors package a model as a **Cog** container (Python class with typed `predict()` + `cog.yaml`) and push with `cog push r8.im/<user>/<model>`. Fully self-serve, no approval gate for public models. Revenue share is not the default model — Replicate bills the caller and the author generally does not earn per-call unless on a specific partner arrangement. ([github.com/replicate/cog](https://github.com/replicate/cog))

**Decentralisation.** None. Replicate runs the compute on its own infrastructure (backed by GPU clouds). Single operator.

**Takeaway for the Developer Dashboard.** Adopt almost everything about the shape: capability-as-URL, OpenAPI-driven playground, Cog-equivalent container contract (the Developer Dashboard's Pipeline SDK), one-call SDK. The key departure is billing — Replicate's platform-managed billing is exactly what the Developer Dashboard is *not* doing.

## Modal.com

**Discovery / catalog.** There is no browseable catalogue of capabilities in the Replicate sense. Modal's front door is docs and examples; discovery happens through the example gallery and blog, not a searchable model index. The unit is "a function you deployed", not "a capability someone else published". ([modal.com/docs/examples](https://modal.com/docs/examples))

**Playground.** None in the Replicate sense. You cannot try a stranger's Modal function from a public form; you can read example code, copy it, and deploy it yourself. Try-before-signup is docs and code samples, not hosted runs.

**Auth & key issuance.** GitHub/Google OAuth, then `modal token new` from the CLI writes a token locally. Landing to deployed function is minutes but requires local Python and CLI, not just a browser. ([modal.com/docs/guide](https://modal.com/docs/guide))

**Pricing / billing.** Per-second compute, priced per CPU core, GB RAM, and GPU SKU separately (H100, A100, A10G, L4, T4, etc.). Free tier of monthly credit. Billing is platform-managed, card on file. ([modal.com/pricing](https://modal.com/pricing))

**SDK shape.** Not a client SDK at all — an **authoring** SDK. `@app.function(gpu="H100")` decorators turn Python functions into remote endpoints; `modal deploy` ships them. Calling someone else's function is possible via `Function.lookup()` but isn't the headline pattern.

**Provider / publisher side.** Everyone is a publisher; there is no catalogue to publish *to*. Sharing is done by publishing code (GitHub) or by exposing a web endpoint from a Modal app. No approval flow, no revenue share — you pay Modal for the compute your function consumes.

**Decentralisation.** None. Single operator on top of hyperscaler GPU supply.

**Takeaway for the Developer Dashboard.** Deliberately avoid. Modal's "run any code" generality is the reason it cannot offer a consistent playground or an honest catalogue — exactly the funnel the Developer Dashboard exists to provide. The Developer Dashboard's opinionated Pipeline SDK (Python class → container → OpenAPI schema) is the trade: less flexibility than Modal, far more discovery and playground consistency.

## Chutes

**Discovery / catalog.** `chutes.ai` lists "chutes" (deployed model endpoints) with a browseable model index skewed toward open-weights LLMs, image, and audio. Categorisation is lighter than Replicate; the surface is closer to a model list than a curated taxonomy. Free public chat UIs exist for several top LLMs. ([chutes.ai](https://chutes.ai))

**Playground.** Partial. Chat-style playgrounds for popular LLMs are reachable without signup; arbitrary-model playgrounds typically need an account and credits. Input forms for non-chat models are hand-assembled per chute rather than universally schema-driven (unclear from public docs how strictly this is enforced).

**Auth & key issuance.** Email / wallet signup; API keys issued from the dashboard. Tight loop — minutes from landing to first call.

**Pricing / billing.** Per-token for LLMs, per-step or per-second for image/video, billed from a prepaid credit balance. Underlying compute is supplied by miners on **Bittensor subnet 64**; the user-facing price is set by the Chutes frontend, not the miner. ([docs.chutes.ai](https://docs.chutes.ai))

**SDK shape.** OpenAI-compatible HTTP for LLMs (so any OpenAI SDK drops in with a base-URL swap) plus a native Python client for non-LLM chutes. Single-call per chute.

**Provider / publisher side.** Self-serve: `chutes deploy` publishes a Python class as a chute, analogous to Cog push. Authors can in principle earn from miner rewards / subnet emissions rather than per-call platform revenue share — the economic model is subnet-mediated, not Stripe-mediated. ([docs.chutes.ai/developers](https://docs.chutes.ai/developers))

**Decentralisation.** Yes, via Bittensor SN64 — miners run the inference, validators score, emissions flow through TAO. The frontend (chutes.ai) is still a centralised product surface over a decentralised execution layer. Closest published analogue to the Developer Dashboard's "centralised funnel over decentralised network" stance.

**Takeaway for the Developer Dashboard.** Study carefully. Chutes shows the pattern works end-to-end: a clean capabilities-first portal can sit over a decentralised compute market without leaking the complexity to developers. The Developer Dashboard's remote-signer / clearinghouse model is the equivalent plumbing for Livepeer's payment rails.

## Prime Intellect

**Discovery / catalog.** Two product surfaces: a **GPU compute marketplace** (rent H100s/A100s by the hour from a pool of providers) and a smaller **inference / model** surface. The inference catalogue is narrower than Replicate or Chutes — curated flagship models (including Prime Intellect's own trained models such as INTELLECT-*) rather than a long tail. ([primeintellect.ai](https://www.primeintellect.ai))

**Playground.** Limited. Chat UIs exist for hosted LLMs; broad any-model playground in the Replicate sense is not the main product. Unclear from public docs whether playgrounds are reachable pre-signup.

**Auth & key issuance.** Email / OAuth, dashboard-issued API keys. Standard cloud-console loop.

**Pricing / billing.** Per-hour GPU pricing for the compute marketplace; per-token for hosted inference endpoints. Billing is platform-managed credits. ([primeintellect.ai/pricing](https://www.primeintellect.ai))

**SDK shape.** OpenAI-compatible API for inference; compute marketplace is driven via web console + REST. No single unifying capability-call SDK of the Replicate shape.

**Provider / publisher side.** Compute providers onboard to supply GPUs to the marketplace. Model publishing by third parties is limited / curated rather than open self-serve — Prime Intellect's brand is "we train and serve frontier open models on decentralised compute", not "anyone can ship a model here". (Unclear from public docs whether an open self-serve publishing flow exists.)

**Decentralisation.** Yes for compute supply (distributed providers, including training runs coordinated across untrusted nodes). Inference catalogue itself is centrally curated.

**Takeaway for the Developer Dashboard.** Confirms the Developer Dashboard's "minimum viable centralisation" framing: a decentralised compute layer does not preclude a centrally-curated, high-signal catalogue at the top. Where Prime Intellect leans curated, the Developer Dashboard should lean self-serve (closer to Replicate/Chutes) because the whole point is community ownership.

## Fal.ai

**Discovery / catalog.** A model gallery at `fal.ai/models` heavy on image and video (Flux variants, SDXL, video-gen, voice) with task-based filtering. Real-time inference — particularly for image and video — is the explicit positioning. ([fal.ai/models](https://fal.ai/models))

**Playground.** Yes, browsable pre-signup with live previews. Input forms are schema-driven per model and consistent across the catalogue. Latency itself is the demo — many playgrounds stream partial results.

**Auth & key issuance.** GitHub / Google OAuth; key issued from dashboard. Fast.

**Pricing / billing.** Per-second GPU or per-megapixel / per-request for some models, credit-based prepaid. Rate shown on model pages. ([fal.ai/pricing](https://fal.ai/pricing))

**SDK shape.** `fal.subscribe("model-id", { input })` in JS and a matching Python client — single-call, with first-class streaming and websocket variants for realtime use cases. OpenAPI schemas per model back the typed wrappers.

**Provider / publisher side.** Mix of Fal-hosted flagship models and a growing self-serve publishing path (author packages a Python app, deploys to Fal). More curated than Replicate, less than Modal. (Exact self-serve terms unclear from public docs.)

**Decentralisation.** None. Single operator.

**Takeaway for the Developer Dashboard.** Fal is the proof that a capabilities-first portal can be organised around **latency-class** (realtime vs. batch) rather than just task — directly relevant to Livepeer's realtime video pipelines. The Developer Dashboard should treat realtime capabilities as first-class surface objects, not footnotes on a generic model page.

## Brief mentions

- **Together.ai**: OpenAI-compatible inference for a curated list of open-weights LLMs; per-token billing; playground behind signup; no third-party model publishing. Relevant only as the "OpenAI-compatible-endpoint" baseline every LLM-adjacent capability on the Developer Dashboard will be compared against.
- **Hugging Face Inference Endpoints**: Deploy-any-model-from-the-Hub as a dedicated endpoint, billed per-hour of reserved hardware. Adjacent to Modal in shape (you pay for the machine, not the call). The HF Hub itself is the closer analogue to a capabilities catalogue, but Inference Endpoints is the billing product.
- **RunPod Serverless**: Bring-your-own-container GPU serverless, per-second. No catalogue. Interesting as a reference for Pipeline-SDK-container contracts; not interesting as a discovery funnel.

## Possible learnings for the Developer Dashboard

These are hypotheses drawn from the five platforms above, not decisions. Each bullet is a candidate direction worth testing against the Developer Dashboard's constraints before committing.

- **Capabilities-first URL shape, Replicate-style.** One capability per page, OpenAPI-driven playground, "Run" / "API" / "README" tabs could be the load-bearing discovery primitive, with search and SDK ergonomics inheriting from it. *(Learning from: Replicate.)*
- **Pipeline SDK as the Cog-equivalent, deliberately opinionated.** A Python class with a typed `run()` compiling to a container with an OpenAPI schema — which in turn generates the playground form — may be the single decision that keeps the catalogue consistent where Modal's cannot be. Worth considering whether to resist broadening it into "run any code". *(Learning from: Modal, inverted.)*
- **Frontend-curated, execution-decentralised — and say so out loud.** Chutes suggests developers may not care where the GPU is as long as the funnel is clean. A centralised portal over Livepeer's decentralised network could be the right trade, with the path-to-decentralised surfaced as a visible design element rather than a roadmap line. *(Learning from: Chutes.)*
- **Billing is not platform-managed; signers are pluggable.** Remote-signer / clearinghouse routing in place of Stripe-on-file. An SDK that carries a token / signer config instead of a platform API key — with support for registering multiple signers — would be the sharpest departure from all five platforms and likely warrants its own top-level explanation in the spec. *(Learning from: none of them; this is the Developer Dashboard's own bet.)*
- **Self-serve publishing from day one.** Replicate- and Chutes-style `push` flows rather than Prime Intellect-style curation. If community ownership is the goal, the catalogue likely needs to be populated by the community, and gatekeeping the publish path could kill the flywheel. *(Learning from: Replicate / Chutes vs. Prime Intellect.)*
- **Realtime as a first-class capability class, not a footnote.** Fal's structure suggests latency-class may be a legitimate top-level axis. Livepeer's realtime video pipelines could surface as their own catalogue section with streaming-aware playgrounds rather than being shoehorned into a request/response template. *(Learning from: Fal.)*
- **Investigate further: revenue share for capability authors.** None of the five cleanly solve "author earns per call" without a platform taking custody of billing. The remote-signer model may make this natively possible (token flow could split on-chain) — worth a dedicated design note before committing either way.
