# PAI_SERVER_CONTEXT.md — Ecosystem Context for the Pydantic AI Server Blog Post

## 1. Pydantic AI: Structured Agent Framework

### What is Pydantic AI?

[Pydantic AI](https://ai.pydantic.dev) is a Python framework for building production-ready AI agents. It brings the reliability and ergonomics of FastAPI to GenAI development — type-safe agents, validated structured outputs, dependency injection, and first-party observability.

### Core Concepts Relevant to PAIS

- **Agent**: The central abstraction. A `pydantic_ai.Agent` instance owns a model, tools, and an agentic loop. You configure it with a model identifier, system instructions, and tool decorators.
- **Agentic Loop**: Pydantic AI manages the entire reason-act cycle internally. The `run()` / `run_stream()` methods handle model calls, tool invocations, and multi-step reasoning. PAIS never reimplements this loop.
- **Tools**: Functions decorated with `@agent.tool` or `@agent.tool_plain` are exposed to the model as callable tools. Pydantic AI validates arguments and return types.
- **AbstractToolset**: A protocol for dynamic tool sets. PAIS's `DelegationToolset` implements this to expose sub-agents as tools. `MCPServerStreamableHTTP` is another built-in toolset for MCP servers.
- **RunContext + Dependency Injection**: Per-request dependencies (like session ID, memory) are passed via `RunContext[AgentDeps]`, enabling concurrency-safe state without globals.
- **FunctionModel**: A testing/adaptation layer where a Python function handles model calls. PAIS uses this for "string mode" (injecting tool descriptions into prompts for models without native function calling) and for `DEBUG_MOCK_RESPONSES` testing.
- **instrument_all()**: Class-level OpenTelemetry configuration. When called, all Agent instances emit tracing spans for model calls, tool invocations, and the agentic loop.

### Key Design Principle

Pydantic AI is the *runtime*. PAIS is the *server*. KAOS is the *orchestrator*. Three layers, each owning its domain.

### References

- [Pydantic AI Documentation](https://ai.pydantic.dev)
- [Pydantic AI GitHub](https://github.com/pydantic/pydantic-ai) (15k+ stars)
- [Pydantic AI v1 Release Article](https://pydantic.dev/articles/pydantic-ai-v1)
- [Building Production-Ready AI Agents (Podcast)](https://www.aiengineeringpodcast.com/pydantic-ai-type-safe-agent-framework-episode-63)

---

## 2. Agent Server Responsibilities

An "agent server" is the HTTP layer that makes an AI agent accessible as a network service. PAIS addresses the following concerns:

### Streaming (SSE)

- **Pattern**: OpenAI-compatible Server-Sent Events (SSE) via `POST /v1/chat/completions` with `stream: true`
- **Format**: `data: {"choices": [{"delta": {"content": "..."}}]}\n\n` chunks, terminated by `data: [DONE]\n\n`
- **PAIS additions**: Progress events during tool execution and delegation (step count, tool name, delegation target)
- **Challenge**: SSE spans must remain open during the entire streaming response. PAIS creates the `server-run` OTEL span inside the generator function, not the route handler, to prevent premature closure.

### State / Session Management

- **Problem**: LLMs are stateless. Multi-turn conversations require external memory.
- **PAIS solution**: Memory ABC with three backends:
  - `LocalMemory`: In-process dict (single replica, development)
  - `RedisMemory`: Distributed sessions (multi-replica, production)
  - `NullMemory`: No-op (stateless agents)
- **Conversion layer**: KAOS memory events ↔ Pydantic AI `ModelRequest`/`ModelResponse` messages
- **Context windowing**: `memory_context_limit` bounds how much history is passed to the model

### Tool Routing

- **MCP tools**: Connected via `MCPServerStreamableHTTP(url + "/mcp")` as Pydantic AI toolsets
- **Delegation tools**: `DelegationToolset` exposes sub-agents as `delegate_to_{name}` tools
- **String mode**: For models without native function calling, tools are described in the system prompt and responses are parsed for `{"tool_calls": [...]}` JSON

### Health and Readiness

- `GET /health`: Kubernetes liveness probe
- `GET /ready`: Kubernetes readiness probe
- Enables zero-downtime deployments and automatic restart on failure

### Agent Discovery

- `GET /.well-known/agent.json`: A2A-compliant agent card with name, description, skills, capabilities
- Enables runtime discovery — orchestrating agents can inspect what tools and skills are available before delegating

### References

- [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat)
- [OpenAI Streaming Guide](https://developers.openai.com/api/docs/guides/streaming-responses)
- [vLLM OpenAI-Compatible Server](https://docs.vllm.ai/en/latest/serving/openai_compatible_server/)
- [Server-Sent Events Spec (W3C)](https://html.spec.whatwg.org/multipage/server-sent-events.html)

---

## 3. Protocols and Standards

### A2A (Agent-to-Agent Protocol)

The [A2A protocol](https://a2a-protocol.org/latest/specification/), launched by Google in April 2025, defines how AI agents discover and communicate with each other:

- **Agent Card**: JSON at `/.well-known/agent.json` describing agent identity, capabilities, skills
- **Task-based workflow**: Agents delegate tasks to each other with lifecycle management
- **Transport**: JSON-RPC 2.0 over HTTP(S), with SSE for streaming
- **Opaque execution**: Agents interact via declared interfaces, not shared internal state

PAIS implements the Agent Card portion of A2A. The `AgentCard` Pydantic model uses `alias_generator=to_camel` for camelCase JSON output (A2A convention). Skills are dynamically populated from connected MCP servers.

**How PAIS uses A2A:**
- `GET /.well-known/agent.json` returns the agent's card
- Delegation targets are discovered by fetching their agent cards
- The `DelegationToolset` creates `delegate_to_{name}` tools based on discovered agent capabilities

### MCP (Model Context Protocol)

The [Model Context Protocol](https://modelcontextprotocol.io/), developed by Anthropic, standardizes how LLMs connect to external tools and data:

- **Tools**: Server-exposed functions with typed parameters
- **Resources**: External data (documents, structured data)
- **Transport**: JSON-RPC 2.0, supports stdio, HTTP+SSE, and Streamable HTTP
- **Security**: Capability-based access, user consent model

PAIS connects to MCP servers via Pydantic AI's built-in `MCPServerStreamableHTTP` toolset. No custom MCP client code is needed.

**KAOS MCP architecture:**
- `MCPServer` CRD defines tool servers with runtime types (`python-string`, `kubernetes`, `slack`, `custom`)
- Operator deploys MCP servers as Kubernetes services
- Agent CRD references MCP servers by name → operator injects URLs as env vars
- PAIS reads `MCP_SERVERS` and `MCP_SERVER_<NAME>_URL` env vars at startup

### OpenAI Chat Completions API

The de facto standard for LLM interaction:
- `POST /v1/chat/completions` with `messages`, `model`, `stream`, `tools`
- Response format: `choices[].message.content` (non-streaming) or `choices[].delta.content` (streaming)
- Tool calling: `tool_calls` array in assistant messages, `tool` role for results

PAIS exposes this API, making any agent accessible to OpenAI-compatible clients, dashboards, and orchestrators.

### References

- [A2A Protocol Specification](https://a2a-protocol.org/latest/specification/)
- [A2A GitHub](https://github.com/a2aproject/A2A)
- [Google A2A Announcement](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- [MCP Specification](https://modelcontextprotocol.io/specification/2025-03-26)
- [MCP GitHub](https://github.com/modelcontextprotocol/modelcontextprotocol)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference/chat)

---

## 4. OpenTelemetry for AI Agents

### Why OTEL for Agents?

AI agents are inherently non-deterministic. Tracing provides:
- **Visibility**: What did the model decide? Which tools were called? How many steps?
- **Performance**: Model inference latency, tool execution time, end-to-end request duration
- **Debugging**: Why did delegation fail? What was the token usage? Where is the bottleneck?
- **Compliance**: Audit trails for tool invocations and data access

### PAIS OTEL Architecture

PAIS uses a layered approach:

1. **Pydantic AI instrumentation** (`Agent.instrument_all()`): Automatic spans for:
   - Agent run lifecycle
   - Individual model calls (with token counts)
   - Tool invocations
   - Multi-step reasoning iterations

2. **KAOS additions**:
   - `server-run` span wrapping the HTTP request-response cycle
   - Delegation spans (`kaos.delegation`) for sub-agent calls
   - `kaos.delegations` counter and `kaos.delegation.duration` histogram
   - Log correlation via `KaosLoggingHandler` (adds `trace_id` and `span_id` to log records)

3. **Configuration** (environment variables):
   - `OTEL_ENABLED`: Master toggle
   - `OTEL_EXPORTER_OTLP_ENDPOINT`: Where to send telemetry
   - `OTEL_INSTRUMENTATION_VERSION` (1-4): Controls Pydantic AI instrumentation detail
   - `OTEL_EVENT_MODE` (`attributes` or `logs`): Version 1 + logs emits LogRecord events; version 2+ uses span attributes

### Span Hierarchy

```
server-run (HTTP request)
  └── agent-run (Pydantic AI agentic loop)
       ├── model-call (LLM inference)
       ├── tool-call: echo (MCP tool)
       ├── model-call (LLM decides to delegate)
       ├── kaos.delegation: worker-agent (HTTP call to sub-agent)
       └── model-call (final response)
```

### Pitfalls Discovered During Development

1. **`instrument=True` on Agent constructor**: Creates fresh OTEL defaults, ignoring `instrument_all()` settings. Always use `None` (the default) and call `instrument_all()` separately.
2. **Span lifecycle with streaming**: Spans created in route handlers close before streaming completes. Create them inside the generator function.
3. **Per-tool spans**: Attempted and reverted — Pydantic AI already provides tool-level tracing, so custom spans cause double-counting.
4. **FunctionModel + ContextVar**: Pydantic AI runs FunctionModel handlers in a copied context, so `ContextVar` state doesn't persist across calls. Workaround: use mutable objects captured by closure.

### Recommended Local Observability Stacks

| Tool | Strengths | Use Case |
|------|-----------|----------|
| [SigNoz](https://signoz.io/) | All-in-one (traces, metrics, logs), OpenTelemetry-native | Full observability in one UI |
| [Jaeger](https://www.jaegertracing.io/) | Best trace visualization, lightweight | Trace-focused debugging |
| [Grafana Tempo](https://grafana.com/oss/tempo/) | Integrates with Grafana dashboards, scalable | Existing Grafana users |
| [Uptrace](https://uptrace.dev/) | Built for OTLP, good for small teams | Quick setup, free tier |

### Quick Local Setup (SigNoz)

```bash
# Docker Compose
git clone https://github.com/SigNoz/signoz.git && cd signoz/deploy
docker compose -f docker/clickhouse-setup/docker-compose.yaml up -d

# Point PAIS at it
export OTEL_ENABLED=true
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

### References

- [OpenTelemetry Python Instrumentation](https://opentelemetry.io/docs/languages/python/instrumentation/)
- [Pydantic AI + Logfire](https://pydantic.dev/logfire)
- [OTEL Semantic Conventions for GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [SigNoz Documentation](https://signoz.io/docs/)
- [Jaeger Getting Started](https://www.jaegertracing.io/docs/getting-started/)

---

## 5. Comparison with Other Frameworks

| Aspect | Pydantic AI + PAIS | LangChain/LangGraph | Google ADK | CrewAI |
|--------|-------------------|---------------------|------------|--------|
| **Philosophy** | Type-safe, minimal, Pythonic | Everything-framework, high abstraction | GCP-native, Vertex-first | Role-based agents, crew metaphor |
| **Agentic Loop** | Built-in, automatic | Manual graph construction | Built-in (GCP-dependent) | Built-in (opinionated roles) |
| **Type Safety** | Pydantic validation end-to-end | Optional, mostly runtime | Limited | Limited |
| **Memory** | Custom (Local/Redis/Null) | LangChain memory modules | Vertex-only distributed | Built-in but rigid |
| **Observability** | Native OTEL + Logfire | LangSmith (proprietary) | Cloud Trace (GCP) | Limited |
| **Vendor Lock-in** | None | Low (but LangSmith pressure) | High (GCP/Vertex) | Low |
| **Server Wrapper** | PAIS (this project) | LangServe | ADK server | CrewAI deploy |
| **MCP Support** | Native via toolsets | Via plugins | Limited | Limited |
| **A2A Support** | Agent card at /.well-known/ | None built-in | Native (same origin) | None |

### Why Pydantic AI for KAOS?

The evaluation (documented in `REPORT-RESEARCH.md`) concluded:
- ADK's distributed memory requires Vertex AI — unacceptable for a Kubernetes-native system
- LangChain's abstraction layers add complexity without proportional value
- CrewAI's role-based model doesn't map to KAOS's operator-managed agent topology
- Pydantic AI's clean interfaces, type safety, and extensibility (AbstractToolset, FunctionModel) make it the best foundation for a thin server wrapper

---

## Further Reading

### Pydantic AI
- [Official Documentation](https://ai.pydantic.dev)
- [Multi-Agent Patterns Tutorial](https://dev.to/hamluk/advanced-pydantic-ai-agents-building-a-multi-agent-system-in-pydantic-ai-1hok)
- [Agent Skills Package](https://pypi.org/project/pydantic-ai-skills/)
- [Agentic AI with Pydantic-AI (Blog Series)](https://han8931.github.io/pydantic-ai/)

### Protocols
- [A2A Protocol Full Guide (2025)](https://a2aprotocol.ai/blog/2025-full-guide-a2a-protocol)
- [MCP Developer Portal](https://modelcontextprotocol.io/)
- [A2A vs MCP: Complementary Protocols](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)

### Observability
- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Instrumenting Python with OpenTelemetry](https://edgedelta.com/company/knowledge-center/how-to-instrument-a-python-app-using-opentelemetry)
- [OTEL Metrics Best Practices](https://betterstack.com/community/guides/observability/otel-metrics-python/)

### Server Patterns
- [FastAPI + SSE Streaming](https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse)
- [OpenAI API Streaming Reference](https://developers.openai.com/api/docs/guides/streaming-responses)
- [Kubernetes Health Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
