# Orchestrator observations

*Status: stub. The architectural sketch is **J0sh's**, and J0sh is authoring the canonical wire-level spec. This page only captures why the approach is better and the open questions raised when sketching the flow — it intentionally does not invent schemas, signing, or envelope shapes. Those belong in J0sh's doc, not here.*

## Why this is better than today

Current BYOC capability registration is fire-and-forget: a runner POSTs once to `/capability/register` and the orch holds the entry until something — manual or otherwise — clears it. That leaves stale advertisements when a worker disappears and gives no real assurance the registered container is still available.

**Heartbeat-based registration** fixes that:

- Workers send heartbeats periodically; the orch evicts entries whose TTL has elapsed.
- Stale advertisements are impossible — stop heartbeating, stop being advertised.
- Restart is just a fresh heartbeat; no separate deregister step.
- The orch's view of "what is currently being served" becomes a TTL'd liveness cache, not a registration log.

It also composes cleanly with the [pull-based production-data architecture](./production-data.md): the orch's heartbeat-derived registry *is* what it serves on `/observations`. Aggregators pull from orchs they care about; consumers cross-reference; nothing is centrally pushed. One protocol covers both internal routing and external visibility.

## What a heartbeat carries (illustrative, not authoritative)

To make the framing concrete — a heartbeat carries the kinds of information a router or aggregator needs to make a decision: who the runner is, what app it serves, what hardware it has, current load, recent performance, pricing intent, and how long the entry should be considered live.

```json
{
  "runner_id": "...",
  "app":         { "model": "...", "pipeline": "...", "schema_hash": "..." },
  "hardware":    { "gpu": "...", "vram_gb": "..." },
  "capacity":    { "max_concurrent": "...", "active": "..." },
  "performance": { "p50_ms": "...", "success_rate": "..." },
  "endpoint":    { "url": "...", "scheme": "..." },
  "health":      "ready | degraded | draining",
  "ttl_s":       "...",
  "signature":   "..."
}
```

Field names, signing scheme, exact types, and which fields are mandatory vs. optional are defered to J0sh's spec — this is just to anchor the conversation, not to predefine the contract.
