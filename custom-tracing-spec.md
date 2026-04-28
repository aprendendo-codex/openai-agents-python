# Custom Tracing Spec (Agents SDK)

This specification explains how custom tracing works in the OpenAI Agents SDK, what extension points exist, and how to implement reliable custom trace processing.

## 1) Tracing model and terminology

The SDK emits telemetry as:

- **Trace**: A top-level container for one logical workflow/run (`workflow_name`, `trace_id`, optional `group_id`, metadata).
- **Span**: A timed unit of work within a trace (agent turn, LLM generation, tool call, guardrail, handoff, custom spans, etc.).
- **Processor**: A consumer that receives trace/span lifecycle callbacks and exports or transforms them.

By default, tracing is enabled and the SDK sends traces to OpenAI via a batched processor pipeline.

## 2) Default tracing pipeline

At startup, the SDK configures a global tracing provider and default processor stack:

1. A `TraceProvider` creates trace/spans.
2. A `BatchTraceProcessor` buffers and periodically flushes items.
3. A `BackendSpanExporter` exports batches to OpenAI tracing backend.

This means you get out-of-the-box tracing without additional setup when API credentials are configured.

## 3) Customization modes

You can customize tracing in two modes.

### A. Additive mode (recommended first)

Use `add_trace_processor(...)` to add your own processor **in addition** to defaults. In this mode:

- OpenAI export remains active.
- Your processor receives the same events for side-exports (e.g., Datadog, OTEL bridge, internal analytics).

### B. Replacement mode

Use `set_trace_processors([...])` to replace all default processors. In this mode:

- Nothing is exported to OpenAI unless you explicitly include a processor that does so.
- You fully own durability, retries, batching, and failure behavior.

## 4) Processor contract

Custom processors implement the tracing processor interface and handle lifecycle callbacks for traces/spans (start/end and shutdown/flush events).

Implementation requirements:

- Be **non-blocking/low-latency** on hot paths.
- Avoid raising uncaught exceptions in callbacks.
- Be thread-safe if your runtime can execute callbacks concurrently.
- Provide explicit `force_flush`/`shutdown` handling to avoid data loss on process exit.

## 5) Data flow and event timing

A typical run emits:

1. Trace started.
2. Nested spans started/finished as agent orchestration proceeds.
3. Trace finished after orchestration completes.
4. Batch processor exports asynchronously (or synchronously when `flush_traces()` is called).

Important behavior:

- `flush_traces()` should be called after the trace context exits when you need immediate delivery guarantees.
- In long-running workers, periodic/background flush is usually sufficient.

## 6) Creating custom spans

For business- or domain-level instrumentation, create spans using `custom_span(...)` under an active trace.

Guidelines:

- Keep span names stable and human-readable.
- Put dynamic values in span data/metadata (not in names).
- Record start/finish around meaningful work units.
- Attach errors when exceptions occur so failures are visible in trace UIs.

## 7) Configuration surfaces

Custom tracing behavior can be controlled at multiple levels:

- **Global SDK settings**
  - `set_trace_processors(...)`
  - `add_trace_processor(...)`
  - `set_tracing_disabled(...)`
  - `set_tracing_export_api_key(...)`
- **Per-run overrides (`RunConfig`)**
  - `tracing_disabled`
  - `tracing` (e.g., per-run tracing API key)
  - `workflow_name`, `trace_id`, `group_id`
  - `trace_include_sensitive_data`

Use global config for service-wide defaults and `RunConfig` for request/job-level overrides.

## 8) Sensitive data handling

Tracing may include model inputs/outputs and tool payloads depending on settings.

If you need stricter controls:

- Set `trace_include_sensitive_data=False` for runs handling sensitive content.
- Redact or hash values in your custom processor before forwarding to third-party systems.
- Avoid storing raw secrets in metadata fields.

## 9) Reliability and backpressure guidance

When building custom exporters/processors:

- Prefer batching and bounded queues.
- Define backpressure policy (drop-oldest, drop-newest, or block with timeout).
- Add retry with jittered exponential backoff for transient failures.
- Add structured logs/metrics for dropped events, queue depth, flush latency.
- Ensure shutdown hooks flush pending items.

## 10) Example patterns

### Pattern 1: Dual export (OpenAI + internal)

- Keep default processors.
- Add one custom processor that forwards normalized events to your analytics stream.
- Benefit: OpenAI traces dashboard remains available while you get internal observability.

### Pattern 2: Full custom backend

- Replace processors via `set_trace_processors`.
- Install your own batch processor + exporter.
- Benefit: complete control, useful in strict compliance or non-OpenAI observability stacks.

### Pattern 3: Per-tenant trace routing

- Configure global processor for standard behavior.
- Inject tenant metadata at trace start.
- Route in custom exporter by tenant/project IDs.

## 11) Minimal operational checklist

Before production rollout:

- Verify flush behavior on normal exit and crash/termination scenarios.
- Load-test callback overhead and queue saturation behavior.
- Validate redaction policy with synthetic sensitive payloads.
- Confirm replacement-mode deployments still export somewhere (to avoid silent observability loss).

## 12) Reference pointers

For implementation details and APIs, see:

- `docs/tracing.md`
- `src/agents/tracing/processor_interface.py`
- `src/agents/tracing/processors.py`
- `src/agents/tracing/setup.py`
- `src/agents/tracing/create.py`
