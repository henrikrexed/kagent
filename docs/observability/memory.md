# Memory observability

kagent emits dedicated OpenTelemetry spans for the agent memory subsystem, alongside
the existing `gen_ai.*` spans on `invoke_agent`. These spans give operators direct
visibility into when and how an agent reads, writes, and consolidates long-term memory,
and are designed as a **governance surface** — not just operation timing.

The spans are produced by the Go ADK memory service
(`go/adk/pkg/memory/kagent_service.go`) using helpers in
`go/adk/pkg/telemetry/memory.go`. They are only exported when tracing is enabled
(`OTEL_TRACING_ENABLED=true`, see [helm/kagent/values.yaml](../../helm/kagent/values.yaml)
`otel.tracing`).

## Spans

| Span name | Emitted by | `memory.operation` | Parent |
|-----------|------------|--------------------|--------|
| `memory.read` | `SearchMemory` (recall) | `prefetch` | the active `invoke_agent` span, when recall happens before LLM dispatch |
| `memory.write` | `AddSessionToMemory` | `save` | current span in context |
| `memory.consolidate` | `summarizeContent` (LLM fact extraction) | `extract` | its parent `memory.write` |
| `memory.embed` | query/content vectorization | `embed` | the active `memory.read` / `memory.write` span |

`memory.read`/`memory.write` time is dominated by vectorizing the query/content, not
by the pgvector search or store. `memory.embed` breaks that out as an explicit child so
the embed-vs-search/store split is visible instead of one opaque block.

`memory.read` is started with the caller's context, so when recall runs before the
model is invoked it attaches as a **child of `invoke_agent`**. This keeps the trace
tree connected with the spans users already see in Dynatrace / Honeycomb / Tempo.

## Attributes

All memory spans carry the SUT (system-under-test) descriptor for the backend:

| Attribute | Value | Notes |
|-----------|-------|-------|
| `memory.sut.name` | `kagent` | |
| `memory.sut.architecture` | `vector` | |
| `memory.sut.store_backend` | `pgvector` | matches the controller's pgvector store |
| `memory.operation` | `save` / `prefetch` / `extract` | operation performed |
| `memory.scope` | `user` | kagent scopes memory by user within an agent namespace |
| `memory.index_ref` | `<agent name>` | the logical memory index targeted |

Operation-specific attributes:

| Span | Attribute | Value |
|------|-----------|-------|
| `memory.write` | `memory.source` | `user` (raw session text) or `agent_inference` (LLM-summarized facts) |
| `memory.write` / `memory.consolidate` | `memory.item.count` | number of items stored / extracted |
| `memory.read` | `memory.item.count` | number of memories returned |
| `memory.read` | `memory.injection_result` | `injected` (≥1 memory passed the pgvector min-score gate) or `filtered` (none passed) |
| `memory.read` | `memory.query.top_k` / `memory.query.min_score` | the pgvector search shape used (limit + min-score gate) |
| `memory.embed` | `memory.item.count` | number of texts vectorized (1 for recall/save, N for batch session ingest) |

### Governance vocabulary and reserved attributes

This instrumentation adopts the `memory.*` governance vocabulary proposed in
[kagent-dev/kagent#1909](https://github.com/kagent-dev/kagent/issues/1909) (see the
memory-semantics discussion). kagent emits **only the attributes it can populate
truthfully** today. The following are part of the convention but **intentionally not
emitted**, because kagent's memory subsystem does not yet model them — emitting them
would mean fabricating values:

| Reserved attribute | Blocked on |
|--------------------|-----------|
| `memory.status` (`raw_trace` / `candidate` / `active` / `superseded` / `revoked`) | a memory lifecycle / state machine |
| `memory.authority` (`cite_only` / `recallable` / `injectable` / `directive`) | an injection-authority model |
| `memory.record_id` on `memory.write` | the store returning a record id on write |
| operations `promote` / `revoke` | memory governance operations |

When kagent gains a memory governance model, these can be emitted without changing the
span names or the existing attribute contract.

## A2A (delegation) span attributes

Cross-agent delegation is already wrapped by the ADK `execute_tool <subagent>` span.
Rather than add a span layer, kagent stamps delegation attributes onto that active span so
it reflects the actual call:

| Attribute | Value |
|-----------|-------|
| `a2a.subagent.name` | the remote agent being delegated to |
| `a2a.context_id` | the A2A context id used for the sub-agent session |
| `a2a.parent_context_id` / `a2a.root_context_id` | conversation lineage (immediate caller / top-of-chain) |
| `a2a.task.id` / `a2a.task.state` | the delegated task id and how it resolved (`completed` / `failed` / `input_required`) |

## Trace verbosity: configurable auto-instrumentation

kagent keeps the standard auto-instrumentation **enabled by default** (upstream parity),
so agent traces carry the same detail they did before — including a2a SDK protocol/queue
spans and outbound httpx client-transport spans. The httpx client spans also carry W3C
trace context on the wire, keeping agent→controller and agent→agent hops stitched into a
single trace.

For operators who want leaner, high-signal-only traces, both auto-instrumentations are
**opt-out** via helm. Trace continuity is preserved even when disabled: the
`_SubagentInterceptor` (A2A) and `inject_trace_context` (memory/session httpx) hooks carry
the W3C correlation headers **without emitting spans**.

| Helm value | Env var (forwarded to agent pods) | Default | Set `false` to drop |
|------------|-----------------------------------|---------|---------------------|
| [`otel.tracing.a2aSdkInstrumentation`](../../helm/kagent/values.yaml) | `OTEL_INSTRUMENTATION_A2A_SDK_ENABLED` (`a2a/utils/telemetry.py`) | `true` | a2a SDK `@trace_class` plumbing spans (~85% of a Python trace) |
| [`otel.tracing.httpxClientInstrumentation`](../../helm/kagent/values.yaml) | `OTEL_INSTRUMENTATION_HTTPX_CLIENT_ENABLED` (`kagent/core/tracing/_utils.py`) | `true` | raw outbound httpx `POST`/`GET` transport spans (~65% of a post-a2a trace) |

```yaml
otel:
  tracing:
    a2aSdkInstrumentation: false      # default true — drop a2a plumbing spans
    httpxClientInstrumentation: false # default true — drop raw transport spans
```

The Go ADK emits only deliberate high-level spans (no decorator auto-instrumentation), so
the a2a toggle is a2a-Python-SDK-specific. The FastAPI **server boundary** span is kept
with the standard ASGI request spans; only the agent-card health-check endpoint is
excluded (high-frequency polling, no diagnostic value).

## Verifying live

See [`docs/verification/kagent-after.dql`](../verification/kagent-after.dql) for a
Dynatrace DQL query that lists memory spans and their governance attributes, and
[`docs/verification/kagent-baseline.dql`](../verification/kagent-baseline.dql) for the
pre-instrumentation baseline (zero `memory.*` spans).
