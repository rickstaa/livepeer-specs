# Client SDK (gateway SDK)

*Status: stub — initial Python implementation already shipped; this spec covers refinement toward the Developer Dashboard-integrated shape.*

The local, language-specific SDK a developer uses to **request jobs** from the Livepeer network. It is the request-side counterpart to the [Pipeline SDK](./pipeline-sdk.md) (which is how job types get *onto* the network). Also known as the **gateway SDK** — the underlying Python package is `livepeer_gateway`.

## What exists today

An initial Python implementation is live at [`livepeer-python-gateway`](https://github.com/livepeer/livepeer-python-gateway). Key modules:

- `lv2v.py` — `start_lv2v()` orchestrates a live video-to-video job end-to-end.
- `orchestrator.py` · `orch_info.py` · `selection.py` — orchestrator discovery and selection.
- `capabilities.py` — capability querying.
- `remote_signer.py` — client-side integration with the go-livepeer [remote signer](./remote-signer.md); supports both **offchain mode** (no signer) and **on-chain mode** (`--signer <host:port>`).
- `trickle_publisher.py` · `trickle_subscriber.py` — the trickle transport that moves bidirectional media frames.

Nine runnable examples in `examples/` demonstrate the current API surface: `start_job.py`, `write_frames.py`, `camera_capture.py`, `subscribe_events.py`, `payments.py`, `select_orchestrator.py`, and more.

Today's coverage is oriented around **Daydream pipelines** — the live video-to-video (lv2v) surface Daydream runs on — which is why `start_lv2v` is the headline entry point. Extension to arbitrary **BYOC jobs** is already in progress (owned by John), which will bring the full capability surface into reach rather than just lv2v.

This is the starting point. The spec below describes where it needs to go for the [Developer Dashboard](./developer-dashboard.md).

## Problem

Today, calling the network requires understanding orchestrators, discovery, gRPC or trickle transport, and — worst — running a local gateway to sign payment tickets. Most developers bounce before writing their first call.

## Proposed shape

A thin client library that exposes each network capability as a normal function (or typed method), with authentication and payment abstracted away.

Minimum it must do:

- **Authenticate** against the Developer Dashboard via API key (header-based; no custom auth dance).
- **Call a capability** by name and parameters defined in the capability's OpenAPI schema.
- **Route payment** through a remote-signer clearinghouse — partnered-with, not run locally. See [remote-signer.md](./remote-signer.md).
- **Return a typed result** — bytes, URL, structured data, or a stream — based on the capability's declared output type.
- **Stay language-idiomatic** — Python, JavaScript/TypeScript at minimum, others as they land.

The same call the developer runs in the [Developer Dashboard playground](./developer-dashboard.md) is the call that ships into their app. No playground-only runtime; no drift.
