# Client SDK (gateway SDK)

*Status: proposal — design captured, not yet ratified. Initial Python implementation lives at [`livepeer-python-gateway`](https://github.com/livepeer/livepeer-python-gateway) (today's `start_lv2v` entry point). This spec covers the v1 shape — `run` / `stream` / `live` — that would replace it. Subject to change while [open questions](#open-questions-before-implementation) are resolved.*

The local, language-specific SDK a developer uses to **request jobs** from the Livepeer network. It is the request-side counterpart to the [Pipeline SDK](./pipeline-sdk.md) (which is how job types get *onto* the network). Comparisons to other platforms live in [landscape.md](./landscape.md).

## Problem

Today, calling the network requires understanding orchestrators, discovery, gRPC or trickle transport, and — worst — running a local gateway to sign payment tickets. Most developers bounce before writing their first call.

The shipped Python implementation already abstracts orchestrator selection, capability discovery, and trickle transport, but its public surface is hard-coded to live video-to-video (`start_lv2v`). Each new BYOC capability would need a new top-level function. The v1 design replaces that with a capability-parameterised API that matches the verb idioms users bring from Replicate / fal / OpenAI.

## Main interfaces

What a caller writes:

```python
from livepeer.client import Gateway

gw = Gateway(token=TOKEN)                # or signer_url=, discovery_url=, orch_url=

# Single-shot HTTP — matches Replicate / fal
result = gw.run("sentiment", input={"text": "hello"}, model="distilbert-base")

# SSE streaming — plain iteration
for evt in gw.stream("llm-chat", input={"messages": [...]}, model="qwen-2.5-7b"):
    print(evt.data)

# SSE streaming — context manager for early-exit safety (matches OpenAI / Anthropic)
with gw.stream("llm-chat", input=..., model=...) as stream:
    for evt in stream:
        if evt.event == "done":
            break

# Live trickle (bidirectional, LiveKit-style)
async with gw.live("live-video-to-video", model="noop") as session:
    media_in  = session.publish(fps=30)
    media_out = session.subscribe()
    await session.control.write({"prompt": "a cat"})
    async for evt in session.events():
        ...
```

**Three verbs, three resource models:**

| Verb | Holds resources? | Idiom | v1 status |
|---|---|---|---|
| `gw.run(capability, *, input=, model=, timeout=)` → `dict` | No | Plain return value, no `with` | API present; raises `NotImplementedError` until BYOC HTTP routing lands |
| `gw.stream(capability, *, input=, model=)` → `StreamResponse` | Yes — open HTTP connection | **Iterable AND `with`-able** (mirrors OpenAI / Anthropic) | API present; raises `NotImplementedError` until BYOC SSE routing lands |
| `gw.live(capability, *, model, params=, request_id=, stream_id=)` → `LiveSession` | Yes — 4 channels + payment task | `async with` | **Implemented** for `live-video-to-video` (delegates to existing `start_lv2v`) |

Asymmetric on purpose: `with` matches expectations for sessions and SSE streams (LiveKit, OpenAI streaming, Anthropic, httpx, aiohttp). `run` is a plain return value because it doesn't hold resources, matching Replicate / fal expectations.

**Async variants** are first-class — every major SDK (Replicate, OpenAI, fal) ships them:

| Sync | Async |
|---|---|
| `gw.run(...)` | `await gw.run_async(...)` |
| `gw.stream(...)` | `await gw.stream_async(...)` → `AsyncStreamResponse` |
| `gw.live(...)` | already async-native |

Design principle: **call a capability by name, get the right transport automatically based on what the capability emits.**

## Public API

### `Gateway` — config carrier

```python
class Gateway:
    def __init__(
        self,
        *,
        token: str | None = None,                       # base64 JSON config bundle
        signer_url: str | None = None,
        signer_headers: dict[str, str] | None = None,
        discovery_url: str | None = None,
        discovery_headers: dict[str, str] | None = None,
        orch_url: str | Sequence[str] | None = None,
    ) -> None: ...
```

Selection / discovery is inherited from the existing `orchestrator_selector`. Precedence: `orch_url` → `discovery_url` → token's `discovery` → signer's `/discover-orchestrators`. **Users do not need to specify an orchestrator** — passing only `token=` or `signer_url=` works.

Payment routing goes through a remote-signer clearinghouse, partnered-with rather than locally hosted. See [remote-signer.md](./remote-signer.md).

### `StreamEvent` and `StreamResponse`

`stream()` does **not** yield raw `dict`s. Real SSE SDKs (OpenAI, Anthropic, Replicate, sseclient-py, httpx-sse) yield typed event objects exposing all SSE fields:

1. Not every SSE payload is JSON — BYOC pipelines may emit text deltas (LLM tokens), bytes, or keep-alives. Pre-decoding to `dict` breaks for those.
2. The `event` field is how SSE distinguishes message types (e.g. `progress` vs `done`). Throwing it away is lossy.
3. The `id` field is needed for resumption via `Last-Event-ID`.

```python
@dataclass(frozen=True)
class StreamEvent:
    data: str                           # raw SSE data field
    event: str | None = None            # SSE event type
    id: str | None = None               # SSE event id (for resumption)
    retry: int | None = None

    def json(self) -> Any:              # convenience for JSON-payload pipelines
        return json.loads(self.data)


class StreamResponse:
    """Sync iterator + context manager. Mirrors OpenAI's stream pattern."""
    def __iter__(self) -> Iterator[StreamEvent]: ...
    def __enter__(self) -> "StreamResponse": ...
    def __exit__(self, *exc) -> None: ...      # closes the HTTP connection
    def close(self) -> None: ...


class AsyncStreamResponse:
    """Async iterator + async context manager."""
    def __aiter__(self) -> AsyncIterator[StreamEvent]: ...
    async def __aenter__(self) -> "AsyncStreamResponse": ...
    async def __aexit__(self, *exc) -> None: ...
    async def aclose(self) -> None: ...
```

### `LiveSession` — four-channel session

```python
class LiveSession:
    manifest_id: str | None
    raw: dict[str, Any]
    control: ChannelWriter | None      # always exposed if URL present
    events: ChannelReader | None       # always exposed if URL present

    async def __aenter__(self) -> "LiveSession": ...
    async def __aexit__(self, *exc) -> None:    # auto-cleanup

    def publish(self, *, fps: float = 30.0, **media_kwargs) -> MediaPublish: ...
    def subscribe(self, **kwargs) -> MediaOutput: ...

    async def close(self) -> None: ...
```

| Channel | Surface | Backed by |
|---|---|---|
| Media in (caller → orch) | `session.publish(fps=...)` → `MediaPublish` | `TricklePublisher` |
| Media out (orch → caller) | `session.subscribe()` → `MediaOutput` | `TrickleSubscriber` |
| Control (caller → orch, JSONL) | `session.control` | `JSONLWriter` |
| Events (orch → caller, JSONL) | `session.events` | `TrickleSubscriber` (JSONL) |
| Payments | implicit, auto-started in `__aenter__` | `PaymentSession` + `_payment_sender` task |

Non-video live capabilities (deferred): `publish` / `subscribe` will fall back to raw `TricklePublisher` / `TrickleSubscriber` when the capability isn't video. v1 only supports video; the contract is forward-compatible.

## Why `live()` over today's `start_lv2v`

| Concern | `start_lv2v` (today) | `gw.live(...)` (proposed) |
|---|---|---|
| BYOC fit | Hardcoded to LV2V; new live capabilities need new functions | Capability is a string parameter; zero new public API needed |
| Lifecycle | Manual `try/finally: await job.close()`; every example repeats it; forgetting leaks 4 trickle connections + payment task | `async with` makes cleanup automatic, including on exception paths |
| Input shape | `StartJobRequest` dataclass with fixed fields | `params: dict` — capability defines its own schema |
| Config repetition | Each call re-passes `token`, `signer_url`, `discovery_url` | `Gateway(...)` once, reuse for N jobs |

The BYOC capability-parameterisation is the load-bearing reason. `async with` and `Gateway` are correct independently and don't add design risk; they're standard Python idioms applied honestly to the resource model that already exists.

## Packaging — three distributions, no `livepeer.gateway`

The repo will be reorganised into a `uv` workspace publishing three distributions under a shared `livepeer.*` namespace, in lockstep with the [Pipeline SDK's monorepo migration](./pipeline-sdk.md).

| Distribution | Imports as | Contents | Audience |
|---|---|---|---|
| `livepeer-client` | `livepeer.client` | `Gateway`, `LiveSession`, `run` / `stream` / `live`, capability + selection + payment glue, media helpers | Anyone requesting jobs |
| `livepeer-runner` | `livepeer.runner` | `Pipeline`, `LivePipeline`, `serve()` | BYOC pipeline authors (see [pipeline-sdk.md](./pipeline-sdk.md)) |
| `livepeer-trickle` | `livepeer.trickle` | `TricklePublisher`, `TrickleSubscriber`, `SegmentReader`, JSONL channel helpers | Shared by client and runner |

**Why no `livepeer.gateway`:** the current `livepeer-gateway` distribution is misnamed — it doesn't contain a gateway, it contains code that *talks to* a gateway/orchestrator. `livepeer.client` corrects that. `livepeer-trickle` earns a separate distribution because both client and runner need it; a separate gateway package has no second consumer.

**Layout:**

- `packages/livepeer-client/src/livepeer/client/` (PEP 420 namespace; **no** `__init__.py` at `src/livepeer/`)
- `packages/livepeer-trickle/src/livepeer/trickle/`
- Workspace bootstrap: `[tool.uv.workspace] members = ["packages/*"]` at root, with `livepeer-gateway` becoming a deprecation shim.

## Mapping from current code

| Today (`src/livepeer_gateway/`) | New home |
|---|---|
| `lv2v.py`, `orchestrator.py`, `selection.py`, `capabilities.py`, `errors.py`, `remote_signer.py`, `orch_info.py` | `livepeer.client` (rename `start_lv2v` → internal; expose `Gateway`) |
| `media_publish.py`, `media_output.py`, `media_decode.py` | `livepeer.client` (video helpers stay user-facing) |
| `trickle_publisher.py`, `trickle_subscriber.py`, `segment_reader.py`, `channel_writer.py`, `channel_reader.py`, `control.py`, `events.py` | `livepeer.trickle` (shared with runner) |
| `lp_rpc_pb2*.py`, `codegen.py` | `livepeer.client._proto` (private) |

## Backwards compatibility

`livepeer-gateway` distribution keeps shipping for one release as a deprecation shim — every public name from today's `livepeer_gateway/__init__.py` is preserved by re-export from `livepeer.client` and `livepeer.trickle`, with a `DeprecationWarning` on import. Existing users get zero code changes and one release of grace.

## Out of scope for v1

| Item | Why deferred |
|---|---|
| `Gateway.submit(...)` handle pattern (`.wait` / `.cancel` / `.id`) | Needs orch-side reattach + cancel semantics designed first |
| `run` / `stream` actual implementations | Pending finalised BYOC HTTP / SSE routing on the orchestrator |
| Non-video live capabilities (raw trickle fallback) | No non-video live capability exists upstream yet |
| Sync wrapper for `live()` | Live jobs are inherently async; a sync wrapper would mask that |

## Open questions before implementation

1. Does the orchestrator expose BYOC HTTP / SSE on `POST /{pipeline_id}` (matching the LV2V pattern), or on a different path like `POST /capability/{name}`? Determines `run` / `stream` routing.
2. Should `Gateway` be importable both as `livepeer.client.Gateway` and re-exported from a `livepeer.client.gateway` module (for module-level `gateway.run(...)` style)?
3. For `live()`, should `publish` / `subscribe` be lazy (created on first access) or eager (created in `__aenter__`)? Today's `start_media` is lazy.
4. Logger naming: `livepeer.client` vs continuing `livepeer_gateway.*`?

## Inspirations surveyed

| SDK | `run` shape | `stream` item type | Async variant |
|---|---|---|---|
| Replicate | `replicate.run(model, input=...)` blocking | `ServerSentEvent(data, event, id)` | `async_run`, `async_stream` |
| fal.ai | `fal_client.run(...)` blocking | `dict` (JSON payloads only) | `run_async`, `stream_async` |
| OpenAI | `client.chat.completions.create(...)` | typed event (`ChatCompletionChunk`); `with stream:` for early-exit safety | `AsyncOpenAI` |
| Anthropic | `client.messages.create(...)` | typed event union; **encouraged via `with stream:`** | `AsyncAnthropic` |
| LiveKit | n/a | n/a | `Room.connect(url, token)` then publish/subscribe to tracks (closest analogue to Livepeer's trickle live jobs) |
| Hyperbolic | OpenAI-compatible — users bring the OpenAI SDK | OpenAI-compatible | OpenAI-compatible |

The `run` / `stream` / `live` split is the honest reflection of Livepeer having three genuinely different request shapes, not two. The `StreamEvent` shape (typed event with `data` / `event` / `id` / `retry`, plus a `.json()` helper) is the consensus across OpenAI, Anthropic, Replicate, and `sseclient-py`.

## Related

- [Pipeline SDK](./pipeline-sdk.md) — the deploy-side counterpart
- [Remote signer](./remote-signer.md) — payment routing
- [Developer Dashboard](./developer-dashboard.md) — same call as the playground ships into your app
- [Landscape](./landscape.md) — Replicate / fal / Modal / Chutes comparisons
- Tracking epic: [livepeer-python-gateway#9](https://github.com/livepeer/livepeer-python-gateway/issues/9)
- Initial implementation PR: [livepeer-python-gateway#6](https://github.com/livepeer/livepeer-python-gateway/pull/6)
