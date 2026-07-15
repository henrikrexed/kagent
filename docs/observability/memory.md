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

## Trace signal-to-noise: a2a SDK plumbing spans

Python agents run on the a2a Python SDK, which auto-instruments its own internals
(event-queue and request-handler plumbing) via `@trace_class` decorators. On a single
Python memory-agent invocation this framework plumbing accounts for ~85% of the emitted
spans, burying the high-value `gen_ai.*` / `memory.*` / `db.memory.*` / `invoke_agent`
boundary spans.

kagent disables this SDK-internal instrumentation **by default** so agent traces stay
focused. The a2a SDK reads `OTEL_INSTRUMENTATION_A2A_SDK_ENABLED`
(`a2a/utils/telemetry.py`, default `true`) and turns its decorators into no-ops when the
value is `false`. The controller emits this env from the helm value
[`otel.tracing.a2aSdkInstrumentation`](../../helm/kagent/values.yaml) (default `false`)
and forwards it to agent pods alongside the other `OTEL_*` vars. Set it to `true` to
re-enable a2a SDK spans for deep protocol/queue debugging:

```yaml
otel:
  tracing:
    a2aSdkInstrumentation: true  # default false
```

The Go ADK is unaffected — it emits only deliberate high-level spans (no decorator
auto-instrumentation), so this setting is a2a-Python-SDK-specific.

## Trace signal-to-noise: httpx client + ASGI transport spans

After the a2a SDK spans are disabled (above), the largest remaining source of low-value
plumbing in a Python memory-agent trace is kagent's own auto-instrumentation of outbound
HTTP calls. The Python runtime instruments every httpx client request (Ollama LLM, Ollama
embedding, controller memory API) via `HTTPXClientInstrumentor` — roughly **65%** of the
post-a2a trace is bare client `POST`/`GET` spans. These are redundant with the curated
`gen_ai.*` / `memory.*` / `db.memory.*` spans, which already capture the same operations
with richer attributes and operation-level timing, and their count scales with
turns × tool-calls × embeddings.

kagent disables httpx client instrumentation **by default**. The runtime
(`kagent/core/tracing/_utils.py`) reads `OTEL_INSTRUMENTATION_HTTPX_CLIENT_ENABLED`
(default `false`) and only activates `HTTPXClientInstrumentor` when it is `true`. The
controller emits this env from the helm value
[`otel.tracing.httpxClientInstrumentation`](../../helm/kagent/values.yaml) (default
`false`) and forwards it to agent pods alongside the other `OTEL_*` vars. Set it to `true`
to re-enable raw outbound transport/latency spans for deep debugging:

```yaml
otel:
  tracing:
    httpxClientInstrumentation: true  # default false
```

Separately, the FastAPI **server boundary** span (the valuable request entrypoint) is
always kept, but the ASGI lifecycle sub-spans (`http send` / `http receive`) are dropped
unconditionally via `exclude_spans=["receive", "send"]` — they carry no diagnostic value.

Unlike the a2a toggle (helm-only, read by the third-party SDK), this is a change to
kagent's own Python runtime code, so it ships in the agent image rather than purely in
helm.

## Verifying live

See [`docs/verification/kagent-after.dql`](../verification/kagent-after.dql) for a
Dynatrace DQL query that lists memory spans and their governance attributes, and
[`docs/verification/kagent-baseline.dql`](../verification/kagent-baseline.dql) for the
pre-instrumentation baseline (zero `memory.*` spans).
