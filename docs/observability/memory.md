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

## Verifying live

See [`docs/verification/kagent-after.dql`](../verification/kagent-after.dql) for a
Dynatrace DQL query that lists memory spans and their governance attributes, and
[`docs/verification/kagent-baseline.dql`](../verification/kagent-baseline.dql) for the
pre-instrumentation baseline (zero `memory.*` spans).
