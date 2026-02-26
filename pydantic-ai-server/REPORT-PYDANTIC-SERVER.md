# Pydantic AI Server Implementation Analysis

> **Audience**: Senior engineers evaluating KAOS ↔ Pydantic AI server architecture alignment.
> **Status**: Research report — this file is gitignored.

---

## 1. Pydantic AI Server Architecture

Pydantic AI does **NOT** have a traditional "server" in the way KAOS does. There is no built-in FastAPI app, no `/v1/chat/completions` endpoint, and no health probes. Instead, it offers a single server surface:

### A. `agent.to_a2a()` — A2A Protocol Server (via fasta2a)

The `to_a2a()` method (defined in `pydantic_ai/_a2a.py`, ~280 lines) returns a `FastA2A` Starlette ASGI application. This is built on the **fasta2a** library — a separate package (`from fasta2a.applications import FastA2A`), NOT FastAPI.

#### Architecture

```
Agent.to_a2a(**kwargs)
  └─► agent_to_a2a(agent, ...) in _a2a.py
        └─► FastA2A(
              worker=AgentWorker(agent),
              broker=InMemoryBroker(),   # or custom
              storage=InMemoryStorage(), # or custom
              ...
            )
```

**Core components:**

| Component | Class | Role |
|-----------|-------|------|
| `FastA2A` | `fasta2a.applications.FastA2A` | Starlette ASGI app, routes JSON-RPC requests |
| `TaskManager` | `fasta2a.TaskManager` | Orchestrates broker + storage + worker |
| `Broker` | `fasta2a.broker.InMemoryBroker` | Task scheduling queue (replaceable) |
| `Storage` | `fasta2a.storage.InMemoryStorage` | Dual-purpose: A2A task state + Pydantic AI context |
| `AgentWorker` | `pydantic_ai._a2a.AgentWorker` | Wraps `Agent`, calls `agent.run()` |

#### `AgentWorker` internals

```python
class AgentWorker(Worker):
    def __init__(self, agent: Agent[Any, Any], **kwargs):
        self.agent = agent
        ...

    async def run_task(self, params: TaskSendParams, context_id: str | None) -> AsyncIterator[...]:
        # 1. Load message_history from storage via context_id
        # 2. Call agent.run() with message_history
        # 3. Yield A2A task updates (artifacts, status)
        # 4. Store updated context back to storage
```

The worker calls `agent.run()` (not `agent.run_stream()`) — the A2A protocol uses task-based async execution, not SSE streaming. The `message_history` is reconstructed from stored Pydantic AI messages keyed by `context_id`.

#### Storage dual purpose

Storage serves two roles simultaneously:
1. **Task storage**: A2A task objects (id, status, artifacts) per the A2A spec
2. **Context storage**: Pydantic AI `ModelMessage` lists keyed by `context_id`

The `InMemoryStorage` default keeps both in memory. Custom implementations can persist to databases, Redis, etc. by implementing the `Storage` protocol.

#### Lifespan management

```python
# FastA2A lifespan (simplified):
async def lifespan(app):
    async with worker:           # starts AgentWorker
        async with agent:        # enters agent's context manager (model init, etc.)
            yield
```

The lifespan starts the worker's background task consumer and the agent's context managers (model client initialization, MCP server connections, etc.).

#### Extensibility

`FastA2A` inherits from `Starlette`, so it supports:
- Custom routes via `routes=[Route('/health', health_handler)]`
- Middleware via `middleware=[Middleware(CORSMiddleware, ...)]`
- Exception handlers, on_startup/on_shutdown hooks
- Mount as sub-app in a larger ASGI application

### B. No Built-in OpenAI-Compatible Server

Pydantic AI provides **none** of the following out of the box:
- `/v1/chat/completions` endpoint (OpenAI-compatible)
- `/health`, `/ready`, `/liveness` probes
- Memory management REST endpoints
- SSE streaming endpoint
- Agent card at `/.well-known/agent` (KAOS format)

The `to_a2a()` method exposes **JSON-RPC** endpoints per the A2A specification:
- `POST /` — JSON-RPC dispatch (`tasks/send`, `tasks/get`, `tasks/cancel`, etc.)
- `GET /.well-known/agent-card.json` — A2A agent card (different from KAOS format)

---

## 2. OTEL in Pydantic AI Server

### What exists

- `to_a2a()` does **NOT** add any OTEL instrumentation to the ASGI app itself
- fasta2a has `opentelemetry-api` as a dependency but uses it only for span context propagation (extracting trace context from incoming requests)
- Agent-level OTEL is handled via `InstrumentationSettings` passed to `Agent()`:

```python
from pydantic_ai import Agent
from pydantic_ai._instrumentation import InstrumentationSettings

agent = Agent(
    'openai:gpt-4o',
    instrumentation=InstrumentationSettings(
        event_mode='logs',      # or 'attributes'
        semantic_conventions=True,
    ),
)
```

- This instruments model calls (LLM spans, token counts, etc.) but NOT HTTP request/response spans

### What's missing

| Layer | Instrumented? | How |
|-------|---------------|-----|
| HTTP request/response | ❌ No | Would need `opentelemetry-instrumentation-asgi` middleware |
| Agent run | ✅ Yes | Via `InstrumentationSettings` on Agent |
| Model calls | ✅ Yes | Via logfire/OTEL integration in model classes |
| Tool execution | ✅ Yes | Spans created per tool call |
| A2A task lifecycle | ❌ No | No spans for task creation/completion/cancellation |
| Broker operations | ❌ No | No spans for enqueue/dequeue |

To get full observability on a `to_a2a()` server, you'd need to manually add:
```python
from opentelemetry.instrumentation.asgi import OpenTelemetryMiddleware

app = agent.to_a2a(middleware=[Middleware(OpenTelemetryMiddleware)])
```

---

## 3. Comparison with KAOS AgentServer

| Feature | KAOS AgentServer | Pydantic AI `to_a2a()` |
|---------|------------------|------------------------|
| **Framework** | FastAPI (Uvicorn) | Starlette (via fasta2a) |
| **Protocol** | OpenAI-compat `/v1/chat/completions` | A2A JSON-RPC (`tasks/send`, `tasks/get`) |
| **Health probes** | `GET /health`, `GET /ready` | None (must add custom routes) |
| **Agent card** | `GET /.well-known/agent` (custom schema) | `GET /.well-known/agent-card.json` (A2A spec) |
| **Memory** | KAOS Memory (Local/Redis/Null) with REST endpoints (`GET/POST /memory/{session_id}`) | fasta2a Storage (InMemory/custom), no REST API |
| **Streaming** | SSE via `text/event-stream` with `data:` deltas + `[DONE]` sentinel | Not supported — tasks are async, poll for results |
| **OTEL** | Full stack: HTTP spans, agent spans, tool spans, metrics, log correlation | Model-level only via `InstrumentationSettings` |
| **Session management** | Explicit session ID in request, memory REST API | Implicit `context_id` in A2A task params |
| **Delegation** | `delegate_to_` tool prefix with HTTP forwarding to sub-agent URLs | Not built into server — agent-level only |
| **Concurrency** | Single agent instance, async handlers | Worker-based with broker queue |
| **Error handling** | HTTP status codes (4xx, 5xx) | A2A task status (`failed` with error message) |
| **Authentication** | None built-in (relies on K8s network policies) | None built-in |

### Protocol difference detail

**KAOS request:**
```json
POST /v1/chat/completions
{
  "model": "agent",
  "messages": [{"role": "user", "content": "Hello"}],
  "stream": true,
  "session_id": "abc-123"
}
```

**A2A request:**
```json
POST /
{
  "jsonrpc": "2.0",
  "method": "tasks/send",
  "params": {
    "id": "task-uuid",
    "message": {
      "role": "user",
      "parts": [{"kind": "text", "text": "Hello"}]
    }
  }
}
```

These are fundamentally different protocols — there is no overlap.

---

## 4. Key Source Files in Pydantic AI

### Core server code

| File | Size | Contents |
|------|------|----------|
| `pydantic_ai_slim/pydantic_ai/_a2a.py` | ~280 lines | `agent_to_a2a()` function, `AgentWorker(Worker)` class, A2A ↔ Pydantic AI message conversion |
| `pydantic_ai_slim/pydantic_ai/agent/__init__.py` | ~79KB | Main `Agent` class with `to_a2a()` method (delegates to `_a2a.agent_to_a2a()`) |
| `pydantic_ai_slim/pydantic_ai/agent/abstract.py` | ~71KB | `AbstractAgent` base class — core agentic loop, `run()`, `run_stream()`, `run_sync()` |

### Instrumentation

| File | Size | Contents |
|------|------|----------|
| `pydantic_ai_slim/pydantic_ai/_instrumentation.py` | ~3KB | `InstrumentationSettings` dataclass, `InstrumentationContext` for OTEL setup |

### fasta2a (separate package)

fasta2a is imported as `from fasta2a.applications import FastA2A`. Key classes:

| Class | Role |
|-------|------|
| `FastA2A` | Starlette ASGI app — routes, lifespan, JSON-RPC dispatch |
| `TaskManager` | Coordinates broker + storage + worker for task lifecycle |
| `Worker` (ABC) | Abstract base — `AgentWorker` implements `run_task()` |
| `Broker` (ABC) | Task queue — `InMemoryBroker` is default |
| `Storage` (ABC) | Task + context persistence — `InMemoryStorage` is default |

### `to_a2a()` method signature

```python
# In Agent class (agent/__init__.py):
def to_a2a(
    self,
    *,
    storage: Storage | None = None,
    broker: Broker | None = None,
    # ... plus all Starlette app kwargs (routes, middleware, etc.)
) -> FastA2A:
    return agent_to_a2a(self, storage=storage, broker=broker, **kwargs)
```

### `agent_to_a2a()` function signature

```python
# In _a2a.py:
def agent_to_a2a(
    agent: Agent[Any, Any],
    *,
    storage: Storage | None = None,
    broker: Broker | None = None,
    name: str | None = None,
    description: str | None = None,
    version: str | None = None,
    **kwargs: Any,
) -> FastA2A:
    worker = AgentWorker(agent, name=name, description=description, version=version)
    return FastA2A(
        worker=worker,
        storage=storage or InMemoryStorage(),
        broker=broker or InMemoryBroker(),
        **kwargs,
    )
```

### `AgentWorker.run_task()` flow

```python
async def run_task(self, params, context_id):
    # 1. Retrieve stored context (message history) by context_id
    stored_messages = await self.storage.get_context(context_id)

    # 2. Convert A2A message parts to Pydantic AI message format
    user_message = convert_a2a_to_pydantic(params.message)

    # 3. Run the agent
    result = await self.agent.run(
        user_message,
        message_history=stored_messages,
    )

    # 4. Store updated context
    await self.storage.set_context(context_id, result.all_messages())

    # 5. Yield A2A task artifacts
    yield TaskArtifact(parts=[TextPart(text=result.output)])
```

---

## 5. What KAOS Could Adopt

### Option A: Use `to_a2a()` as the primary server

**Replace KAOS `AgentServer` entirely with `agent.to_a2a()`.**

| Aspect | Assessment |
|--------|------------|
| OpenAI-compat endpoint | ❌ Not available — would need custom route |
| SSE streaming | ❌ Not supported — A2A is task-based |
| Health probes | ❌ Must add as custom Starlette routes |
| Memory REST API | ❌ Must implement custom storage + routes |
| OTEL HTTP instrumentation | ❌ Must add ASGI middleware manually |
| Delegation | ❌ Not server-aware — agent-level only |
| A2A compliance | ✅ Full A2A spec support |

**Verdict**: Too many gaps. Would require more custom code bolted onto fasta2a than the current KAOS server contains. The result would be harder to maintain than the current approach.

### Option B: Keep KAOS server, add `to_a2a()` as optional route

Mount the A2A app alongside existing KAOS endpoints:

```python
from fastapi import FastAPI
from starlette.routing import Mount

kaos_app = FastAPI()
# ... existing KAOS routes ...

a2a_app = agent.to_a2a()
kaos_app.mount("/a2a", a2a_app)
```

This gives:
- `POST /v1/chat/completions` — existing KAOS OpenAI-compat
- `GET /health`, `GET /ready` — existing KAOS probes
- `POST /a2a/` — A2A JSON-RPC
- `GET /a2a/.well-known/agent-card.json` — A2A agent card

**Pros**: Adds A2A interoperability without losing anything. Both protocols coexist.

**Cons**: Two code paths for the same agent. Storage/memory divergence (KAOS Memory vs fasta2a Storage). Moderate implementation effort.

**Effort estimate**: ~2-3 days for basic integration, ~1 week with shared storage.

### Option C: Keep KAOS server as-is, use Pydantic AI agent directly

**Current approach.** KAOS `AgentServer` wraps a Pydantic AI `Agent` (or equivalent), handling:
- Protocol translation (OpenAI-compat ↔ Pydantic AI)
- Memory management (session-based)
- Health probes
- OTEL instrumentation
- Delegation routing

The server is a thin adapter around the agent runtime.

**Pros**: Already working, tested, deployed. Full control over protocol, observability, memory.

**Cons**: No A2A compliance. Custom server code to maintain.

**Verdict**: Most pragmatic for current needs.

### Recommended: Option C with Option B elements

1. **Keep KAOS server** as the primary server (Option C)
2. **Add optional A2A mount** when A2A interoperability is needed (Option B)
3. **Share storage**: Implement a KAOS `Storage` adapter that bridges fasta2a Storage ↔ KAOS Memory
4. **Phase**: Ship Option C first, add Option B when A2A adoption grows

---

## 6. Potential Upstream Contributions

### To fasta2a

| Contribution | Difficulty | Impact |
|--------------|------------|--------|
| Health/ready probe routes | Easy | High — every production deployment needs these |
| OTEL ASGI middleware (opt-in) | Medium | High — observability is table stakes |
| OpenAI-compatible endpoint adapter | Hard | Medium — alternative protocol for non-A2A clients |
| Custom storage examples (Redis, PostgreSQL) | Easy | Medium — only InMemory exists today |
| Task lifecycle OTEL spans | Medium | Medium — broker/worker operations are opaque |

### To pydantic-ai

| Contribution | Difficulty | Impact |
|--------------|------------|--------|
| `to_openai_server()` method (like `to_a2a()`) | Hard | High — many teams need OpenAI-compat serving |
| Streaming support in A2A worker | Medium | Medium — `run_stream()` instead of `run()` |
| Built-in OTEL for `to_a2a()` app | Medium | Medium — zero-config observability |

### Contribution approach

1. Start with health probes for fasta2a — small, uncontroversial, easy to review
2. Follow with OTEL middleware — demonstrates production-readiness focus
3. Propose OpenAI-compat endpoint as RFC — needs design discussion with maintainers

---

## 7. Conclusions for KAOS Refactor

### Core finding

The Pydantic AI "server" (`fasta2a`/`to_a2a()`) is designed for **A2A protocol compliance**, NOT for the OpenAI-compatible server pattern that KAOS uses. The architectures serve fundamentally different purposes:

| KAOS needs | Pydantic AI provides |
|------------|---------------------|
| OpenAI-compat SSE streaming | A2A task-based async execution |
| Health/ready probes | Nothing (must add manually) |
| Memory REST API | Context storage (no REST) |
| Full-stack OTEL | Model-level OTEL only |
| Delegation routing | Agent-level delegation only |

### Architectural implications

1. **The KAOS server layer is NOT redundant** — it provides capabilities that Pydantic AI's server does not
2. **The KAOS server should remain thin** — protocol adapter + KAOS-specific additions (memory, delegation, OTEL)
3. **A2A support can be additive** — mount alongside, not replace
4. **Storage unification is the key integration point** — bridging KAOS Memory and fasta2a Storage enables both protocols to share state

### Decision matrix

| If your goal is... | Then... |
|---------------------|---------|
| OpenAI-compat serving with SSE | Keep KAOS server (fasta2a can't do this) |
| A2A interoperability | Add `to_a2a()` mount (Option B) |
| Minimal maintenance | Keep KAOS server, skip A2A for now (Option C) |
| Maximum interoperability | KAOS server + A2A mount + shared storage (Option C+B) |
| Contributing upstream | Start with fasta2a health probes and OTEL middleware |

### Summary

The KAOS `AgentServer` and Pydantic AI's `to_a2a()` are complementary, not competing. KAOS should keep its server for OpenAI-compat workloads and optionally mount A2A for interoperability. The refactor focus should be on making the KAOS server a thin adapter around standard Pydantic AI agents, with KAOS-specific additions (memory, delegation, OTEL) as composable, well-tested utilities.
