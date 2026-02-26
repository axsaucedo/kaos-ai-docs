# OpenTelemetry Integration Journey - KAOS Framework

This document presents a cohesive narrative of integrating OpenTelemetry into the KAOS (Kubernetes Agent Orchestration System) framework, transforming raw commits into a story of iterative development, lessons learned, and architectural evolution.

---

## Introduction

Integrating observability into agentic AI systems presents unique challenges. Unlike traditional microservices where request-response patterns are well understood, AI agents operate in iterative loops—calling models, executing tools, delegating to sub-agents—creating complex trace hierarchies that span multiple services and decision points.

This document chronicles the journey of adding comprehensive OpenTelemetry instrumentation to KAOS, a Kubernetes-native framework for orchestrating multi-agent AI systems. The implementation spans ~50 commits over 7 days, revealing patterns, pitfalls, and practical solutions for instrumenting agentic systems.

---

## Phase 1: Foundation - Core OTEL Module

**Timeline:** Day 1 (January 24)  
**Commits:** 0879a9a, 2d698f1, e1e442c, d966d7c, 79e78de, 91fc10a, 89aa37c

### The Initial Architecture

The journey began with establishing a foundational telemetry module:

```python
# Initial approach: Separate config, tracing, and metrics modules
python/agent/telemetry/
├── __init__.py
├── config.py      # TelemetryConfig with OTLP settings
├── tracing.py     # Span creation utilities
└── metrics.py     # Counters and histograms
```

**Key decisions made:**
1. **Semantic conventions** for agent-specific spans (agent.process_message, model.inference, tool.{name}, delegate.{name})
2. **Metrics design**: Counters for requests/tools/delegations, histograms for latencies
3. **Log correlation**: Inject trace_id/span_id into Python logs

### The Agentic Loop Challenge

The first interesting challenge: how to instrument an iterative loop where each iteration may have different operations?

```python
# The agentic loop pattern that needed instrumentation:
while not done:
    # 1. Call LLM model
    response = await model.chat(messages)
    
    # 2. If tool calls, execute them
    for tool_call in response.tool_calls:
        result = await execute_tool(tool_call)
        
    # 3. If delegation, call sub-agent
    if response.needs_delegation:
        result = await delegate_to_agent(...)
```

**Solution:** Create parent span for entire message processing, with child spans for each step:

```
agent.process_message (parent)
├── model.inference (LLM call)
├── tool.mcp_weather (tool execution)
├── tool.mcp_calendar (tool execution)
├── model.inference (LLM with tool results)
└── delegate.planner_agent (sub-agent delegation)
```

### Documentation First

Notably, documentation was added early (89aa37c) rather than as an afterthought. This established:
- Configuration examples for SigNoz, Uptrace, OTel Collector
- Environment variable reference
- Architecture diagrams showing trace flow

---

## Phase 2: Simplification - The KaosOtelManager Refactor

**Timeline:** Day 2 (January 25)  
**Commits:** f46e82c, 8932ef9, 39c69af, 21ac2e8, 9fd6c7d

### The Problem with Complexity

The initial implementation worked but was cumbersome:

```python
# Before: Too many settings, duplicate configuration
class AgentServerSettings(BaseSettings):
    otel_enabled: bool = False
    otel_endpoint: str = "http://localhost:4317"
    otel_service_name: str = ""
    otel_insecure: bool = True
    otel_log_correlation: bool = True
    # ... more settings
```

**Pain points discovered:**
1. Duplicate settings in AgentServerSettings and MCPServerSettings
2. Non-standard env vars conflicting with OTEL conventions
3. Initialization complexity

### The Refactored Solution

```python
# After: KaosOtelManager with simple API
class KaosOtelManager:
    """Lightweight wrapper for OpenTelemetry operations."""
    
    def __init__(self, service_name: str = None):
        # Use standard OTEL_* env vars
        # Single initialization point
        
    def span(self, name: str) -> Span:
        """Create a span within current context."""
        
    def model_span(self, model_name: str) -> Span:
        """Create span for model inference."""
        
    def tool_span(self, tool_name: str) -> Span:
        """Create span for tool execution."""
```

**Key architectural shift:** Use standard OpenTelemetry environment variables (OTEL_EXPORTER_OTLP_ENDPOINT, OTEL_SERVICE_NAME, OTEL_SDK_DISABLED) instead of custom ones. This:
- Reduces documentation burden
- Enables standard OTel tooling
- Simplifies operator configuration

### Module Reorganization

The telemetry module was moved from `python/agent/telemetry/` to `python/telemetry/` to make it a library-level concern rather than agent-specific.

---

## Phase 3: Operator Integration - CRD Extensions

**Timeline:** Days 2-3 (January 25-26)  
**Commits:** 06148a3, e1ef29b, 1865e89, 25efdbc

### The Control Plane Challenge

KAOS uses Kubernetes operators to manage AI workloads. The challenge: how should telemetry configuration flow from Helm → Operator → Data Plane components?

```yaml
# Goal: Single configuration point in Helm values.yaml
telemetry:
  enabled: true
  endpoint: "http://signoz-otel-collector.observability:4317"
```

### CRD Design

Added `TelemetryConfig` to the Agent and MCPServer CRDs:

```go
// api/v1alpha1/agent_types.go
type TelemetryConfig struct {
    // Enabled enables OpenTelemetry instrumentation
    Enabled *bool `json:"enabled,omitempty"`
    
    // Endpoint for OTLP exporter (e.g., "http://otel-collector:4317")
    Endpoint string `json:"endpoint,omitempty"`
}
```

### The Merge Challenge

A subtle bug discovered: what happens when both global (Helm) and per-resource telemetry configs exist?

```yaml
# Global in values.yaml
telemetry:
  enabled: true
  endpoint: "http://signoz:4317"

# Per-Agent override
spec:
  telemetry:
    enabled: false  # Should this disable telemetry for this agent?
```

**Solution (2eacb36):** Implement field-wise merge with validation:
- If per-resource config is empty, use global defaults
- If per-resource specifies fields, they override global
- Validate endpoint format

---

## Phase 4: Bug Fixes - Learning from Production

**Timeline:** Days 3-4 (January 26-27)  
**Commits:** ccc6ea2, 775ad49, f4922f7, 210dc69

### Critical Bug: Span Leakage

One of the most insidious bugs discovered was span leakage:

```python
# Bug: Span not properly closed on exception
def process_with_span():
    span = tracer.start_span("my_span")
    context = trace.set_span_in_context(span)
    token = context.attach(context)
    
    result = do_work()  # If this throws...
    
    span.end()  # ... this never executes
    context.detach(token)  # ... neither does this
```

**Symptoms in production:**
- Subsequent spans attached to wrong parent
- Trace trees with orphaned spans
- Memory leaks from accumulated context tokens

**Fix (775ad49):** Always use try/except/finally:

```python
def process_with_span():
    span = tracer.start_span("my_span")
    context = trace.set_span_in_context(span)
    token = context.attach(context)
    
    try:
        result = do_work()
        span.set_status(Status(StatusCode.OK))
        return result
    except Exception as e:
        span.set_status(Status(StatusCode.ERROR, str(e)))
        span.record_exception(e)
        raise
    finally:
        span.end()
        context.detach(token)
```

### LiteLLM Integration Quirks

ModelAPI supports LiteLLM as an LLM proxy. Discovered that LiteLLM's OTEL integration has specific requirements:

```python
# Wrong: LiteLLM doesn't understand generic 'otlp'
OTEL_EXPORTER = "otlp"

# Correct: LiteLLM requires explicit exporter type
OTEL_EXPORTER = "otlp_grpc"
```

This required adding OpenTelemetry packages to Dockerfile.litellm and understanding LiteLLM's callback-based OTEL implementation.

---

## Phase 5: Health Check Noise - Operational Excellence

**Timeline:** Day 5 (January 28)  
**Commits:** 1f02aa2, 47eee19, cba50bc

### The Problem

In Kubernetes, liveness and readiness probes hit endpoints every 10-30 seconds:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
```

This creates **massive trace noise**—thousands of /health spans per hour, burying actual request traces.

### The Solution Journey

**First attempt:**
```python
OTEL_PYTHON_FASTAPI_EXCLUDED_URLS = "^/health$,^/ready$"
```

**Problem discovered:** OpenTelemetry's URL exclusion uses `re.search()`, not `re.match()`. Anchors (^,$) are unnecessary and can cause issues.

**Final solution (cba50bc):**
```python
# Agent/MCPServer - FastAPI based
OTEL_PYTHON_FASTAPI_EXCLUDED_URLS = "/health,/ready"

# ModelAPI/LiteLLM - different env var
OTEL_PYTHON_EXCLUDED_URLS = "/health"
```

**Lesson learned:** Different OpenTelemetry instrumentations use different env vars. Always check the specific instrumentation library's documentation.

---

## Phase 6: Log Export & Correlation

**Timeline:** Days 5-6 (January 28-29)  
**Commits:** 22b0fd2, fb9dcef, b0fea19, 4cd4338, d785364, 3dde9b7

### The Three Pillars

OpenTelemetry's three pillars—traces, metrics, logs—need to work together. Initially, only traces were exported.

**Adding log export:**
```python
from opentelemetry.sdk._logs import LoggerProvider
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor
from opentelemetry.exporter.otlp.proto.grpc._log_exporter import OTLPLogExporter

log_provider = LoggerProvider(resource=resource)
log_provider.add_log_record_processor(
    BatchLogRecordProcessor(OTLPLogExporter(endpoint=endpoint))
)
```

### Log-Trace Correlation Challenge

A subtle ordering bug discovered:

```python
# Bug: Log emitted AFTER span context is detached
span_failure(span)  # Detaches context
logger.error(f"Tool execution failed: {e}")  # No trace context!
```

**Symptoms:** Logs in SigNoz appeared without trace_id, couldn't correlate logs to traces.

**Fix (60e7efb):** Emit logs BEFORE detaching context:
```python
logger.error(f"Tool execution failed: {e}")  # Has trace context
span_failure(span)  # Now safe to detach
```

### Unified Log Level Configuration

Challenge: Different components use different log level env vars:
- Python: `LOG_LEVEL` or `AGENT_LOG_LEVEL`
- LiteLLM: `LITELLM_LOG`
- Ollama: `OLLAMA_DEBUG`

**Solution:** Operator handles mapping:
```go
func (r *ModelAPIReconciler) buildLogLevelEnvVar(modelAPI *kaosv1alpha1.ModelAPI) corev1.EnvVar {
    level := util.GetDefaultLogLevel()
    
    if modelAPI.Spec.Mode == kaosv1alpha1.ModelAPIModeProxy {
        // LiteLLM uses different values
        litellmLevel := map[string]string{
            "TRACE": "DEBUG", "DEBUG": "DEBUG",
            "INFO": "INFO", "WARNING": "WARNING", "ERROR": "ERROR",
        }[level]
        return corev1.EnvVar{Name: "LITELLM_LOG", Value: litellmLevel}
    }
    // Ollama uses boolean for debug
    return corev1.EnvVar{Name: "OLLAMA_DEBUG", Value: level == "DEBUG" || level == "TRACE"}
}
```

---

## Phase 7: Trace Context Propagation

**Timeline:** Day 6 (January 29)  
**Commits:** c2e88f9, 8ff1a09, 3826f82, 1d52e86

### The Multi-Agent Challenge

KAOS supports multi-agent systems where Agent A delegates to Agent B. Each agent runs in a separate Pod with separate processes. How do traces connect?

```
[Agent A: Task Planner]
        |
        | HTTP POST /v1/chat/completions
        v
[Agent B: Code Writer]
        |
        | HTTP POST /v1/chat/completions
        v
[Agent C: Code Reviewer]
```

Without propagation, each agent creates independent traces—you can't see the full workflow.

### W3C Trace Context Propagation

**Solution:** Inject trace context in HTTP headers using W3C Trace Context standard:

```python
# Agent A: Inject context into outgoing request
from opentelemetry.propagate import inject

headers = {}
inject(headers)  # Adds traceparent, tracestate headers
response = await client.post(url, headers=headers, json=payload)
```

```python
# Agent B: Extract context from incoming request
from opentelemetry.propagate import extract

context = extract(request.headers)
with tracer.start_as_current_span("process", context=context):
    # Now this span is a child of Agent A's span
```

### The RemoteAgent Integration

Added context propagation to RemoteAgent class:

```python
class RemoteAgent:
    async def send_task(self, message: str) -> str:
        headers = {"Content-Type": "application/json"}
        
        # Inject trace context for distributed tracing
        from opentelemetry.propagate import inject
        inject(headers)
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.url}/v1/chat/completions",
                headers=headers,
                json={"messages": [{"role": "user", "content": message}]}
            )
```

### Memory Event Correlation

Agent memory events (storing conversation history) now include trace context:

```python
def create_event(self, event_type: str, data: dict) -> dict:
    event = {
        "id": str(uuid.uuid4()),
        "type": event_type,
        "data": data,
        "timestamp": datetime.utcnow().isoformat(),
        "metadata": {}
    }
    
    # Add trace context if available
    trace_context = get_current_trace_context()
    if trace_context:
        event["metadata"]["trace_id"] = trace_context["trace_id"]
        event["metadata"]["span_id"] = trace_context["span_id"]
    
    return event
```

This enables querying memory events by trace_id to understand what the agent "remembered" during a specific request.

---

## Phase 8: Singleton Patterns & Noise Reduction

**Timeline:** Days 6-7 (January 29-30)  
**Commits:** 65da1bc, a99d040, 31260be, 07167f4, 97271cc, b9fb19b, f9fccbe

### The Singleton Evolution

KaosOtelManager went through several iterations to achieve proper singleton behavior:

**Version 1: Class method**
```python
class KaosOtelManager:
    _instance = None
    
    @classmethod
    def get_instance(cls):
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance
```
Problem: Still allows `KaosOtelManager()` to create new instances.

**Version 2: Module-level singleton**
```python
_manager_instance = None

def get_otel_manager() -> KaosOtelManager:
    global _manager_instance
    if _manager_instance is None:
        _manager_instance = KaosOtelManager()
    return _manager_instance
```
Problem: Two patterns to maintain (class and module function).

**Version 3: __new__ pattern (final)**
```python
class KaosOtelManager:
    _instance = None
    _initialized = False
    
    def __new__(cls, service_name: str = None):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        elif service_name and service_name != cls._instance._service_name:
            logger.warning(
                f"KaosOtelManager already initialized with service_name="
                f"'{cls._instance._service_name}', ignoring '{service_name}'"
            )
        return cls._instance
```

### HTTPX Noise Reduction

MCP tools use Server-Sent Events (SSE) for streaming responses. HTTPX instrumentation creates spans for every SSE chunk—hundreds of spans for a single tool call!

**Solution:** Make HTTPX instrumentation opt-in:
```python
OTEL_INCLUDE_HTTP_CLIENT = "false"  # Default

if os.getenv("OTEL_INCLUDE_HTTP_CLIENT", "false").lower() == "true":
    HTTPXClientInstrumentor().instrument()
```

Also suppress noisy loggers:
```python
logging.getLogger("httpx").setLevel(logging.WARNING)
logging.getLogger("httpcore").setLevel(logging.WARNING)
logging.getLogger("mcp.client.streamable_http").setLevel(logging.WARNING)
logging.getLogger("docket.worker").setLevel(logging.WARNING)
logging.getLogger("fakeredis").setLevel(logging.WARNING)
```

---

## Key Learnings & Patterns

### 1. Standard Env Vars Over Custom

**Anti-pattern:**
```yaml
KAOS_OTEL_ENDPOINT: "http://collector:4317"
KAOS_OTEL_ENABLED: "true"
```

**Pattern:**
```yaml
OTEL_EXPORTER_OTLP_ENDPOINT: "http://collector:4317"
OTEL_SDK_DISABLED: "false"
```

Using standard OpenTelemetry environment variables enables:
- Automatic integration with OTel tooling
- Less documentation to maintain
- Familiar configuration for users

### 2. Context Management is Critical

Always use try/finally for span management:
```python
span = tracer.start_span("operation")
token = context.attach(trace.set_span_in_context(span))
try:
    return do_work()
finally:
    span.end()
    context.detach(token)
```

### 3. Log Before Span Close

For log-trace correlation, emit logs while span context is active:
```python
# Correct order
logger.error(f"Operation failed: {e}")  # Has trace context
span.set_status(Status(StatusCode.ERROR))
span.end()

# Wrong order  
span.end()
logger.error(f"Operation failed: {e}")  # No trace context!
```

### 4. Different Components, Different Env Vars

OpenTelemetry instrumentations aren't always consistent:
- FastAPI: `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS`
- Generic: `OTEL_PYTHON_EXCLUDED_URLS`
- LiteLLM: `OTEL_EXPORTER` requires "otlp_grpc" not "otlp"

### 5. Noise is the Enemy of Observability

Health check exclusion, HTTP client opt-in, and log level tuning are essential for useful traces. A trace with 10,000 /health spans is worse than no traces at all.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                        Helm Chart                                │
│  values.yaml: telemetry.enabled, telemetry.endpoint, logLevel   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Go Operator (Control Plane)                  │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐ │
│  │ Agent Controller │  │ MCPServer Ctrl   │  │ ModelAPI Ctrl  │ │
│  │                  │  │                  │  │                │ │
│  │ BuildTelemetry() │  │ BuildTelemetry() │  │ BuildTelemetry │ │
│  └──────────────────┘  └──────────────────┘  └────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Python Data Plane (Pods)                       │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               KaosOtelManager (Singleton)                 │   │
│  │                                                           │   │
│  │  • TracerProvider with OTLP exporter                      │   │
│  │  • MeterProvider with OTLP exporter                       │   │
│  │  • LoggerProvider with OTLP exporter                      │   │
│  │  • FastAPI/Starlette auto-instrumentation                 │   │
│  │  • Context propagation (W3C Trace Context)                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Agent → model.inference → tool.{name} → delegate.{name}        │
│  MCPServer → tool.execute                                        │
│  ModelAPI → LiteLLM callbacks / Ollama logs                      │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     OTEL Collector / SigNoz                      │
│                                                                  │
│  Traces ─── Jaeger UI / SigNoz Traces                            │
│  Metrics ── Prometheus / SigNoz Metrics                          │
│  Logs ───── SigNoz Logs (correlated by trace_id)                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Conclusion

Integrating OpenTelemetry into an agentic AI framework reveals challenges not present in traditional microservices:

1. **Iterative loops** require careful span hierarchy design
2. **Multi-agent delegation** needs explicit trace context propagation
3. **Tool execution** with streaming (SSE) creates noise without filtering
4. **Memory systems** benefit from trace context for debugging
5. **Heterogeneous runtimes** (Python agents, LiteLLM, Ollama) need unified configuration

The ~50 commits in this integration journey represent not just code changes, but discoveries about how observability patterns apply to a new paradigm of AI-native applications.
