# From Chaos to Clarity: Instrumenting Agentic AI Systems with OpenTelemetry

*A practical guide to adding production-grade observability to multi-agent AI systems*

---

You've built an AI agent that works beautifully in development. It chains tools together, delegates tasks to specialist sub-agents, and produces impressive results. Then you deploy it to production.

A user reports that a request "took forever." Another says they got a strange response. Your logs show the agent ran—but what happened inside those 45 seconds between request and response?

Welcome to the observability challenge of agentic systems.

In this article, I'll walk through a complete, real-world implementation of OpenTelemetry instrumentation for an agentic AI framework. We'll cover the unique challenges these systems present, the core concepts you need to understand, and the practical lessons learned from ~50 commits of development. By the end, you'll have a clear blueprint for making your own agentic systems observable.

---

## Why Agentic Systems Need Different Observability

### It's Not Just a Request-Response

Traditional microservices have predictable patterns: a request comes in, some processing happens, a response goes out. Latency is relatively consistent, code paths are deterministic, and debugging usually involves tracing a single thread of execution.

Agentic systems break all of these assumptions:

| Traditional API | Agentic System |
|-----------------|----------------|
| Synchronous request-response | Iterative reasoning loops |
| Predictable latency (50-500ms) | Variable: 100ms to 60+ seconds |
| Deterministic code paths | Non-deterministic LLM decisions |
| Single service per request | Model calls + tool calls + delegations |
| Fixed cost per request | Cost varies by token usage |

Consider the core loop of an AI agent:

```python
async def process_message(self, messages):
    for step in range(self.max_steps):
        # 1. Call the LLM
        response = await self.model.chat(messages)
        
        # 2. If the model wants to use a tool, execute it
        if response.has_tool_call:
            result = await self.execute_tool(response.tool_call)
            messages.append({"role": "tool", "content": result})
            continue
        
        # 3. If the model wants to delegate, call another agent
        if response.needs_delegation:
            result = await self.delegate_to_agent(response.delegation)
            messages.append({"role": "assistant", "content": result})
            continue
        
        # 4. Otherwise, we have our final answer
        return response.content
```

Each iteration of this loop may take a different path. The model might need one tool call or five. It might delegate to one sub-agent or chain through three. Traditional logging—"request started," "request completed"—tells you almost nothing about what actually happened.

### The Three Pillars of Agent Observability

OpenTelemetry provides three types of telemetry data, each serving a distinct purpose for agentic systems:

**Traces** answer: "What path did this request take through my agents?"

```
HTTP POST /v1/chat/completions (15.2s total)
└── agent.agentic_loop
    ├── agent.step.1 (3.1s)
    │   └── model.inference (3.0s)
    ├── agent.step.2 (8.5s)
    │   ├── model.inference (2.1s)
    │   └── tool.web_search (6.3s)   ← Here's your bottleneck
    └── agent.step.3 (3.4s)
        └── model.inference (3.3s)
```

**Metrics** answer: "How is my system performing overall?"

- How many tokens am I using per request?
- What's my tool success rate?
- What's my P99 latency for model calls?

**Logs** answer: "What did the agent 'think' at each step?"

```
2024-01-15 10:30:45 INFO [trace_id=abc123] Starting message processing
2024-01-15 10:30:47 DEBUG [trace_id=abc123] Model response: calling tool 'web_search'
2024-01-15 10:30:53 ERROR [trace_id=abc123] Tool execution failed: API rate limited
```

The magic happens when these three are correlated. Click on that ERROR log in your observability backend, and it takes you to the exact span in the trace where the failure occurred.

### Why OpenTelemetry?

Before diving into implementation, it's worth understanding why OpenTelemetry has become the industry standard:

1. **Vendor-neutral**: Works with SigNoz, Jaeger, Datadog, Honeycomb—your choice
2. **Comprehensive**: One SDK for traces, metrics, and logs
3. **Battle-tested**: CNCF project, second only to Kubernetes in activity
4. **Future-proof**: AI-specific semantic conventions are actively being developed

No vendor lock-in means you can switch backends without changing your instrumentation code.

---

## The Real-World Example: KAOS Framework

To make this practical, I'll use a real implementation as our example: KAOS (Kubernetes Agent Orchestration System), an open-source framework for deploying AI agents on Kubernetes.

KAOS provides:
- **Agents as Kubernetes resources** (CRDs)
- **Multi-agent delegation** with automatic service discovery
- **Tool integration** via the Model Context Protocol (MCP)
- **OpenAI-compatible APIs** for all agents

The architecture separates concerns cleanly:

```
┌─────────────────────────────────────────────────────────────┐
│                   Control Plane (Go)                         │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   Agent     │  │  MCPServer   │  │   ModelAPI   │        │
│  │ Controller  │  │  Controller  │  │  Controller  │        │
│  └─────────────┘  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Plane (Python)                       │
│  ┌──────────────────┐ ┌──────────────────┐ ┌─────────────┐  │
│  │ Agent Runtime    │ │ MCPServer Runtime│ │ LiteLLM/    │  │
│  │ (FastAPI)        │ │ (MCP Protocol)   │ │ Ollama      │  │
│  └──────────────────┘ └──────────────────┘ └─────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

Adding OpenTelemetry to this system involved ~50 commits over a week. The patterns discovered apply to any agentic system, not just KAOS.

---

## Core Implementation: The Telemetry Manager

The foundation of our implementation is a singleton manager that handles all OpenTelemetry operations. Here's the key design:

```python
class KaosOtelManager:
    """OpenTelemetry manager using standard OTEL_* environment variables."""
    
    _instance = None
    _initialized = False
    
    def __new__(cls, service_name: str = None):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def should_enable(self) -> bool:
        """Check if telemetry should be enabled."""
        # Use standard OTEL env var (inverse logic: disabled=false means enabled)
        return os.getenv("OTEL_SDK_DISABLED", "true").lower() == "false"
    
    def init(self, service_name: str = None):
        """Initialize OpenTelemetry SDK (idempotent)."""
        if self._initialized or not self.should_enable():
            return
        
        # Build resource (service identity)
        resource = Resource.create({
            SERVICE_NAME: service_name or os.getenv("OTEL_SERVICE_NAME", "unknown"),
        })
        
        endpoint = os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
        
        # TracerProvider for distributed tracing
        tracer_provider = TracerProvider(resource=resource)
        tracer_provider.add_span_processor(
            BatchSpanProcessor(OTLPSpanExporter(endpoint=endpoint))
        )
        trace.set_tracer_provider(tracer_provider)
        
        # MeterProvider for metrics
        meter_provider = MeterProvider(
            resource=resource,
            metric_readers=[PeriodicExportingMetricReader(
                OTLPMetricExporter(endpoint=endpoint)
            )]
        )
        metrics.set_meter_provider(meter_provider)
        
        # LoggerProvider for OTLP log export
        logger_provider = LoggerProvider(resource=resource)
        logger_provider.add_log_record_processor(
            BatchLogRecordProcessor(OTLPLogExporter(endpoint=endpoint))
        )
        
        # Attach handler to Python root logger
        handler = LoggingHandler(
            level=logging.INFO,
            logger_provider=logger_provider
        )
        logging.getLogger().addHandler(handler)
        
        self._initialized = True
```

Key design decisions:

1. **Singleton pattern via `__new__`**: Ensures only one instance exists, even if constructors are called multiple times.

2. **Standard environment variables**: Using `OTEL_SDK_DISABLED` and `OTEL_EXPORTER_OTLP_ENDPOINT` instead of custom names. This works with standard OTEL tooling out of the box.

3. **All three providers**: TracerProvider for traces, MeterProvider for metrics, LoggerProvider for OTLP log export.

4. **Idempotent initialization**: Safe to call `init()` multiple times—subsequent calls are no-ops.

---

## Instrumenting the Agentic Loop

The core of agent observability is instrumenting the reasoning loop. Here's how we approach it:

```python
async def process_message(self, session_id: str, messages: List[Dict]) -> str:
    """Process message through agentic loop with full tracing."""
    
    # Start root span for entire message processing
    span = otel.span_begin("agent.agentic_loop", SpanKind.INTERNAL)
    span.set_attribute("agent.name", self.name)
    span.set_attribute("session.id", session_id)
    span.set_attribute("agent.max_steps", self.max_steps)
    
    try:
        for step in range(self.max_steps):
            # Span for each iteration
            step_span = otel.span_begin(f"agent.step.{step + 1}")
            step_span.set_attribute("step", step + 1)
            
            try:
                # Model inference with its own span
                model_span = otel.span_begin("model.inference", SpanKind.CLIENT)
                model_span.set_attribute("gen_ai.request.model", self.model_name)
                
                try:
                    response = await self.model_api.chat(messages)
                    model_span.set_attribute("gen_ai.response.finish_reason", 
                                             response.finish_reason)
                    otel.span_success(model_span)
                except Exception as e:
                    logger.error(f"Model call failed: {e}")  # Log BEFORE span close
                    otel.span_failure(model_span, e)
                    raise
                
                # Tool execution
                tool_call = self._extract_tool_call(response)
                if tool_call:
                    tool_span = otel.span_begin(f"tool.{tool_call.name}", SpanKind.CLIENT)
                    tool_span.set_attribute("tool.name", tool_call.name)
                    
                    try:
                        result = await self._execute_tool(tool_call)
                        messages.append({"role": "tool", "content": result})
                        logger.debug(f"Tool {tool_call.name} returned: {result[:100]}...")
                        otel.span_success(tool_span)
                    except Exception as e:
                        logger.error(f"Tool {tool_call.name} failed: {e}")
                        otel.span_failure(tool_span, e)
                        raise
                    
                    otel.span_success(step_span)
                    continue
                
                # Delegation to sub-agent
                delegation = self._extract_delegation(response)
                if delegation:
                    del_span = otel.span_begin(f"delegate.{delegation.agent}", 
                                                SpanKind.CLIENT)
                    del_span.set_attribute("agent.delegation.target", delegation.agent)
                    
                    try:
                        result = await self._delegate(delegation)
                        messages.append({"role": "assistant", "content": result})
                        otel.span_success(del_span)
                    except Exception as e:
                        logger.error(f"Delegation to {delegation.agent} failed: {e}")
                        otel.span_failure(del_span, e)
                        raise
                    
                    otel.span_success(step_span)
                    continue
                
                # Final answer
                otel.span_success(step_span)
                otel.span_success(span)
                return response.content
                
            except Exception as e:
                otel.span_failure(step_span, e)
                raise
        
        # Max steps reached
        otel.span_success(span)
        return response.content
        
    except Exception as e:
        otel.span_failure(span, e)
        raise
```

This produces a trace hierarchy that maps directly to what the agent did:

```
agent.agentic_loop (session_id=abc123, agent.name=coordinator)
├── agent.step.1
│   └── model.inference (gen_ai.request.model=gpt-4)
├── agent.step.2
│   ├── model.inference
│   └── tool.web_search (tool.name=web_search)
├── agent.step.3
│   ├── model.inference
│   └── delegate.researcher (agent.delegation.target=researcher)
└── agent.step.4
    └── model.inference
```

---

## Multi-Agent Context Propagation

The real challenge comes with multi-agent systems. When Agent A delegates to Agent B, which delegates to Agent C, you want a single unified trace—not three disconnected ones.

This requires **context propagation**: passing trace context through HTTP headers.

### Injecting Context (Outgoing Requests)

```python
class RemoteAgent:
    """Client for calling other agents via A2A protocol."""
    
    async def send_task(self, message: str) -> str:
        """Send task to remote agent with trace propagation."""
        
        headers = {"Content-Type": "application/json"}
        
        # This is the magic: inject current trace context into headers
        from opentelemetry.propagate import inject
        inject(headers)  # Adds traceparent, tracestate headers
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.url}/v1/chat/completions",
                headers=headers,  # Trace context included
                json={
                    "model": self.model,
                    "messages": [{"role": "user", "content": message}]
                }
            )
        
        return response.json()["choices"][0]["message"]["content"]
```

### Extracting Context (Incoming Requests)

```python
@app.post("/v1/chat/completions")
async def chat_completions(request: Request):
    """Handle chat request with trace context extraction."""
    
    # Extract trace context from incoming headers
    from opentelemetry.propagate import extract
    context = extract(request.headers)
    
    # Attach to current context
    token = otel_context.attach(context)
    
    try:
        # All spans created here are now children of the caller's span
        result = await agent.process_message(
            session_id=body.get("session_id", str(uuid.uuid4())),
            messages=body["messages"]
        )
        return result
    finally:
        otel_context.detach(token)
```

The result: a unified trace across all agents involved in a request.

```
coordinator.agent.agentic_loop (trace_id: abc123)
├── coordinator.model.inference
├── coordinator.delegate.researcher
│   └── researcher.agent.agentic_loop (same trace_id: abc123!)
│       ├── researcher.model.inference
│       └── researcher.tool.web_search
├── coordinator.model.inference
└── coordinator.delegate.analyst
    └── analyst.agent.agentic_loop (same trace_id: abc123!)
        ├── analyst.model.inference
        └── analyst.tool.calculator
```

---

## Kubernetes Integration: Configuration via CRDs

For KAOS, we wanted telemetry to be as easy to enable as adding a few lines to a YAML manifest:

```yaml
apiVersion: kaos.tools/v1alpha1
kind: Agent
metadata:
  name: my-agent
spec:
  modelAPI: openai-proxy
  model: "openai/gpt-4"
  config:
    description: "My production agent"
    telemetry:
      enabled: true
      endpoint: "http://otel-collector.observability:4317"
```

The Go operator translates this into environment variables for the Python runtime:

```go
func (r *AgentReconciler) buildTelemetryEnvVars(agent *Agent) []corev1.EnvVar {
    cfg := r.getTelemetryConfig(agent)
    if !cfg.Enabled {
        return nil
    }
    
    return []corev1.EnvVar{
        {Name: "OTEL_SDK_DISABLED", Value: "false"},
        {Name: "OTEL_EXPORTER_OTLP_ENDPOINT", Value: cfg.Endpoint},
        {Name: "OTEL_SERVICE_NAME", Value: agent.Name},
        {Name: "OTEL_RESOURCE_ATTRIBUTES", 
         Value: fmt.Sprintf("service.namespace=%s,kaos.resource.name=%s", 
                           agent.Namespace, agent.Name)},
        {Name: "OTEL_PYTHON_FASTAPI_EXCLUDED_URLS", Value: "/health,/ready"},
    }
}
```

Global defaults can be set in Helm values:

```yaml
# values.yaml
telemetry:
  enabled: true
  endpoint: "http://signoz-otel-collector.observability:4317"

logLevel: DEBUG  # TRACE, DEBUG, INFO, WARNING, ERROR
```

Every Agent and MCPServer in the cluster automatically inherits these settings unless overridden.

---

## Lessons Learned: The Hard-Won Insights

Through ~50 commits of development, we encountered (and solved) several non-obvious issues. Here are the key lessons:

### Lesson 1: Use Standard Environment Variables

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

Using standard OpenTelemetry environment variables means:
- Works with standard OTEL tooling (auto-instrumentation, collector config)
- Less documentation to maintain
- Familiar to anyone who's used OTEL before

### Lesson 2: Span Context Management is Critical

One of the most insidious bugs we found was span leakage:

```python
# Bug: If do_work() throws, context is never detached
span = tracer.start_span("operation")
token = context.attach(trace.set_span_in_context(span))
result = do_work()  # Exception here = context leak!
span.end()
context.detach(token)  # Never reached
```

Symptoms: subsequent spans attach to wrong parents, orphaned trace trees, memory leaks.

**Fix: Always use try/finally**

```python
span = tracer.start_span("operation")
token = context.attach(trace.set_span_in_context(span))
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

### Lesson 3: Log Before Span Close for Correlation

A subtle but important ordering issue:

```python
# Wrong: Log emitted after context is detached
span_failure(span, exception)  # Detaches context
logger.error(f"Operation failed: {exception}")  # No trace_id in log!
```

```python
# Right: Log while context is still active
logger.error(f"Operation failed: {exception}")  # Has trace_id
span_failure(span, exception)  # Safe to detach now
```

Logs emitted after span context is detached don't include `trace_id` or `span_id`, breaking log-trace correlation.

### Lesson 4: Health Check Noise is Real

Kubernetes probes hit `/health` and `/ready` every 10-30 seconds. Without exclusions, this creates thousands of traces that drown out actual requests.

```python
# Exclude health endpoints from tracing
os.environ["OTEL_PYTHON_FASTAPI_EXCLUDED_URLS"] = "/health,/ready"
```

Note: Different OTEL instrumentations use different env vars:
- FastAPI: `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS`
- Generic: `OTEL_PYTHON_EXCLUDED_URLS`
- LiteLLM: Uses generic exclusion

### Lesson 5: Make HTTP Client Tracing Opt-In

MCP (Model Context Protocol) uses Server-Sent Events for tool communication. This means every tool call generates dozens of HTTP spans for SSE chunks—massive noise.

**Solution: Disable HTTPX instrumentation by default**

```python
# Only instrument HTTP client if explicitly enabled
if os.getenv("OTEL_INCLUDE_HTTP_CLIENT", "false").lower() == "true":
    HTTPXClientInstrumentor().instrument()
else:
    # Suppress noisy HTTP client loggers
    logging.getLogger("httpx").setLevel(logging.WARNING)
    logging.getLogger("httpcore").setLevel(logging.WARNING)
```

Trace context is still propagated manually via `inject(headers)`, so multi-agent tracing works without the noise.

---

## Seeing It in Action

### Setting Up SigNoz

[SigNoz](https://signoz.io/) is an excellent open-source option for visualizing OpenTelemetry data:

```bash
# Deploy SigNoz
helm repo add signoz https://charts.signoz.io
helm install signoz signoz/signoz -n observability --create-namespace

# Configure agents to send telemetry
# In values.yaml:
telemetry:
  enabled: true
  endpoint: "http://signoz-otel-collector.observability:4317"
```

### What You'll See

**Service Map**: Visual representation of agent dependencies and communication patterns.

**Trace Waterfall**: The complete journey of a request through multiple agents and tools, with timing for each step.

**Log Correlation**: Click any span to see logs emitted during that operation, complete with trace context.

**Metrics Dashboard**: Token usage, latency percentiles, error rates—all the operational metrics you need.

### Debugging a Real Issue

Let's walk through a typical debugging scenario:

1. **User reports**: "My request took 45 seconds"
2. **Find the trace**: Search by session ID or timestamp
3. **Identify bottleneck**: Trace waterfall shows `tool.web_search` took 42 seconds
4. **Drill into logs**: Click the tool span, see logs: "API rate limited, retrying..."
5. **Root cause**: External API throttling causing exponential backoff

Without distributed tracing, this would have required correlating logs across multiple pods, guessing at timing, and hoping the right information was logged.

---

## The Bigger Picture: Where AI Observability is Heading

### Emerging Standards

OpenTelemetry's GenAI working group is developing semantic conventions specifically for AI workloads:

```python
# Emerging standard attributes for LLM calls
span.set_attribute("gen_ai.system", "openai")
span.set_attribute("gen_ai.request.model", "gpt-4")
span.set_attribute("gen_ai.request.max_tokens", 4096)
span.set_attribute("gen_ai.usage.input_tokens", 1523)
span.set_attribute("gen_ai.usage.output_tokens", 847)
span.set_attribute("gen_ai.response.finish_reasons", ["stop"])
```

These conventions will enable consistent tooling across different agent frameworks.

### Beyond Observability

The patterns we've established enable more than debugging:

**Cost Tracking**: Token counts per span → cost attribution per user, per feature, per agent.

**Quality Metrics**: Add evaluation scores as span attributes → track response quality over time.

**Prompt Versioning**: Include prompt template IDs in spans → correlate performance with prompt changes.

---

## Conclusion

Instrumenting agentic AI systems with OpenTelemetry requires understanding the unique challenges these systems present:

1. **Iterative loops** need span hierarchies that map to logical operations
2. **Multi-agent delegation** requires explicit context propagation
3. **Tool execution** benefits from opt-in HTTP tracing to reduce noise
4. **Production deployment** needs health check exclusions and log level control

The ~50 commits in KAOS's OpenTelemetry integration represent real discoveries about how observability patterns apply to AI-native applications. The lessons learned—use standard env vars, always use try/finally for spans, log before span close, exclude health checks, make HTTP tracing opt-in—apply to any agentic system.

The agents of tomorrow will be as ubiquitous as microservices are today. And just like microservices needed distributed tracing to become production-ready, agents need OpenTelemetry.

Start instrumenting now.

---

## Resources

- **KAOS Framework**: [github.com/axsaucedo/kaos](https://github.com/axsaucedo/kaos) - The open-source framework used in this article
- **OpenTelemetry Python**: [opentelemetry.io/docs/languages/python](https://opentelemetry.io/docs/languages/python/) - Official Python SDK documentation
- **SigNoz**: [signoz.io](https://signoz.io/) - Open-source APM with native OpenTelemetry support
- **OpenTelemetry GenAI Conventions**: [github.com/open-telemetry/semantic-conventions](https://github.com/open-telemetry/semantic-conventions) - Emerging standards for AI observability
