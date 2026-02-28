# KAOS_RELEVANCE_OVERVIEW.md — How PAIS Fits in the KAOS Ecosystem

## 1. What is KAOS?

**KAOS (K8s Agent Orchestration System)** is a Kubernetes-native platform for deploying, managing, and orchestrating AI agents at scale. It provides:

- **Custom Resource Definitions (CRDs)**: `Agent`, `MCPServer`, `ModelAPI` — declarative Kubernetes resources
- **Operator**: Go-based controller that reconciles CRDs into running deployments, services, and gateway routes
- **CLI**: `kaos` command-line tool for scaffolding, deploying, and managing agents
- **Gateway API Integration**: Envoy Gateway for HTTP routing, load balancing, and agent exposure

KAOS is the **control plane**. PAIS is the **data plane**.

---

## 2. Architecture: Where PAIS Lives

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                     │
│                                                          │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────┐ │
│  │  KAOS        │   │  Envoy       │   │  OTEL       │ │
│  │  Operator    │   │  Gateway     │   │  Collector  │ │
│  │  (Go)        │   │              │   │             │ │
│  └──────┬───────┘   └──────┬───────┘   └──────┬──────┘ │
│         │ reconciles       │ routes            │ traces  │
│         ▼                  ▼                   │        │
│  ┌─────────────────────────────────────────────┼────┐   │
│  │              Agent Pod                       │    │   │
│  │  ┌────────────────────────────────────────┐ │    │   │
│  │  │  🥧 PAIS (Pydantic AI Server)         │ │    │   │
│  │  │  ┌──────────────────────────────────┐  │ │    │   │
│  │  │  │  Pydantic AI Agent               │  │◄┘    │   │
│  │  │  │  (model calls, tools, agentic    │  │      │   │
│  │  │  │   loop)                          │  │      │   │
│  │  │  └──────────────────────────────────┘  │      │   │
│  │  │  Memory │ Delegation │ Telemetry │ A2A │      │   │
│  │  └────────────────────────────────────────┘      │   │
│  └──────────────────────────────────────────────────┘   │
│         │                    │                           │
│         ▼                    ▼                           │
│  ┌─────────────┐   ┌──────────────┐                    │
│  │  ModelAPI   │   │  MCPServer   │                    │
│  │  (LiteLLM   │   │  (MCP tool   │                    │
│  │   / Ollama) │   │   servers)   │                    │
│  └─────────────┘   └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Component | Responsibility |
|-------|-----------|---------------|
| **Control Plane** | KAOS Operator | CRD reconciliation, deployment management, RBAC, gateway routing |
| **Control Plane** | kaos CLI | Scaffolding, deployment, invocation, status |
| **Data Plane** | **PAIS** | Agent runtime: HTTP API, memory, delegation, telemetry, A2A |
| **Data Plane** | Pydantic AI | Agentic loop, model calls, tool invocation, structured output |
| **Infrastructure** | ModelAPI | LLM backend (LiteLLM proxy or hosted Ollama) |
| **Infrastructure** | MCPServer | External tool servers (Python, Kubernetes, Slack, custom) |
| **Infrastructure** | Gateway | HTTP routing, TLS, load balancing |
| **Infrastructure** | OTEL Collector | Telemetry aggregation and export |

---

## 3. Comparable Concepts

### Memory / Session Patterns

| KAOS Concept | PAIS Implementation | Details |
|-------------|---------------------|---------|
| Session persistence | `Memory` ABC | LocalMemory (dev), RedisMemory (prod), NullMemory (stateless) |
| Session ID | `AgentDeps.session_id` | Per-request, injected via Pydantic AI RunContext |
| Context windowing | `memory_context_limit` | Bounds history passed to model (default: 6) |
| Memory events | `create_event()` / `add_event()` | Types: user_message, agent_response, tool_call, delegation_request/response |
| Cross-session | `GET /memory/sessions` | List all sessions; `GET /memory/events?session_id=X` for events |

**Connection to KAOS:** The operator sets `MEMORY_TYPE` and `MEMORY_REDIS_URL` env vars based on the Agent CRD spec. When `spec.memory.type: redis` is set, KAOS deploys a Redis instance and injects the connection URL.

### Streaming

| KAOS Concept | PAIS Implementation | Details |
|-------------|---------------------|---------|
| OpenAI-compatible streaming | SSE via `/v1/chat/completions` | `stream: true` → `data: {...}\n\n` chunks |
| Progress events | Custom SSE events during tool/delegation execution | Step count, tool name, delegation target |
| Gateway streaming | Envoy Gateway passthrough | No buffering, direct SSE forwarding |

**Connection to KAOS:** The Gateway API `HTTPRoute` maps agent names to PAIS service endpoints. Streaming works transparently through the gateway.

### Observability

| KAOS Concept | PAIS Implementation | Details |
|-------------|---------------------|---------|
| Tracing | Pydantic AI `instrument_all()` + KAOS spans | Agent run, model calls, tools, delegation |
| Metrics | `kaos.delegations` counter, `kaos.delegation.duration` histogram | Plus standard HTTP metrics |
| Log correlation | `KaosLoggingHandler` | Adds trace_id/span_id to Python log records |
| Configuration | `OTEL_ENABLED`, `OTEL_EXPORTER_OTLP_ENDPOINT` | Set by operator from CRD telemetry config |

**Connection to KAOS:** The operator's `TelemetryConfig` (on Agent and MCPServer CRDs) maps to env vars. Global Helm values provide cluster-wide defaults. The OTEL Collector is deployed alongside the operator.

### Delegation / Multi-Agent

| KAOS Concept | PAIS Implementation | Details |
|-------------|---------------------|---------|
| Sub-agent references | `AGENT_SUB_AGENTS` env var | Format: `name:url,name:url` |
| Delegation tools | `DelegationToolset` (AbstractToolset) | Exposes `delegate_to_{name}` tools dynamically |
| Agent discovery | `RemoteAgent` + `AgentCard` fetch | Fetches `/.well-known/agent.json` from delegation targets |
| Network topology | Agent CRD `agentNetwork.access` | Operator resolves service URLs and injects env vars |
| Exposure | Agent CRD `agentNetwork.expose` | Creates Gateway HTTPRoute for external access |

**Connection to KAOS:** The operator reads `spec.agentNetwork.access` (list of agent names) and resolves them to Kubernetes service URLs. These are injected as `AGENT_SUB_AGENTS=name1:http://name1-svc:8000,name2:http://name2-svc:8000`. The `DelegationToolset` in PAIS reads this and creates the corresponding tools.

### RPC Endpoints / Agent Interface

| Endpoint | Purpose | Standard |
|----------|---------|----------|
| `POST /v1/chat/completions` | Chat API (primary interface) | OpenAI-compatible |
| `GET /.well-known/agent.json` | Agent discovery card | A2A protocol |
| `GET /health` | Liveness probe | Kubernetes |
| `GET /ready` | Readiness probe | Kubernetes |
| `GET /memory/events` | Memory inspection | KAOS-specific |
| `GET /memory/sessions` | Session listing | KAOS-specific |

---

## 4. Practical "How They Connect" Examples

### Example 1: Deploying an Agent with Memory

```bash
# 1. Install KAOS operator
kaos system install --gateway-enabled --metallb-enabled --wait

# 2. Deploy a hosted model (Ollama)
kaos modelapi deploy my-model --mode Hosted -m smollm2:135m --wait

# 3. Deploy agent with Redis memory
kaos agent deploy my-agent \
  --modelapi my-model \
  --model smollm2:135m \
  --expose \
  --wait
```

**What happens under the hood:**
1. Operator creates Deployment with PAIS container image
2. Sets env vars: `AGENT_NAME=my-agent`, `MODEL_API_URL=http://my-model-svc:8000`, `MODEL_NAME=smollm2:135m`
3. Creates Service for internal access
4. Creates HTTPRoute for external gateway access (because `--expose`)
5. PAIS starts, connects to ModelAPI, initializes memory, serves endpoints

### Example 2: Multi-Agent Delegation

```yaml
# orchestrator.yaml
apiVersion: kaos.io/v1alpha1
kind: Agent
metadata:
  name: orchestrator
spec:
  modelApiRef: my-model
  model: smollm2:135m
  agentNetwork:
    access: [worker-a, worker-b]
    expose: true
---
apiVersion: kaos.io/v1alpha1
kind: Agent
metadata:
  name: worker-a
spec:
  modelApiRef: my-model
  model: smollm2:135m
```

**What happens:**
1. Operator deploys both agents as separate PAIS instances
2. Orchestrator gets `AGENT_SUB_AGENTS=worker-a:http://worker-a-svc:8000,worker-b:http://worker-b-svc:8000`
3. PAIS's `DelegationToolset` creates tools: `delegate_to_worker_a`, `delegate_to_worker_b`
4. When the model decides to delegate, PAIS calls the worker's `/v1/chat/completions` endpoint
5. Response flows back through the delegation tool → model → client

### Example 3: Custom Agent with MCP Tools

```python
# server.py — pure Pydantic AI, no KAOS imports
from pydantic_ai import Agent

agent = Agent(
    model="test",
    instructions="You are a code assistant. Use the python-exec tool to run code.",
    defer_model_check=True,
)
```

```bash
# Build and deploy
pais init my-agent && cd my-agent
# Edit server.py with your tools/logic
pais build --name my-agent --tag v1 --kind-load

kaos agent deploy my-agent \
  --modelapi my-model --model smollm2:135m \
  --mcp python-runner \
  --expose --wait
```

**What happens:**
1. `pais build` creates Docker image with custom agent code on kaos-agent base image
2. Operator deploys with `MCP_SERVERS=python-runner`, `MCP_SERVER_PYTHON_RUNNER_URL=http://python-runner-svc:8000`
3. PAIS creates `MCPServerStreamableHTTP` toolset from env vars
4. Pydantic AI agent can invoke MCP tools transparently

### Example 4: Local Development Loop

```bash
# Run agent locally with Ollama
ollama serve &  # Start local Ollama

AGENT_NAME=dev-agent \
MODEL_API_URL=http://localhost:11434 \
MODEL_NAME=llama3.2 \
  pais run server.py --reload

# In another terminal
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "agent", "messages": [{"role": "user", "content": "Hello!"}]}'
```

The same code runs locally with `pais run` and in Kubernetes via KAOS — environment variables are the only difference.

---

## 5. Deployment Placement

```
Developer Machine                    Kubernetes Cluster
┌─────────────────┐                 ┌──────────────────────┐
│  pais run        │   same code    │  Agent Pod           │
│  server.py       │ ───────────►   │  PAIS container      │
│  (Ollama local)  │   same image   │  (ModelAPI service)  │
└─────────────────┘                 └──────────────────────┘
     ▲                                    ▲
     │ pais init/build                    │ kaos agent deploy
     │                                    │
     └── Developer workflow ──────────────┘
```

The development experience is:
1. **Scaffold**: `pais init my-agent`
2. **Develop**: Edit `server.py`, add tools
3. **Test locally**: `pais run --reload` (hot-reload)
4. **Build**: `pais build --name my-agent --tag v1`
5. **Deploy**: `kaos agent deploy my-agent ...`
6. **Monitor**: OTEL traces in Jaeger/SigNoz

---

## 6. Assumptions and Limitations

### Assumptions in this document
- KAOS is running on a Kubernetes cluster with Gateway API and OTEL Collector
- ModelAPI is deployed and accessible within the cluster
- The reader is familiar with basic Kubernetes concepts (Deployments, Services, CRDs)

### Current limitations
- PAIS is not published to PyPI — it's a monorepo package installed from source
- A2A implementation covers Agent Card discovery but not the full task lifecycle protocol
- String mode is a workaround — native function calling is preferred for production
- Memory is eventually consistent (Redis) — no strong ordering guarantees across replicas
- Custom agent images must extend the kaos-agent base image (not standalone Python packages)
