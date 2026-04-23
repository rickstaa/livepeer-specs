# Pipeline SDK

*Status: design validated against Replicate / Modal / Chutes / AI-Runner / Scope; BYOC contract alignment proposed, not yet implemented. MVP is the `runner/` module (~1,100 lines) plus the go-livepeer BYOC changes below.*

The Pipeline SDK is the developer-facing interface for deploying AI capabilities to the Livepeer network. Developers write a standard Python class; the SDK handles trickle transport, protobuf serialisation, Docker packaging, and orchestrator registration. Livepeer's equivalent of Replicate's `cog push` or Chutes' `chutes deploy` — with first-class support for real-time video that those platforms don't offer.

Deploy-side counterpart to the [Client SDK](./client-sdk.md). Comparisons to other platforms live in [prior-art.md](./prior-art.md).

## Main interfaces

What a builder writes:

```python
from livepeer_gateway.runner import Pipeline, Input, Output

class TextToImage(Pipeline):
    pipeline_id = "text-to-image"
    version = "1.0.0"
    gpu = "A100"
    min_vram_gb = 24

    @classmethod
    def prepare_models(cls):
        # download weights at build time
        ...

    def setup(self):
        self.model = load_model("/models/sdxl.safetensors")

    def predict(self,
        prompt: str = Input(description="Image description"),
        steps: int = Input(default=30, ge=1, le=100),
    ) -> Output(type="image"):
        return self.model.generate(prompt, steps=steps)
```

Then: `livepeer push my_pipeline.py`. Done.

**CLI surface:**

| Command | Purpose |
|---|---|
| `livepeer predict <file> -i k=v ...` | Run one prediction locally (no container) |
| `livepeer serve <file>` | Start the container's HTTP server locally for testing |
| `livepeer schema <file>` | Print the JSON Schema the SDK derives from the class |
| `livepeer prepare <file>` | Download model weights (usually run inside Docker build) |
| `livepeer push <file>` | Package, build, and publish the container to the network |

**Endpoints the container exposes automatically** (builder never writes these):

| Endpoint | Purpose | Consumer |
|---|---|---|
| `GET /health` | `{"status": "ready \| loading \| error \| idle"}` | Orchestrator readiness check |
| `POST /predict` | Request-response (or SSE if `predict()` yields) | Gateway job proxy |
| `POST /stream` | Start real-time processing (`StreamPipeline` only) | BYOC orchestrator |
| `GET /schema` | JSON Schema for inputs / outputs | Dashboard UI, agents, client-SDK snippet generation |
| `GET /pipelines` | List all registered pipelines | Multi-pipeline discovery |

**Transport is chosen automatically from the class shape:**

| What you wrote | Transport selected |
|---|---|
| `predict()` returns a value | HTTP request / response |
| `predict()` *yields* values (generator) | SSE (`text/event-stream`) |
| `StreamPipeline.on_frame()` / `on_video_frame()` / `on_audio_frame()` | Trickle subscribe / publish |

Design principle: **write Python, get the right transport automatically.**

## Problem

Shipping a new AI capability onto Livepeer today demands extensive manual work: hand-building a Docker container with the right inference code, manually configuring it to speak the trickle protocol, coordinating with orchestrators to pull and run it, building a frontend so developers can try and integrate it, and writing client libraries and docs to match. The process is manual, undocumented, and inaccessible to most AI developers.

The SDK collapses that to: **write a Python class, run `livepeer push`, done.** The developer never touches trickle, protobuf, Docker, or orchestrator APIs. The schema-first design means AI coding agents can generate Pipeline classes from natural language and integrate the result into the Dashboard — the schema is the contract.

## Pipeline types

### Pipeline (request-response)

For workloads that take an input and return a single result — text-to-image, image-to-image, audio transcription, one-shot inference.

```python
class ImageUpscale(Pipeline):
    pipeline_id = "image-upscale"
    gpu = "T4"

    def setup(self):
        self.model = load_upscaler()

    def predict(self,
        image: bytes = Input(media_type="image/png"),
        scale: int = Input(default=2, choices=[2, 4]),
    ) -> Output(type="image"):
        return self.model.upscale(image, scale)
```

### Pipeline with SSE streaming (LLM)

When `predict()` yields values (is a generator), the SDK automatically streams responses as Server-Sent Events. Matches the pattern used by Replicate, OpenAI, and other LLM-serving platforms.

```python
class TextGenerator(Pipeline):
    pipeline_id = "text-generator"
    gpu = "A100"

    def setup(self):
        self.model = load_llm()

    def predict(self,
        prompt: str = Input(description="The input prompt"),
        max_tokens: int = Input(default=256, ge=1, le=4096),
        temperature: float = Input(default=0.7, ge=0.0, le=2.0),
    ) -> Output(type="text_stream"):
        for token in self.model.generate(prompt, max_tokens=max_tokens, temp=temperature):
            yield token
```

SSE response format:

```text
data: {"output": "Hello", "type": "token"}

data: {"output": " world", "type": "token"}

data: [DONE]
```

SSE is triggered by any of: `predict()` returning a generator (sync or async), `Output(type="text_stream")`, or the client sending `Accept: text/event-stream`.

### StreamPipeline (real-time)

For real-time video / audio processing with continuous frame-by-frame I/O via trickle transport and dynamic parameter updates.

```python
class StyleTransfer(StreamPipeline):
    pipeline_id = "style-transfer"
    gpu = "T4"

    def setup(self):
        self.model = load_style_model()
        self.current_style = "starry_night"

    def on_frame(self,
        frame: bytes,
        style: str = Input(default="starry_night", choices=["starry_night", "mosaic"]),
        strength: float = Input(default=0.8, ge=0.0, le=1.0),
    ) -> bytes:
        return self.model.apply(frame, style, strength)

    def on_params_update(self, params):
        # Called when the control channel sends parameter updates mid-stream
        if "style" in params:
            self.current_style = params["style"]
```

StreamPipelines support multi-modal input via typed callbacks — `on_video_frame(frame, **params)` and `on_audio_frame(frame, **params)` with a declared `inputs = ["video", "audio"]` class attribute. See **Design decisions → Multi-modal StreamPipeline** below.

## Input / Output system

### `Input()`

Annotate `predict()` / `on_frame()` parameters with constraints and metadata:

```python
Input(
    description="Human-readable description",
    default=30,              # Default value (makes parameter optional)
    ge=1,                    # Minimum value (>=)
    le=100,                  # Maximum value (<=)
    min_length=1,            # Minimum string length
    max_length=1000,         # Maximum string length
    choices=[256, 512],      # Allowed values (enum)
    media_type="image/png",  # MIME type for binary inputs
)
```

### `Output()`

Declare the return type of `predict()`:

```python
Output(
    type="image",              # json | image | audio | video | text | text_stream
    media_type="image/png",    # MIME type for binary outputs
    description="Description",
)
```

## Pipeline metadata

Declared as class attributes — identity plus resource hints:

```python
class MyPipeline(Pipeline):
    pipeline_id = "my-pipeline"        # Unique ID (auto-derived from class name if omitted)
    version = "1.0.0"                  # Semantic version
    description = "What this does"     # Human-readable description
    gpu = "A100"                       # GPU type hint
    min_vram_gb = 24                   # Minimum VRAM requirement
    cpu_only = False                   # Set True for CPU-only pipelines
```

This metadata appears in the JSON schema (as `x-pipeline-id`, `x-version`, `x-gpu`, etc.), the `/health` response, the `/pipelines` listing, and the [Developer Dashboard](./developer-dashboard.md) catalog.

## Pipeline lifecycle

```text
1. prepare_models()        ← Docker build time: download weights
2. __init__()              ← Container start: construct instance
3. setup()                 ← Container start: load model into GPU
4. predict() / on_frame()  ← Runtime: handle requests
```

`prepare_models()` is a classmethod called separately from `setup()`. This lets large model files download during Docker build, so container startup only has to load weights into memory.

## Pipeline registry

Pipelines are auto-registered when loaded via the CLI. Manual registration works too:

```python
from livepeer_gateway.runner import PipelineRegistry

PipelineRegistry.register(TextToImage)
PipelineRegistry.register(ImageUpscale)

cls = PipelineRegistry.get("text-to-image")
ids = PipelineRegistry.list()
info = PipelineRegistry.list_with_info()
```

Class-level dict — no metaclass magic, no plugin framework.

## Health states

The `/health` endpoint returns the instance's current state with HTTP status codes:

| State | HTTP | Meaning |
|---|---|---|
| `loading` | 503 | Pipeline is loading model weights |
| `ready` | 200 | Pipeline is ready to accept requests |
| `error` | 503 | Pipeline encountered a fatal error |
| `idle` | 503 | Pipeline is loaded but not yet initialised |

```json
{
    "status": "ready",
    "pipeline_id": "text-to-image",
    "version": "1.0.0"
}
```

## Schema generation

The SDK introspects `predict()` / `on_frame()` signatures to generate JSON Schema automatically. That schema is the **single contract** that drives Dashboard playground UI generation, API docs, agent-driven code generation, and request validation in the serve layer.

`livepeer schema my_pipeline.py` produces:

```json
{
    "title": "TextToImage",
    "type": "object",
    "x-pipeline-id": "text-to-image",
    "x-version": "1.0.0",
    "description": "Generate images from text prompts",
    "properties": {
        "prompt": {
            "type": "string",
            "description": "Text description of the image"
        },
        "steps": {
            "type": "integer",
            "description": "Diffusion steps",
            "minimum": 1,
            "maximum": 100,
            "default": 30
        }
    },
    "required": ["prompt"],
    "x-gpu": "A100",
    "x-min-vram-gb": 24,
    "x-output": {
        "type": "image",
        "media_type": "image/png"
    }
}
```

## What `livepeer push` does

```text
1. Read pipeline.py → find Pipeline subclass
2. Extract predict() / on_frame() signature → generate schema JSON
3. Read dependencies (imports, requirements.txt, or pyproject.toml)
4. Generate Dockerfile:
     - Base image with GPU support + Python
     - Install dependencies
     - Copy pipeline code + model weights
     - Entrypoint: python -m livepeer_gateway.runner.serve my_pipeline:MyPipeline
5. Build container image
6. Push to container registry
7. Register capability on the Livepeer network:
     - Pipeline name, schema, GPU requirements
     - Container image reference
8. Orchestrators discover the new capability → pull container → advertise it
9. The Dashboard detects it → auto-creates catalog entry + playground
```

## Architecture

The SDK lives inside `livepeer-python-gateway` as a `runner/` module:

```text
livepeer-python-gateway/
  src/livepeer_gateway/
    runner/                 # New — run pipelines ON the network (~1,100 lines)
      __init__.py           # Public API exports
      pipeline.py     ~230  # Base classes, health states, modality declarations
      inputs.py       ~110  # Input() / Output() descriptors
      schema.py       ~190  # Signature → JSON Schema
      serve.py        ~550  # HTTP server, SSE, trickle dispatch
      registry.py     ~100  # Pipeline discovery
      decorators.py   ~100  # @pipeline shortcut
      cli.py          ~420  # predict, serve, schema, prepare, push

    # Existing — shared transport (reused by runner)
    trickle_publisher.py · trickle_subscriber.py
    channel_reader.py · channel_writer.py
    control.py · media_decode.py · media_output.py
    lp_rpc_pb2.py · ...
```

Three layers, one repo:

- `livepeer_gateway.runner` — "I run pipelines on the network" (this SDK)
- `livepeer_gateway.gateway` — "I route requests to the network" (future; eventually replaces the Go gateway for multi-tenant Dashboard use)
- `livepeer_gateway` (root) — shared transport primitives (trickle, protobuf, media)

The serve layer is intentionally thin: validate inputs, call into the Pipeline, serialise the response. Streaming I/O is delegated to the existing trickle transport primitives — the SDK reuses all existing transport code and does not reimplement trickle or media handling.

### How `serve.py` bridges pipeline → transport

**Request-response (`Pipeline`):**

- `POST /predict` → deserialise inputs → call `pipeline.predict()` → serialise → respond.
- `GET /schema` → auto-generated input/output schema from the `predict()` signature.
- `GET /health` → status.

**Streaming (`StreamPipeline`):**

- `trickle_subscriber` receives encoded frames from the gateway.
- `media_decode` decodes them.
- `pipeline.on_frame(frame, **params)` runs per frame.
- `media_output` encodes the result.
- `trickle_publisher` sends it back.
- Parameter updates arrive via the control channel → `on_params_update()`.

The provider never imports trickle, protobuf, or media modules.

### End-to-end data flow

**Request-response:**

```text
Developer's app
  → POST https://studio.livepeer.org/v1/run/text-to-image
  → Dashboard → Gateway → Orchestrator
  → Container: serve.py receives HTTP → calls predict() → returns result
  → Result flows back to the developer's app
```

**Real-time streaming:**

```text
User's browser (webcam)
  → WebRTC → Gateway
  → Trickle → Orchestrator
  → Container: trickle_subscriber → on_frame() → trickle_publisher
  → Trickle → Gateway → WebRTC
  → User's browser (AI output)

Param updates: browser → control channel → on_params_update()
```

## Integration with the rest of the stack

- **[Developer Dashboard](./developer-dashboard.md)** consumes the schema to auto-generate catalog entries and playground UIs. Push a Python class → full capability card with zero frontend code.
- **Orchestrator runtime** pulls and runs the container the SDK produces, applies isolation (runc / gVisor / Kata) per image policy, and advertises the capability. Lives in go-livepeer; not covered here.
- **BYOC** is the mechanism by which SDK-built containers register on the network. The contract is documented in **BYOC contract alignment** below.
- **Inference adapter (non-Python path)** — the [naap](https://github.com/livepeer/naap) project's [`livepeer-inference-adapter`](https://github.com/livepeer/naap/tree/main/containers/livepeer-inference-adapter) container is a lighter-weight, less-opinionated way to put an existing inference service on the network via BYOC, without rewriting it as a Python class. The Pipeline SDK is the opinionated, schema-first, agent-legible path for new work; the inference adapter is the wrap-what-you-have path for existing code. Operators run and manage these through the [operator app](https://operator.livepeer.org).

## BYOC contract alignment

The SDK targets **BYOC** as its deployment path. For an SDK-built container to "just work" on the Livepeer network, the SDK's container interface and go-livepeer's BYOC layer must agree on endpoints, request bodies, and control flow. This section is the highest-signal technical content in the spec because it pins the cross-repo contract.

### Three container execution models in go-livepeer

| Model | Description | Container protocol |
|---|---|---|
| Managed | go-livepeer pulls and runs containers via `DockerManager` | Livepeer runner API (OpenAPI-generated) |
| External | Operator runs the container; registers the URL via `Warm()` | Same runner API, remote |
| **BYOC** | Container registers as an external capability; requests are proxied through | HTTP passthrough + trickle streaming |

SDK containers target BYOC.

### Current contract mismatches

**Streaming endpoints:**

| SDK exposes | BYOC calls on container | Match? |
|---|---|---|
| `POST /stream` | `POST {url}/stream/start` | Path mismatch |
| *(none)* | `POST {url}/stream/stop` | Missing |
| `POST /params` | `POST {url}/stream/params` | Path mismatch |
| `GET /health` | `GET /health` | Match |

**Batch jobs:** `POST /predict` works today if the container registers as `http://container:8000/predict`.

**Streaming request body** that BYOC sends includes trickle URLs:

```json
{
  "gateway_request_id": "abc123",
  "subscribe_url": "http://orch:8935/ai/trickle/abc123",
  "publish_url": "http://orch:8935/ai/trickle/abc123-out",
  "control_url": "http://orch:8935/ai/trickle/abc123-control",
  "events_url": "http://orch:8935/ai/trickle/abc123-events",
  "data_url": "http://orch:8935/ai/trickle/abc123-data"
}
```

The SDK's `StreamPipelineServer` doesn't currently parse these URLs or connect to them. **This is the main gap.**

### Design decisions

1. **SDK defines the standard** — don't make container endpoints configurable. The whole point is "it just works." Non-SDK containers can still use BYOC directly with any shape.
2. **One stream endpoint** — simplify from 3 container endpoints to 1. Route stop/params through the trickle control channel instead of separate HTTP calls.
3. **Fix BYOC, not the SDK** — update go-livepeer's BYOC to call the SDK's standard paths. The SDK's developer experience drives the API design.
4. **Protocol versioning** — add a `"protocol": "v1"` field to capability registration for future evolution.

### Proposed changes to go-livepeer (BYOC)

File: `byoc/stream_orchestrator.go`.

1. Change `/stream/start` → `/stream`:

   ```go
   workerRoute := orchJob.Req.CapabilityUrl + "/stream"
   ```

2. Remove direct HTTP calls for stop/params; send control messages through the trickle control channel:

   ```go
   controlMsg := map[string]interface{}{"type": "stop"}
   controlBytes, _ := json.Marshal(controlMsg)
   controlPubCh.Write(bytes.NewReader(controlBytes))
   ```

   ```go
   controlMsg := map[string]interface{}{"type": "params", "data": json.RawMessage(body)}
   controlBytes, _ := json.Marshal(controlMsg)
   controlPubCh.Write(bytes.NewReader(controlBytes))
   ```

3. `monitorOrchStream` sends stop via the control channel instead of HTTP.
4. **Gateway-facing endpoints stay the same.** Clients still call `/process/stream/{id}/stop` and `/process/stream/{id}/update`. Only the orchestrator-to-container contract changes.

Note: there's a comment in `stream_gateway.go` (line 1007–1008) blaming the control publisher for "not sending down full data when including base64 encoded binary data" — almost certainly an implementation bug tied to `FirstByteTimeout` or pipe buffering on large payloads, not a protocol limitation. Control messages for stop/params are small JSON; they won't hit this.

### Proposed changes to the Pipeline SDK

File: `src/livepeer_gateway/runner/serve.py`.

In the `/stream` handler, parse trickle URLs from the request body and use the **existing** trickle primitives:

```python
async def handle_stream_start(self, request):
    body = await request.json()

    sub = TrickleSubscriber(body["subscribe_url"])
    pub = TricklePublisher(body["publish_url"])
    control = ChannelReader(body["control_url"])
    events = ChannelWriter(body["events_url"])
    data = JSONLWriter(body.get("data_url"))

    # control → on_params_update / stop
    async for msg in control:
        if msg.get("type") == "params":
            self.pipeline.on_params_update(msg["data"])
        elif msg.get("type") == "stop":
            break

    # subscribe → on_frame → publish
    async for frame in sub:
        result = self.pipeline.on_frame(frame)
        await pub.write(result)
```

`Control`, `Events`, `TricklePublisher`, `TrickleSubscriber`, `ChannelReader`, `JSONLWriter` already exist. No new trickle code needed.

### Final container contract

After these changes, an SDK-built container needs only:

| Endpoint | Purpose | Called by |
|---|---|---|
| `GET /health` | `{"status": "OK"}` | go-livepeer readiness check |
| `POST /stream` | Receive trickle URLs, start processing | BYOC orchestrator (once) |
| `POST /predict` | Request-response inference | BYOC job proxy |
| `GET /schema` | JSON Schema for inputs/outputs | Dashboard UI, auto-registration |

Everything else flows through trickle channels: input video via `subscribe_url`, output via `publish_url`, params/stop via `control_url`, status via `events_url`, data as JSONL via `data_url`.

### What stays the same

- Gateway-to-client API (`/process/stream/*`, `/process/request/*`).
- Trickle protocol and channel creation.
- Payment / capacity management.
- WHIP/RTMP ingress, WHEP egress.
- SSE data output to clients.
- Capability registration (adds only the optional `protocol` field).
- Orchestrator discovery and selection.

### Implementation order

1. **SDK serve layer** — wire `StreamPipelineServer` to trickle primitives. Python-only; can land independently of go-livepeer.
2. **BYOC simplification** — update `stream_orchestrator.go` to use `/stream` + control channel. Go change.
3. **End-to-end test** — SDK container registered as BYOC capability; streaming works.
4. **Batch jobs** — verify `Pipeline.predict()` works through the BYOC job proxy. Likely works already.
5. **Schema integration** — optionally use `/schema` for auto-registration into the Dashboard catalog.

## Out of scope

- **Modal.com-style "run any code" automated deployment.** Orchestrators remain in the loop — they *opt in* to running a new job type. See the [track README](./README.md#out-of-scope).
- Automatic orchestrator discovery / matchmaking for a newly pushed container.
- Non-Python authoring surfaces for MVP.
- A first-party model / metadata registry. The container *is* the registry until the opinionated-container pattern proves insufficient.

## Open questions

- **Metadata flow to the Dashboard** — push on publish, pull from a live orchestrator, or a lightweight registry? The opinionated-container model points toward pull, but the tradeoff isn't decided.
- **Versioning semantics** — what happens when a builder pushes v2 of an existing capability while v1 is in active use?
- **Minimum metadata** — what a container must declare to earn a Dashboard card, vs. what's optional polish.
- **Plugin discovery via entry_points** — pip-installable pipeline packages are post-MVP. Do we need them at all, or is one-container-one-pipeline fine indefinitely?
- **Progress callbacks** — a callback param on `setup()` would let the Dashboard show loading progress while model weights download. Small work; not MVP.

## Future work

Tracked separately, not blocking v0.1:

| Feature | Motivation | Complexity |
|---|---|---|
| Pipeline chaining | Multi-step workflows in one container | Medium — queue-based executor |
| Artifact declaration | Declarative model dependencies | Small — extend `prepare_models()` |
| Progress callbacks | Dashboard loading UI | Small |
| Plugin discovery | pip-installable pipeline packages | Medium — Python entry_points |
| Pydantic params | Optional alternative to `Input()` | Medium — parallel validation path |
| Synchronised A/V | `on_av_frame(video, audio)` with temporal alignment | Medium — sync buffer design |
| Decorator shortcut | `@livepeer.pipeline(gpu=...)` for simple stateless functions | Small — creates a Pipeline class under the hood |

## Appendix: design rationale

Brainstorm-level reasoning for the choices made above. Secondary material — the main spec stands without it — but useful when a reader wants to know *why* a decision was made, or when the team revisits one.

### Classes, not decorators

Stateful lifecycle (load weights once, predict many times), multi-method interfaces (setup + predict + on_params_update + on_frame), and class-attribute resource declarations all point at classes. Every competing platform that serves GPU models falls back to classes too (`@app.cls` in Modal, `FastAPI` subclass in Chutes), because models are inherently stateful. A `@pipeline` decorator exists as a shortcut for simple stateless functions — it creates a class under the hood.

### `Input()` descriptors, not Pydantic

Both produce identical JSON Schema; the choice is developer experience. `Input()` on the method signature keeps all parameters visible at a glance, matches Replicate's Cog pattern (which the largest AI-developer community already knows), and avoids the extra parameter-class boilerplate. Pydantic compatibility can be added later as an optional alternative without breaking changes.

### Three transports, auto-selected

Batch, SSE, and trickle each serve a workload the others can't. The SDK picks the right one from the class shape (return vs yield vs `StreamPipeline`) so the developer writes Python and gets the correct transport without having to choose. WebSocket isn't added because trickle already covers bidirectional streaming and integrates with orchestrator / payment rails; no competing platform uses WebSocket for its primary serving path.

### Multi-modal StreamPipeline

Typed callbacks (`on_video_frame`, `on_audio_frame`) with a declared `inputs = [...]` class attribute. The serve layer dispatches decoded frames by type using the existing `MediaOutput` decoder, which already demuxes audio and video tracks from MPEG-TS trickle streams — no transport change needed. Pattern 1 (one primary stream, one reference stream; state-sharing) ships first. Pattern 2 (`on_av_frame(video, audio)` with temporal alignment) is deferred.

More general than Scope's named-ports approach because the callbacks are format-agnostic rather than tied to video tensor shape `(B, H, W, C)`.

### Pipeline registry and health states

Class-level registry dict; no metaclass magic, no plugin framework. Health states (`LOADING | READY | ERROR | IDLE`) live on the instance and are exposed via `/health` with HTTP status codes. `prepare_models()` is a classmethod, not a declarative artifact system. Metadata is identity + resource hints only — not Scope's full config system. Keep it simple; add declarative layers later if needed.

### SDK boundary

**In the SDK:** base classes, I/O descriptors, schema generation, HTTP serve layer with protocol auto-selection, CLI, registry, health states.

**External:** orchestrator discovery / selection, payment sessions, capability advertisement, container-image-to-model-ID mapping, multi-orchestrator routing.

Rationale: the SDK owns developer experience; the network owns infrastructure. Clean boundary — SDK generates the schema + Docker image, the network uses the schema for routing and the image for deployment.

## References

- [Replicate Cog SDK](https://github.com/replicate/cog) — `BasePredictor`, `Input()`, `ConcatenateIterator`.
- [Modal](https://modal.com/docs) — `@app.cls`, `@modal.enter()`, `@modal.method()`.
- [Chutes SDK](https://github.com/chutes-ai/chutes-sdk) — `Chute(FastAPI)`, `@cord()`, `NodeSelector`.
- [Livepeer AI Runner](https://github.com/livepeer/ai-runner) — `Pipeline` ABC, `PipelineSpec`, `ProcessGuardian`.
- [Scope (Daydream)](https://github.com/daydreamlive/scope) — `Pipeline` ABC, `PipelineRegistry`, `GraphExecutor`.
