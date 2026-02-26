# KAOS: Kubernetes Agent Orchestration System - Comprehensive Overview

This document provides a comprehensive technical overview of KAOS, a Kubernetes-native framework for deploying and orchestrating AI agents at scale.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Core Concepts](#core-concepts)
4. [Custom Resource Definitions (CRDs)](#custom-resource-definitions-crds)
5. [Control Plane - Go Operator](#control-plane---go-operator)
6. [Data Plane - Python Runtime](#data-plane---python-runtime)
7. [OpenTelemetry Integration](#opentelemetry-integration)
8. [Multi-Agent Systems](#multi-agent-systems)
9. [Deployment Model](#deployment-model)
10. [Design Philosophy](#design-philosophy)

---

## Executive Summary

### What is KAOS?

KAOS (Kubernetes Agent Orchestration System) is an open-source framework that brings Kubernetes-native orchestration to AI agent systems. It enables teams to:

- **Deploy AI agents** as first-class Kubernetes resources
- **Integrate tools** via the Model Context Protocol (MCP) standard
- **Build multi-agent systems** with hierarchical delegation
- **Observe everything** through comprehensive OpenTelemetry instrumentation

### Key Differentiators

| Feature | Description |
|---------|-------------|
| **Kubernetes-Native** | Agents are CRDs, not containers in a monolith |
| **Protocol Standards** | OpenAI-compatible APIs, MCP for tools, W3C Trace Context |
| **Observability-First** | Built-in OpenTelemetry for traces, metrics, logs |
| **Separation of Concerns** | Control plane (Go) vs Data plane (Python) |

### Technology Stack

```
┌─────────────────────────────────────────────────────────────┐
│                      User Interface                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ KAOS CLI    │  │ KAOS UI     │  │ kubectl/Helm        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Control Plane (Go)                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           KAOS Operator (kubebuilder)                │    │
│  │  ┌─────────────┐ ┌──────────────┐ ┌──────────────┐  │    │
│  │  │ Agent       │ │ MCPServer    │ │ ModelAPI     │  │    │
│  │  │ Controller  │ │ Controller   │ │ Controller   │  │    │
│  │  └─────────────┘ └──────────────┘ └──────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Plane (Python)                       │
│  ┌──────────────────┐ ┌──────────────────┐ ┌─────────────┐  │
│  │ Agent Runtime    │ │ MCPServer Runtime│ │ ModelAPI    │  │
│  │ • AgentServer    │ │ • MCP Protocol   │ │ • LiteLLM   │  │
│  │ • Agent Client   │ │ • Tool Execution │ │ • Ollama    │  │
│  │ • Memory         │ │                  │ │             │  │
│  └──────────────────┘ └──────────────────┘ └─────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Architecture Overview

### Kubernetes-Native Design

KAOS follows the Kubernetes Operator Pattern:

1. **Custom Resources** define desired state (Agent, MCPServer, ModelAPI)
2. **Controllers** reconcile actual state to match desired state
3. **Pods/Services** are created to run the actual workloads

```yaml
# User creates Agent resource
apiVersion: kaos.tools/v1alpha1
kind: Agent
metadata:
  name: researcher
spec:
  modelAPI: my-llm
  model: "openai/gpt-4"
  mcpServers:
    - search-tools
    - web-tools
  config:
    description: "Research agent with web search capabilities"
    instructions: "You are a research assistant..."
```

The operator creates:
- Deployment with agent runtime container
- Service for internal/external access
- ConfigMaps with agent configuration
- HTTPRoute for Gateway API ingress

### Request Flow

```
External Request
      │
      ▼
┌─────────────────┐
│ Gateway API     │  ← Envoy Gateway / Kong / etc.
│ (HTTPRoute)     │
└─────────────────┘
      │
      ▼
┌─────────────────┐
│ Agent Service   │  ← Kubernetes Service (ClusterIP)
└─────────────────┘
      │
      ▼
┌─────────────────┐
│ Agent Pod       │  ← Python runtime (agent/server.py)
│                 │
│ ┌─────────────┐ │
│ │ AgentServer │ │  ← FastAPI app with /v1/chat/completions
│ └──────┬──────┘ │
│        │        │
│ ┌──────▼──────┐ │
│ │    Agent    │ │  ← Agentic loop, tool calling
│ └──────┬──────┘ │
│        │        │
└────────┼────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌───────┐
│ModelAPI│ │MCPSrv │
└───────┘ └───────┘
```

---

## Core Concepts

### 1. Agent

An **Agent** is an AI entity that:
- Processes natural language requests
- Calls LLMs for reasoning
- Executes tools via MCP servers
- Delegates to other agents

**Key Properties:**
- Exposes OpenAI-compatible `/v1/chat/completions` endpoint
- Implements agentic loop (model → tools → model → ...)
- Has configurable memory for session state
- Can participate in agent networks

### 2. MCPServer (Model Context Protocol)

An **MCPServer** provides tools to agents following the MCP standard:

```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: calculator
spec:
  type: python-runtime
  config:
    tools:
      fromString: |
        def add(a: float, b: float) -> float:
            """Add two numbers."""
            return a + b
            
        def multiply(a: float, b: float) -> float:
            """Multiply two numbers."""
            return a * b
```

**Key Properties:**
- Implements MCP protocol (Server-Sent Events transport)
- Exposes tools with typed arguments and descriptions
- Automatic schema generation from Python type hints
- Supports dynamic tool loading from inline code

### 3. ModelAPI

A **ModelAPI** provides LLM access to agents:

**Proxy Mode (LiteLLM):**
```yaml
apiVersion: kaos.tools/v1alpha1
kind: ModelAPI
metadata:
  name: openai-proxy
spec:
  mode: Proxy
  proxyConfig:
    models: ["*"]
    provider: openai
    apiBase: https://api.openai.com/v1
    apiKeySecretRef:
      name: openai-secret
      key: api-key
```

**Hosted Mode (Ollama):**
```yaml
apiVersion: kaos.tools/v1alpha1
kind: ModelAPI
metadata:
  name: local-llm
spec:
  mode: Hosted
  hostedConfig:
    model: "smollm2:135m"
```

**Key Properties:**
- Exposes OpenAI-compatible API endpoint
- Proxy mode: Route to external providers (OpenAI, Anthropic, etc.)
- Hosted mode: Run models locally with Ollama
- Supports telemetry for model call instrumentation

---

## Custom Resource Definitions (CRDs)

### Agent CRD

```go
type AgentSpec struct {
    // ModelAPI reference
    ModelAPI string `json:"modelAPI"`
    Model    string `json:"model"`
    
    // Tool servers
    MCPServers []string `json:"mcpServers,omitempty"`
    
    // Multi-agent network
    AgentNetwork *AgentNetworkConfig `json:"agentNetwork,omitempty"`
    
    // Configuration
    Config *AgentConfig `json:"config,omitempty"`
}

type AgentConfig struct {
    Description           string           `json:"description,omitempty"`
    Instructions          string           `json:"instructions,omitempty"`
    ReasoningLoopMaxSteps *int32           `json:"reasoningLoopMaxSteps,omitempty"`
    Memory                *MemoryConfig    `json:"memory,omitempty"`
    Telemetry             *TelemetryConfig `json:"telemetry,omitempty"`
    Env                   []EnvVar         `json:"env,omitempty"`
}

type TelemetryConfig struct {
    Enabled  bool   `json:"enabled,omitempty"`
    Endpoint string `json:"endpoint,omitempty"`
}
```

### MCPServer CRD

```go
type MCPServerSpec struct {
    // Type of MCP server
    Type MCPServerType `json:"type"`  // "python-runtime" | "external"
    
    // Configuration
    Config *MCPServerConfig `json:"config,omitempty"`
}

type MCPServerConfig struct {
    Tools      *ToolsConfig     `json:"tools,omitempty"`
    Telemetry  *TelemetryConfig `json:"telemetry,omitempty"`
    Env        []EnvVar         `json:"env,omitempty"`
}

type ToolsConfig struct {
    FromString string `json:"fromString,omitempty"`  // Inline Python code
    FromFile   string `json:"fromFile,omitempty"`    // ConfigMap reference
}
```

### ModelAPI CRD

```go
type ModelAPISpec struct {
    Mode ModelAPIMode `json:"mode"`  // "Proxy" | "Hosted"
    
    // For Proxy mode (LiteLLM)
    ProxyConfig *ProxyConfig `json:"proxyConfig,omitempty"`
    
    // For Hosted mode (Ollama)
    HostedConfig *HostedConfig `json:"hostedConfig,omitempty"`
    
    // Telemetry
    Telemetry *TelemetryConfig `json:"telemetry,omitempty"`
}
```

---

## Control Plane - Go Operator

### Controller Architecture

Built with **kubebuilder**, the operator implements three controllers:

```
operator/controllers/
├── agent_controller.go      # Agent reconciliation
├── mcpserver_controller.go  # MCPServer reconciliation
├── modelapi_controller.go   # ModelAPI reconciliation
└── util/                    # Shared utilities
```

### Reconciliation Logic

Each controller follows the same pattern:

```go
func (r *AgentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the Agent resource
    agent := &kaosv1alpha1.Agent{}
    if err := r.Get(ctx, req.NamespacedName, agent); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Create/Update Deployment
    deployment := r.buildDeployment(agent)
    if err := r.createOrUpdate(ctx, deployment); err != nil {
        return ctrl.Result{}, err
    }
    
    // 3. Create/Update Service
    service := r.buildService(agent)
    if err := r.createOrUpdate(ctx, service); err != nil {
        return ctrl.Result{}, err
    }
    
    // 4. Create/Update HTTPRoute (if Gateway enabled)
    if r.GatewayEnabled {
        route := r.buildHTTPRoute(agent)
        if err := r.createOrUpdate(ctx, route); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 5. Update Status
    agent.Status.Ready = isDeploymentReady(deployment)
    return ctrl.Result{}, r.Status().Update(ctx, agent)
}
```

### Telemetry Integration in Operator

The operator propagates telemetry configuration to data plane components:

```go
func (r *AgentReconciler) buildTelemetryEnvVars(agent *Agent) []corev1.EnvVar {
    telemetry := r.mergeTelemetryConfig(agent)
    if !telemetry.Enabled {
        return nil
    }
    
    return []corev1.EnvVar{
        {Name: "OTEL_SDK_DISABLED", Value: "false"},
        {Name: "OTEL_EXPORTER_OTLP_ENDPOINT", Value: telemetry.Endpoint},
        {Name: "OTEL_SERVICE_NAME", Value: agent.Name},
        {Name: "OTEL_RESOURCE_ATTRIBUTES", 
         Value: fmt.Sprintf("service.namespace=%s,kaos.resource.name=%s", 
                           agent.Namespace, agent.Name)},
        {Name: "OTEL_PYTHON_FASTAPI_EXCLUDED_URLS", Value: "/health,/ready"},
    }
}
```

---

## Data Plane - Python Runtime

### Module Structure

```
python/
├── agent/              # Agent runtime
│   ├── client.py       # Agent class, RemoteAgent
│   ├── server.py       # AgentServer (FastAPI)
│   └── memory.py       # LocalMemory, NullMemory
├── mcptools/           # MCP tooling
│   ├── client.py       # MCPClient for tool discovery/execution
│   └── server.py       # MCPServer (FastMCP-based)
├── modelapi/           # Model API client
│   └── client.py       # OpenAI-compatible client
└── telemetry/          # OpenTelemetry instrumentation
    └── manager.py      # KaosOtelManager singleton
```

### Agent Class

The core Agent implements the agentic loop:

```python
class Agent:
    """AI agent with tool calling and delegation capabilities."""
    
    def __init__(
        self,
        model_api: ModelAPI,
        name: str = "agent",
        description: str = "",
        instructions: str = "",
        mcp_clients: List[MCPClient] = None,
        sub_agents: Dict[str, RemoteAgent] = None,
        memory: Memory = None,
        max_steps: int = 5,
    ):
        self.model_api = model_api
        self.name = name
        self.mcp_clients = mcp_clients or []
        self.sub_agents = sub_agents or {}
        self.memory = memory or NullMemory()
        self.max_steps = max_steps
        
    async def process_message(
        self, 
        session_id: str, 
        messages: List[Dict]
    ) -> AsyncIterator[str]:
        """Process message through agentic loop with streaming."""
        
        # Start trace span
        span = otel.span_begin("agent.agentic_loop", SpanKind.INTERNAL)
        span.set_attribute("agent.name", self.name)
        span.set_attribute("session.id", session_id)
        
        try:
            # Build system prompt with tool/agent instructions
            system_prompt = self._build_system_prompt()
            
            for step in range(self.max_steps):
                step_span = otel.span_begin(f"agent.step.{step+1}")
                
                # Call model
                response = await self._call_model(messages)
                
                # Check for tool calls
                tool_call = self._extract_tool_call(response)
                if tool_call:
                    result = await self._execute_tool(tool_call)
                    messages.append({"role": "tool", "content": result})
                    otel.span_success(step_span)
                    continue
                
                # Check for delegation
                delegation = self._extract_delegation(response)
                if delegation:
                    result = await self._delegate(delegation)
                    messages.append({"role": "assistant", "content": result})
                    otel.span_success(step_span)
                    continue
                
                # Final response
                otel.span_success(step_span)
                otel.span_success(span)
                yield response
                return
                
            otel.span_success(span)
            
        except Exception as e:
            otel.span_failure(span, e)
            raise
```

### AgentServer (FastAPI)

```python
def create_agent_server(agent: Agent) -> FastAPI:
    """Create FastAPI app for agent."""
    
    app = FastAPI(title=f"Agent: {agent.name}")
    
    # Initialize telemetry
    otel.init()
    
    @app.post("/v1/chat/completions")
    async def chat_completions(request: ChatRequest):
        """OpenAI-compatible chat endpoint."""
        
        # Extract trace context from incoming request
        from telemetry.manager import extract_and_attach_context
        context = extract_and_attach_context(request.headers)
        
        async def stream_response():
            async for chunk in agent.process_message(
                session_id=request.session_id or str(uuid.uuid4()),
                messages=request.messages
            ):
                yield format_sse_chunk(chunk)
        
        return StreamingResponse(
            stream_response(),
            media_type="text/event-stream"
        )
    
    @app.get("/health")
    async def health():
        return {"status": "healthy"}
    
    @app.get("/.well-known/agent")
    async def agent_card():
        """A2A discovery endpoint."""
        return agent.get_card().to_dict()
    
    return app
```

---

## OpenTelemetry Integration

### KaosOtelManager

The telemetry module provides a singleton manager for all OTEL operations:

```python
class KaosOtelManager:
    """OpenTelemetry manager for KAOS components."""
    
    _instance = None
    _initialized = False
    
    def __new__(cls, service_name: str = None):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def init(self, service_name: str = None):
        """Initialize OpenTelemetry SDK."""
        if self._initialized:
            return
        
        if not self.should_enable():
            return
        
        # Create resource
        resource = Resource.create({
            SERVICE_NAME: service_name or os.getenv("OTEL_SERVICE_NAME", "unknown"),
            "service.version": os.getenv("SERVICE_VERSION", "unknown"),
        })
        
        endpoint = os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
        
        # TracerProvider
        tracer_provider = TracerProvider(resource=resource)
        tracer_provider.add_span_processor(
            BatchSpanProcessor(OTLPSpanExporter(endpoint=endpoint))
        )
        trace.set_tracer_provider(tracer_provider)
        
        # MeterProvider
        meter_provider = MeterProvider(
            resource=resource,
            metric_readers=[PeriodicExportingMetricReader(
                OTLPMetricExporter(endpoint=endpoint)
            )]
        )
        metrics.set_meter_provider(meter_provider)
        
        # LoggerProvider (for OTLP log export)
        logger_provider = LoggerProvider(resource=resource)
        logger_provider.add_log_record_processor(
            BatchLogRecordProcessor(OTLPLogExporter(endpoint=endpoint))
        )
        
        # Add handler to Python root logger
        handler = KaosLoggingHandler(
            level=get_log_level_int(),
            logger_provider=logger_provider
        )
        logging.getLogger().addHandler(handler)
        
        self._initialized = True
```

### Span Management

Inline span API for async-safe tracing:

```python
# Start a span
span = otel.span_begin("model.inference", SpanKind.CLIENT)
span.set_attribute("model.name", "gpt-4")

try:
    result = await call_model()
    span.set_attribute("tokens.used", result.usage.total_tokens)
    otel.span_success(span)
except Exception as e:
    logger.error(f"Model call failed: {e}")  # Log BEFORE span_failure
    otel.span_failure(span, e)
    raise
```

### Context Propagation

For multi-agent delegation:

```python
# Inject context into outgoing request
from opentelemetry.propagate import inject

headers = {"Content-Type": "application/json"}
inject(headers)  # Adds traceparent header

response = await httpx.post(url, headers=headers, json=payload)
```

```python
# Extract context from incoming request
from opentelemetry.propagate import extract

context = extract(request.headers)
with tracer.start_as_current_span("handler", context=context) as span:
    # This span is now a child of the caller's span
    process_request()
```

---

## Multi-Agent Systems

### Agent Network Configuration

```yaml
apiVersion: kaos.tools/v1alpha1
kind: Agent
metadata:
  name: coordinator
spec:
  modelAPI: my-llm
  model: "openai/gpt-4"
  config:
    description: "Coordinator that delegates to specialist agents"
    instructions: |
      You are a coordinator. For research tasks, delegate to 'researcher'.
      For calculations, delegate to 'analyst'.
  agentNetwork:
    expose: true    # Expose A2A endpoint
    access:         # Can call these agents
      - researcher
      - analyst
```

### Delegation Flow

```
User Request: "Research AI trends and calculate market size"
                │
                ▼
┌───────────────────────────────┐
│        Coordinator            │
│                               │
│ 1. Receive request            │
│ 2. Decide: need research      │
│ 3. Delegate to researcher     │──────────────────┐
│ 4. Wait for response          │                  │
│ 5. Decide: need calculation   │                  ▼
│ 6. Delegate to analyst        │──────┐   ┌──────────────┐
│ 7. Wait for response          │      │   │  Researcher  │
│ 8. Synthesize final answer    │      │   └──────────────┘
└───────────────────────────────┘      │
                │                       ▼
                ▼               ┌──────────────┐
         Final Response         │   Analyst    │
                               └──────────────┘
```

### Trace Correlation Across Agents

```
HTTP POST /v1/chat/completions (trace_id: abc123)
└── coordinator.agent.agentic_loop
    ├── coordinator.model.inference
    ├── coordinator.delegate.researcher
    │   └── HTTP POST researcher/v1/chat/completions (trace_id: abc123) ← Same trace!
    │       └── researcher.agent.agentic_loop
    │           ├── researcher.model.inference
    │           └── researcher.tool.web_search
    ├── coordinator.model.inference
    └── coordinator.delegate.analyst
        └── HTTP POST analyst/v1/chat/completions (trace_id: abc123) ← Same trace!
            └── analyst.agent.agentic_loop
                ├── analyst.model.inference
                └── analyst.tool.calculator
```

---

## Deployment Model

### Helm Chart Structure

```
operator/chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml      # Operator deployment
│   ├── service.yaml
│   ├── configmap.yaml       # Global configuration
│   ├── rbac.yaml            # RBAC rules
│   └── crds/                # CRD definitions
└── crds/
    ├── agent-crd.yaml
    ├── mcpserver-crd.yaml
    └── modelapi-crd.yaml
```

### Global Configuration

```yaml
# values.yaml
operator:
  image: ghcr.io/axsaucedo/kaos-operator:latest
  replicas: 1

agent:
  image: ghcr.io/axsaucedo/kaos-agent:latest

mcpserver:
  image: ghcr.io/axsaucedo/kaos-mcpserver:latest

litellm:
  image: ghcr.io/axsaucedo/kaos-litellm:latest

ollama:
  image: alpine/ollama:latest

gateway:
  enabled: true
  className: envoy-gateway

telemetry:
  enabled: false
  endpoint: ""

logLevel: INFO  # TRACE, DEBUG, INFO, WARNING, ERROR
```

### Resource Creation Flow

```
User applies Agent YAML
         │
         ▼
┌─────────────────────────┐
│  Kubernetes API Server  │
└─────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│    KAOS Operator        │
│    (watching Agents)    │
└─────────────────────────┘
         │
         ├── Creates Deployment
         │   └── Pod: agent runtime container
         │       ├── Env: MODEL_API_URL, MCP_SERVER_URLS
         │       ├── Env: OTEL_* (if telemetry enabled)
         │       └── ConfigMap volume: agent config
         │
         ├── Creates Service
         │   └── ClusterIP for internal access
         │
         └── Creates HTTPRoute (if Gateway enabled)
             └── Routes external traffic to Service
```

---

## Design Philosophy

### 1. Kubernetes-Native First

Everything is a Kubernetes resource. No proprietary abstractions.

**Benefits:**
- GitOps compatible (ArgoCD, Flux)
- Familiar for Kubernetes users
- Leverages ecosystem (monitoring, RBAC, networking)

### 2. Separation of Control and Data Planes

| Aspect | Control Plane (Go) | Data Plane (Python) |
|--------|-------------------|---------------------|
| Purpose | Resource management | Request processing |
| Language | Go (kubebuilder) | Python (FastAPI) |
| Runs as | Single operator pod | Pod per Agent/MCPServer |
| State | Kubernetes API | In-memory (per pod) |

**Benefits:**
- Right tool for the job
- Independent scaling
- Clear boundaries

### 3. Protocol Standardization

| Component | Protocol |
|-----------|----------|
| Agent API | OpenAI `/v1/chat/completions` |
| Tool Integration | Model Context Protocol (MCP) |
| Observability | OpenTelemetry (OTLP) |
| Ingress | Kubernetes Gateway API |
| Trace Propagation | W3C Trace Context |

**Benefits:**
- Ecosystem compatibility
- Future-proof
- Lower learning curve

### 4. Observability as First-Class

Observability is not an afterthought:

```yaml
spec:
  config:
    telemetry:
      enabled: true
      endpoint: "http://otel-collector:4317"
```

With one flag, you get:
- Distributed traces across agents
- Metrics for latency, throughput, errors
- Log correlation with trace_id/span_id
- Context propagation for multi-agent systems

### 5. Progressive Complexity

**Simple case:** Single agent, local model
```yaml
apiVersion: kaos.tools/v1alpha1
kind: Agent
metadata:
  name: simple
spec:
  modelAPI: local-ollama
  model: "ollama/smollm2:135m"
```

**Complex case:** Multi-agent, tools, observability, external LLMs
```yaml
apiVersion: kaos.tools/v1alpha1
kind: Agent
metadata:
  name: coordinator
spec:
  modelAPI: openai-proxy
  model: "openai/gpt-4"
  mcpServers:
    - search-tools
    - code-tools
  agentNetwork:
    access:
      - researcher
      - developer
  config:
    telemetry:
      enabled: true
      endpoint: "http://signoz-collector:4317"
    memory:
      contextLimit: 10
      maxSessions: 10000
```

---

## Summary

KAOS provides:

1. **Declarative agent management** via Kubernetes CRDs
2. **Standardized interfaces** (OpenAI API, MCP, OTEL)
3. **Multi-agent orchestration** with automatic service discovery
4. **Comprehensive observability** with OpenTelemetry integration
5. **Production-ready deployment** with Helm charts and Gateway API

The OpenTelemetry integration specifically enables:
- End-to-end request tracing across agent systems
- Performance metrics for model calls, tools, delegations
- Log correlation for debugging complex agentic workflows
- Context propagation for unified traces in multi-agent scenarios
