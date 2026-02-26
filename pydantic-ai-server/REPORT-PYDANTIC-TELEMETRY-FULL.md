# Deep Dive: Pydantic AI OpenTelemetry Instrumentation

## Introduction

This document provides a thorough deep-dive into how Pydantic AI implements OpenTelemetry (OTEL) instrumentation throughout its codebase. It is written for senior engineers who need to understand, extend, or contribute to the Pydantic AI telemetry module.

Pydantic AI's instrumentation follows the [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and provides automatic tracing for agent runs, model calls, tool execution, and output validation.

**Key files** (all under `pydantic_ai_slim/pydantic_ai/`):
- `agent/__init__.py` — Agent run spans, instrumentation setup
- `models/instrumented.py` — `InstrumentedModel` wrapper, model call spans, metrics
- `_tool_manager.py` — Per-tool execution spans
- `_output.py` — Output validation spans
- `_agent_graph.py` — Tool batch span ("running tools")
- `_instrumentation.py` — Span naming configuration
- `concurrency.py` — Concurrency wait spans

---

## 1. Enabling Instrumentation

### Per-Agent

```python
from pydantic_ai import Agent

agent = Agent("openai:gpt-4o", instrument=True)
```

When `instrument=True`, the model is wrapped in `InstrumentedModel` which adds OTEL spans to every model call.

**Source**: `agent/__init__.py`, `_get_model()` method:

```python
# agent/__init__.py:1466-1468
instrument = self.instrument
if instrument is None:
    instrument = self._instrument_default

return instrument_model(model_, instrument)
```

```python
# models/instrumented.py:66-73
def instrument_model(model: Model, instrument: InstrumentationSettings | bool) -> Model:
    if instrument and not isinstance(model, InstrumentedModel):
        if instrument is True:
            instrument = InstrumentationSettings()
        model = InstrumentedModel(model, instrument)
    return model
```

### Global (All Agents)

```python
from pydantic_ai import Agent

Agent.instrument_all(True)
```

Sets `Agent._instrument_default = True`, which applies to all agents where `instrument` is not explicitly set.

---

## 2. InstrumentationSettings

The central configuration class for all telemetry options.

**Source**: `models/instrumented.py:78-156`

```python
@dataclass(init=False)
class InstrumentationSettings:
    tracer: Tracer = field(repr=False)
    meter: Meter
    logger: Logger = field(repr=False)
    event_mode: Literal['attributes', 'logs'] = 'attributes'
    include_binary_content: bool = True
    include_content: bool = True
    version: Literal[1, 2, 3, 4] = DEFAULT_INSTRUMENTATION_VERSION  # Default: 2

    def __init__(
        self,
        *,
        tracer_provider: TracerProvider | None = None,
        meter_provider: MeterProvider | None = None,
        # ...
    ):
        tracer_provider = tracer_provider or get_tracer_provider()  # ← Uses global!
        meter_provider = meter_provider or get_meter_provider()      # ← Uses global!
        scope_name = 'pydantic-ai'
        self.tracer = tracer_provider.get_tracer(scope_name, __version__)
        self.meter = meter_provider.get_meter(scope_name, __version__)
```

**Critical**: When no `tracer_provider` is passed, it uses the **global OTEL provider**. This means if an external system (like KAOS) calls `trace.set_tracer_provider()`, Pydantic AI will automatically use that provider's exporter (OTLP, Jaeger, etc.).

### Version Configuration

The `version` parameter controls span naming and attribute conventions:

**Source**: `_instrumentation.py:14-97`

| Version | Agent Run Span | Tool Span | Attributes |
|---------|---------------|-----------|------------|
| 1-2 | `agent run` | `running tool` | `tool_arguments`, `tool_response` |
| 3-4 | `invoke_agent {name}` | `execute_tool {name}` | `gen_ai.tool.call.arguments`, `gen_ai.tool.call.result` |

Default is version 2.

---

## 3. Span Types and Their Sources

### 3.1 Agent Run Span

Created when `agent.run()`, `agent.iter()`, or `agent.run_stream()` is called.

**Source**: `agent/__init__.py:706-770`

```python
# agent/__init__.py:666-671 — Tracer selection
if isinstance(model_used, InstrumentedModel):
    instrumentation_settings = model_used.instrumentation_settings
    tracer = model_used.instrumentation_settings.tracer
else:
    instrumentation_settings = None
    tracer = NoOpTracer()  # ← No instrumentation!

# agent/__init__.py:708-717 — Span creation
run_span = tracer.start_span(
    instrumentation_names.get_agent_run_span_name(agent_name),
    attributes={
        'model_name': model_used.model_name if model_used else 'no-model',
        'agent_name': agent_name,
        'gen_ai.agent.name': agent_name,
        'logfire.msg': f'{agent_name} run',
    },
)
```

The span is **not** set as current via `start_as_current_span`. Instead, it's passed to the graph via `use_span()`:

```python
# agent/__init__.py:724-728
graph.iter(
    inputs=user_prompt_node,
    state=state,
    deps=graph_deps,
    span=use_span(run_span) if run_span.is_recording() else None,
)
```

The `pydantic_graph` library enters this span as the current span for the duration of the graph execution (`pydantic_graph/graph.py:247`):

```python
# pydantic_graph/graph.py:247
entered_span = stack.enter_context(span)  # Makes run_span the current span
```

**End attributes** (set on completion, `agent/__init__.py:808-870`):
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.total_tokens`
- `pydantic_ai.all_messages` (full conversation history as JSON)
- `gen_ai.system_instructions` (system prompt)
- `final_result` (agent output)
- `metadata` (run metadata)

### 3.2 Model Call Span (LLM Inference)

Created for every model API call (LLM inference request/response).

**Source**: `models/instrumented.py:440-530`

```python
# models/instrumented.py:448-449
operation = 'chat'
span_name = f'{operation} {self.model_name}'  # e.g., "chat openai:gpt-4o"

# models/instrumented.py:475-477
with self.instrumentation_settings.tracer.start_as_current_span(
    span_name, attributes=attributes, kind=SpanKind.CLIENT
) as span:
```

**Attributes** (conforming to GenAI semantic conventions):
- `gen_ai.operation.name`: `'chat'`
- `gen_ai.system`: Provider name (e.g., `'openai'`)
- `gen_ai.request.model`: Model name
- `gen_ai.tool.definitions`: JSON array of available tools
- `gen_ai.request.temperature`, `gen_ai.request.top_p`, etc. (from model settings)
- `gen_ai.response.model`: Actual model used in response
- `gen_ai.response.id`: Provider response ID
- `gen_ai.response.finish_reasons`: e.g., `['stop']`, `['tool_calls']`
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`
- `operation.cost`: Estimated cost in USD

**Metrics recorded** (`models/instrumented.py:340-354`):
- `gen_ai.client.token.usage` histogram (input/output tokens)
- `gen_ai.client.operation.cost` histogram (when cost is calculable)

### 3.3 Tool Batch Span ("running tools")

Created in `_agent_graph.py` when tools need to be executed after a model response.

**Source**: `_agent_graph.py:1097-1103`

```python
# _agent_graph.py:1097-1103
with tracer.start_as_current_span(
    'running tools',
    attributes={
        'tools': [call.tool_name for call in tool_calls],
        'logfire.msg': f'running {len(tool_calls)} tool{"" if len(tool_calls) == 1 else "s"}',
    },
):
```

This wraps ALL tool calls for a given model response. Individual tool calls happen inside this span.

### 3.4 Per-Tool Execution Span

Created for each individual tool call when the tool is actually executed.

**Source**: `_tool_manager.py:244-310`

```python
# _tool_manager.py:285-291
with tracer.start_as_current_span(
    instrumentation_names.get_tool_span_name(call.tool_name),
    # Version 2: "running tool", Version 3+: "execute_tool {name}"
    attributes=span_attributes,
) as span:
    tool_result = await self._call_tool(call, ...)
```

**Attributes**:
- `gen_ai.tool.name`: Tool name
- `gen_ai.tool.call.id`: Tool call ID
- `gen_ai.tool.call.arguments` (v3+) or `tool_arguments` (v2): Input arguments (if content inclusion enabled)
- `gen_ai.tool.call.result` (v3+) or `tool_response` (v2): Tool output (if content inclusion enabled)

**Error handling**: On `ModelRetry` (unknown tool, validation error), the span records the result attribute before re-raising.

### 3.5 Output Validation Span

Created when an output tool is called for structured output validation.

**Source**: `_output.py:132-155`

```python
# _output.py:132-134
with run_context.tracer.start_as_current_span(
    instrumentation_names.get_output_tool_span_name(tool_name), attributes=attributes
) as span:
    output = await function_schema.call(args, run_context)
```

### 3.6 Concurrency Wait Span

Created when an agent or model hits a concurrency limit and must wait.

**Source**: `concurrency.py:204-218`

```python
# concurrency.py:204-218
tracer = self._get_tracer()  # Falls back to get_tracer('pydantic-ai')
span_name = f'waiting for {display_name} concurrency'
with tracer.start_as_current_span(span_name, attributes=attributes):
    await self._limiter.acquire()
```

---

## 4. Span Hierarchy (Trace Tree)

When fully instrumented, a typical agent run produces:

```
invoke_agent my-agent (or "agent run")
├── chat openai:gpt-4o                    ← Model call (user prompt → tool calls)
│   ├── gen_ai.usage.input_tokens: 150
│   ├── gen_ai.usage.output_tokens: 50
│   └── gen_ai.response.finish_reasons: ["tool_calls"]
├── running tools                          ← Tool batch span
│   ├── execute_tool echo                  ← Per-tool span
│   │   ├── gen_ai.tool.name: "echo"
│   │   ├── gen_ai.tool.call.arguments: '{"message":"hello"}'
│   │   └── gen_ai.tool.call.result: '"hello"'
│   └── execute_tool fetch                 ← Per-tool span
│       └── ...
├── chat openai:gpt-4o                    ← Model call (tool results → final response)
│   ├── gen_ai.usage.input_tokens: 200
│   └── gen_ai.response.finish_reasons: ["stop"]
├── pydantic_ai.all_messages: [...]        ← Full conversation on run span
├── gen_ai.usage.total_tokens: 400         ← Aggregated usage on run span
└── final_result: "The echo returned..."   ← Agent output on run span
```

---

## 5. How Tracer Flows Through the System

```
Agent.__init__(instrument=True)
    │
    ▼
Agent._get_model() → instrument_model(model, True)
    │
    ▼
InstrumentedModel(model, InstrumentationSettings())
    │  InstrumentationSettings uses get_tracer_provider() → global OTEL provider
    │
    ▼
Agent._run_agent_graph()  # agent/__init__.py:614
    │
    ├── tracer = model_used.instrumentation_settings.tracer  (or NoOpTracer)
    ├── run_span = tracer.start_span("invoke_agent ...")
    ├── graph_deps.tracer = tracer  ← Tracer passed to graph deps
    │
    ▼
graph.iter(span=use_span(run_span))
    │  run_span becomes current span for all graph nodes
    │
    ├── ModelRequestNode → InstrumentedModel.request()
    │   └── tracer.start_as_current_span("chat openai:gpt-4o")  ← Child of run_span
    │
    ├── CallToolsNode → _call_tools(tracer=ctx.deps.tracer)
    │   └── tracer.start_as_current_span("running tools")  ← Child of run_span
    │       └── _call_tool() → tool_manager.handle_call()
    │           └── _call_function_tool(tracer=self.ctx.tracer)
    │               └── tracer.start_as_current_span("execute_tool echo")  ← Child of "running tools"
    │
    └── End → run_span.set_attributes({usage, messages, result})
```

---

## 6. Integration with KAOS

### Provider Binding

KAOS initializes OTEL and passes explicit providers to Pydantic AI:

```python
# KAOS telemetry/manager.py
init_otel(agent_name)  # Sets global TracerProvider, MeterProvider, LoggerProvider

# KAOS agent/server.py
from pydantic_ai.models.instrumented import InstrumentationSettings

settings = InstrumentationSettings(
    tracer_provider=get_tracer_provider(),
    meter_provider=get_meter_provider(),
    logger_provider=get_logger_provider(),
    version=int(os.environ.get("OTEL_INSTRUMENTATION_VERSION", "4")),
    event_mode=os.environ.get("OTEL_EVENT_MODE", "attributes"),
)
Agent.instrument_all(settings)
```

### Configuration

| Env Var | Default | Description |
|---------|---------|-------------|
| `OTEL_INSTRUMENTATION_VERSION` | `4` | Pydantic AI instrumentation version (1-4) |
| `OTEL_EVENT_MODE` | `attributes` | `attributes` (span attrs) or `logs` (OTEL log records, forces v1) |

### Resulting Trace Tree

```
KAOS: server-run (session.id=...)        ← KAOS span in chat_completions handler
└── Pydantic AI: invoke_agent my-agent   ← Automatic child (same tracer provider)
    ├── chat openai:gpt-4o
    ├── running tools
    │   └── execute_tool echo
    ├── chat openai:gpt-4o
    └── [aggregated usage + messages as span attributes]
```

For delegation, KAOS adds `delegate.{agent_name}` spans that parent the HTTP call to the sub-agent.
```

Context propagation for A2A delegation:

```
Agent A: invoke_agent supervisor
├── chat openai:gpt-4o → tool_calls: [delegate_to_worker]
├── running tools
│   └── execute_tool delegate_to_worker
│       └── KAOS: delegate.worker          ← KAOS delegation span
│           └── HTTP POST → Agent B        ← traceparent header injected
│               └── Agent B: invoke_agent worker  ← Same trace ID
│                   ├── chat openai:gpt-4o
│                   └── [result]
└── chat openai:gpt-4o → final response
```

---

## 7. Upstream Contribution: Per-Tool Spans in _agent_graph.py

### Background

While Pydantic AI already has per-tool spans in `_tool_manager.py`, there are edge cases where tool errors happen at the `_agent_graph.py` level (before `_tool_manager` is called). Our fork at https://github.com/axsaucedo/pydantic-ai/tree/feat/per-tool-otel-spans adds per-tool spans at the `_call_tool()` level in `_agent_graph.py`.

### Changes Made

**File**: `pydantic_ai_slim/pydantic_ai/_agent_graph.py`

1. **Added `nullcontext` import**:
```python
from contextlib import asynccontextmanager, contextmanager, nullcontext
```

2. **Added `tracer` parameter to `_call_tool()`**:
```python
async def _call_tool(
    tool_manager: ToolManager[DepsT],
    tool_call: _messages.ToolCallPart,
    tool_call_result: DeferredToolResult | None,
    tool_call_metadata: dict[str, dict[str, Any]] | None,
    tracer: Tracer | None = None,  # NEW
) -> tuple[...]:
    span_attrs = {
        'gen_ai.tool.name': tool_call.tool_name,
        'gen_ai.tool.call_id': tool_call.tool_call_id or '',
    }
    cm = tracer.start_as_current_span(
        f'tool: {tool_call.tool_name}', attributes=span_attrs
    ) if tracer else nullcontext()
    with cm as span:
        # ... existing logic, now indented ...
```

3. **Error/success attributes**:
```python
except ToolRetryError as e:
    if span is not None:
        span.set_attribute('gen_ai.tool.status', 'retry')
        span.set_attribute('gen_ai.tool.error', str(e.tool_retry.content))
    return e.tool_retry, None

# On success:
if span is not None:
    span.set_attribute('gen_ai.tool.status', 'success')
```

4. **Pass `tracer` from `_call_tools()`** (3 call sites updated):
```python
_call_tool(tool_manager, call, tool_call_results.get(call.tool_call_id), tool_call_metadata, tracer)
```

### Relationship to Existing Spans

This adds a span at the `_agent_graph` level that wraps the entire tool call lifecycle (including deferred results, approval, and tool_manager dispatch). The existing `_tool_manager.py` per-tool span is nested inside this one:

```
_agent_graph: tool: echo          ← NEW (our contribution)
└── _tool_manager: execute_tool echo  ← EXISTING
```

---

## 8. Configuration Reference

### Environment Variables (Standard OTEL)

| Variable | Description |
|----------|-------------|
| `OTEL_SDK_DISABLED` | `true` to disable SDK entirely |
| `OTEL_SERVICE_NAME` | Service name for resource |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP collector endpoint |
| `OTEL_EXPORTER_OTLP_HEADERS` | Authentication headers |

### Pydantic AI Configuration

| Parameter | Location | Description |
|-----------|----------|-------------|
| `instrument=True` | `Agent(instrument=True)` | Enable per-agent |
| `Agent.instrument_all(True)` | Static method | Enable for all agents |
| `InstrumentationSettings(version=4)` | `Agent(instrument=settings)` | Use v4 GenAI naming |
| `include_content=False` | `InstrumentationSettings(...)` | Exclude prompts/responses from spans |
| `tracer_provider=custom` | `InstrumentationSettings(...)` | Override global provider |

### Instrumentation Versions

| Version | Agent Span | Tool Span | Tool Args Attr | Notes |
|---------|-----------|-----------|----------------|-------|
| 1 | `agent run` | `running tool` | `tool_arguments` | Legacy, event-based |
| 2 (default) | `agent run` | `running tool` | `tool_arguments` | Newer message format |
| 3 | `invoke_agent {name}` | `execute_tool {name}` | `gen_ai.tool.call.arguments` | GenAI conventions |
| 4 | `invoke_agent {name}` | `execute_tool {name}` | `gen_ai.tool.call.arguments` | + multimodal |

---

## 9. Testing Instrumentation

### In-Memory Span Exporter (Unit Tests)

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export.in_memory import InMemorySpanExporter

exporter = InMemorySpanExporter()
provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# Run agent...
spans = exporter.get_finished_spans()
assert any(s.name == "invoke_agent my-agent" for s in spans)
assert any(s.name == "execute_tool echo" for s in spans)
```

### Verifying Span Hierarchy

```python
agent_span = next(s for s in spans if "invoke_agent" in s.name)
tool_span = next(s for s in spans if "execute_tool" in s.name)
assert tool_span.parent.span_id == agent_span.context.span_id
```

---

## 10. Dependencies

Pydantic AI's instrumentation requires:
- `opentelemetry-api` — Tracer, Span, SpanKind
- `opentelemetry-sdk` — TracerProvider (for actual tracing, not just NoOp)
- No Logfire dependency required — Logfire is optional and only used for enhanced formatting

The `pydantic-ai-slim` package includes OTEL API as a dependency. The SDK must be installed separately by the user (or by a framework like KAOS).

---

## 11. Potential Upstream Issues

### 11.1 `instrument=True` Ignores `instrument_all()` Settings

**Problem**: When `instrument_all(InstrumentationSettings(...))` is called to set class-level defaults, agents created with `instrument=True` (boolean) ignore these settings and create fresh `InstrumentationSettings()` with defaults (version=2, event_mode='attributes').

The Pydantic AI code (`agent/__init__.py:1463-1467`):
```python
instrument = self.instrument      # True (explicit boolean)
if instrument is None:            # False — skipped
    instrument = self._instrument_default  # Custom settings never used
return instrument_model(model_, instrument)
```

`instrument_model()` treats `True` as "create new defaults":
```python
def instrument_model(model, instrument):
    if instrument is True:
        instrument = InstrumentationSettings()  # Fresh defaults, not instrument_all() settings
    model = InstrumentedModel(model, instrument)
```

**Expected behavior**: `instrument=True` should be interpreted as "enable instrumentation using the class-level default settings from `instrument_all()`", not "enable with fresh defaults".

**Workaround**: Don't pass `instrument=True`; leave it as `None` so `_instrument_default` is used.

**Example**:
```python
from pydantic_ai import Agent
from pydantic_ai.models.instrumented import InstrumentationSettings

# Configure version=1 + event_mode='logs' for all agents
Agent.instrument_all(InstrumentationSettings(version=1, event_mode='logs'))

# BUG: This agent uses version=2 + event_mode='attributes' (ignores instrument_all)
agent = Agent('openai:gpt-4o', instrument=True)

# WORKAROUND: Omit instrument= to use instrument_all() settings
agent = Agent('openai:gpt-4o')  # Uses version=1 + event_mode='logs'
```

### 11.2 OTEL Log Events Only Available in Deprecated Version 1

**Problem**: `event_mode='logs'` (which emits OTEL `LogRecord` events via `logger.emit()`) only works with `version=1`, which is deprecated and will be removed. Versions 2-4 store all data exclusively as span attributes — there is no way to emit log records in modern versions.

The source code (`models/instrumented.py:263-309`):
```python
if self.version == 1:
    # ... builds events, calls self._emit_events(span, events)
    # _emit_events calls self.logger.emit(event) when event_mode='logs'
else:
    # version 2+: only span.set_attributes()
    # No logger.emit() call at all
```

Setting `event_mode='logs'` in any version forces `version=1` with a deprecation warning:
```python
if event_mode == 'logs':
    warnings.warn("event_mode is only relevant for version=1 which is deprecated...")
    version = 1
```

**Impact**: Frameworks that need OTEL log records correlated with traces (for log-based monitoring, alerting, or correlation in backends like SigNoz/Grafana) have no path forward when version 1 is removed.

**Proposed enhancement**: Add `event_mode` support to version 4+ (or a future version 5) that emits `LogRecord` events alongside span attributes. This enables:
- Log-based dashboards showing gen_ai events as correlated log records
- Unified log + trace correlation in OTEL backends
- Backwards-compatible (default `event_mode='attributes'` preserves current behavior)

**Example of desired behavior**:
```python
settings = InstrumentationSettings(
    version=4,
    event_mode='logs',  # Should work without deprecation warning
)
# → Emits both span attributes (v4 format) AND LogRecord events
```
