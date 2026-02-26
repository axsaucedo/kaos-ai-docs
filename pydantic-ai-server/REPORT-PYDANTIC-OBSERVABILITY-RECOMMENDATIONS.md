# Pydantic AI Observability — Recommendations for KAOS

## Current State (Post-Refactor)

### What KAOS Provides
- **SDK Initialization**: `init_otel()` sets up `TracerProvider`, `MeterProvider`, `LoggerProvider`, W3C propagators
- **Log Correlation**: `KaosLoggingHandler` adds `trace_id`/`span_id` to all Python log records
- **Delegation Spans**: `tracer.start_as_current_span("delegate.{name}")` for sub-agent calls
- **Server Span**: `tracer.start_as_current_span("chat_completions", context=parent_ctx, kind=SERVER)` for inbound requests
- **Context Propagation**: `inject_trace_context()` / `extract_trace_context()` for cross-service W3C traceparent
- **Delegation Metrics**: `kaos.delegations` counter + `kaos.delegation.duration` histogram

### What Pydantic AI Provides (via `instrument=True`)
- **Agent Run Span**: `agent run: {name}` — top-level span for the full agentic loop
- **Model Call Spans**: `chat {model}` — per-call spans with GenAI semantic conventions
- **Tool Batch Span**: `running tools` — groups parallel tool executions
- **Per-Tool Spans**: `execute tool: {name}` — individual tool execution timing
- **Span Attributes**: `gen_ai.input.messages`, `gen_ai.output.messages`, `pydantic_ai.all_messages` (version 2+)
- Uses OTEL `Logger` API (not Python `logging`) — data goes as span attributes, not log records

### Trace Tree
```
[W3C traceparent from caller]
  └── server: chat_completions (SERVER, KAOS)
      └── agent run: my-agent (INTERNAL, Pydantic AI)
          ├── chat openai:model (CLIENT, Pydantic AI)
          ├── running tools (INTERNAL, Pydantic AI)
          │   └── execute tool: echo (INTERNAL, Pydantic AI)
          ├── delegate.worker (INTERNAL, KAOS)
          │   └── HTTP POST → sub-agent (with traceparent injection)
          └── chat openai:model (CLIENT, Pydantic AI)
```

## Key Findings

### 1. Pydantic AI Does NOT Use Python Logging
Pydantic AI's observability module (`_output.py`) uses the OTEL `Logger` API directly:
```python
from opentelemetry._logs import Logger, LogRecord, SeverityNumber
```
For version 2+ (default `event_mode`), all data is stored as **span attributes** (e.g., `gen_ai.input.messages`), not as separate log events. This means:
- KAOS's `LoggingInstrumentor` cannot capture Pydantic AI internal logs
- There are no Pydantic AI log lines to correlate with traces
- This is by design — Pydantic AI follows GenAI semantic conventions where data belongs on spans

### 2. KAOS Logs ARE Properly Correlated
KAOS's own Python `logging` calls (via `KaosLoggingHandler`) correctly include `trace_id` and `span_id` in every log record when OTEL is enabled. This covers:
- Agent lifecycle events ("Processing message for session...")
- Memory events ("Added tool_call event to session...")
- Delegation events ("Delegation to worker-agent succeeded")
- Error logs with full exception context

### 3. No Redundant Spans Remain
After the refactor:
- KAOS removed `agent.agentic_loop` span (Pydantic AI's `agent run` covers it)
- KAOS removed model/tool metric counters (Pydantic AI spans provide this via attributes)
- KAOS only adds: server request span, delegation spans, delegation metrics

## Recommendations

### R1: Consider `event_mode='logs'` for Debug Environments
Pydantic AI supports `InstrumentationSettings(event_mode='logs')` which emits data as OTEL `LogRecord`s instead of span attributes. This could be useful for:
- Debug/dev environments where log-based tooling (Loki, CloudWatch) is primary
- Environments without full trace visualization (Jaeger, Tempo)

**Trade-off**: Version 1 event mode is deprecated and may be removed. Not recommended for production.

### R2: Upstream Contribution — Python Logging Bridge
Consider contributing to Pydantic AI a configurable option to emit key events via Python's `logging` module in addition to OTEL spans. This would enable:
- Log correlation without requiring full OTEL infrastructure
- Compatibility with existing log aggregation pipelines
- Easier debugging in development

**Complexity**: Medium. Would need to be opt-in to avoid performance overhead.

### R3: Add Custom Span Attributes for KAOS Context
Enrich Pydantic AI spans with KAOS-specific attributes:
```python
# In agent construction
Agent(
    instrument=InstrumentationSettings(
        additional_span_attributes={
            "kaos.session_id": session_id,
            "kaos.agent_name": agent_name,
        }
    )
)
```
This would allow filtering traces by KAOS session/agent in Jaeger/Tempo.

**Status**: Not yet supported by Pydantic AI. Would require upstream contribution to `InstrumentationSettings`.

### R4: Structured Delegation Metrics via OTEL Metrics API
Current delegation metrics (`kaos.delegations` counter, `kaos.delegation.duration` histogram) are sufficient. Consider adding:
- `kaos.delegation.errors` counter with error type attribute
- `kaos.delegation.retries` counter (if retry logic is added)

### R5: Health Check Span Filtering
Server health/readiness probes (`/health`, `/ready`) generate noisy spans. Consider:
- Adding span filter to exclude health check paths from tracing
- Or using sampling rules to reduce volume

### R6: Upstream Contribution — Per-Tool Span Attributes
Pydantic AI's per-tool spans (`execute tool: {name}`) include `gen_ai.tool.name` and `gen_ai.tool.call.id`. Consider contributing:
- Tool execution duration as a metric (currently only a span)
- Tool input/output as span events (for debugging)
- Error classification attributes for failed tool calls

## Limitations of Current Approach

1. **No Pydantic AI internal logs**: Cannot see model prompt assembly, tool argument parsing, or internal retry logic in logs. Only visible via span attributes in trace viewers.

2. **Span attribute size**: Version 2 event mode stores full message arrays as span attributes. For long conversations, this can produce very large spans. Consider implementing `max_attribute_length` in the OTEL SDK config.

3. **No streaming span support**: Pydantic AI does not currently create spans for streaming responses. The final model call span only appears after streaming completes. This is a known gap in the GenAI semantic conventions.

4. **Cross-language propagation**: KAOS uses W3C TraceContext (`traceparent`/`tracestate`). Ensure all HTTP clients (including model API calls via litellm) propagate these headers. The `RequestsInstrumentor` / `AioHTTPInstrumentor` handle this for instrumented clients.

## Files Reference
- `data-plane/kaos-framework/telemetry/manager.py`: OTEL SDK init + helper functions
- `data-plane/kaos-framework/agent/client.py`: Delegation spans + trace context injection
- `data-plane/kaos-framework/agent/server.py`: Server span + trace context extraction
- `PLAN-FULL-OTEL-REFACTOR.md`: Original 5-phase migration plan (phases 1-3 executed)
- `REPORT-PYDANTIC-TELEMETRY-FULL.md`: Deep-dive into Pydantic AI OTEL internals
