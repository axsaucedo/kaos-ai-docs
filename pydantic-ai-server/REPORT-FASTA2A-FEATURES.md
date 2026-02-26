# REPORT: FastA2A Codebase Analysis

## Overview

**FastA2A** is Pydantic's framework-agnostic implementation of the [A2A (Agent-to-Agent) protocol](https://a2a-protocol.org/latest/specification/) in Python. It was originally part of the pydantic-ai monorepo and has been extracted into a [standalone repository](https://github.com/pydantic/fasta2a). It is published as the `fasta2a` PyPI package and integrates with pydantic-ai via `Agent.to_a2a()`.

**Key takeaway for PAIS:** FastA2A solves the "A2A transport layer" problem — JSON-RPC routing, task lifecycle, agent card serving — while Pydantic AI's `AgentWorker` bridges FastA2A's worker abstraction to the agent runtime. PAIS currently has its own OpenAI-compatible transport layer (REST/SSE) and custom A2A discovery, which overlaps with but is architecturally different from FastA2A.

---

## 1. Architecture

### 1.1 Component Model

```
┌──────────────────────────────────────────────────────────┐
│                      FastA2A (Starlette)                 │
│                                                          │
│  Routes:                                                 │
│  ├── GET  /.well-known/agent-card.json  → AgentCard      │
│  ├── POST /                             → JSON-RPC router│
│  └── GET  /docs                         → Interactive UI │
│                                                          │
│  ┌─────────────┐    ┌──────────┐    ┌────────────────┐   │
│  │ TaskManager  │───▶│  Broker  │───▶│    Worker      │   │
│  │ (coordinate) │    │ (queue)  │    │ (execute)      │   │
│  │              │◀──▶│          │    │                │   │
│  │              │    └──────────┘    │  Your agent    │   │
│  │              │◀──────────────────▶│  runs here     │   │
│  └──────┬───────┘                    └────────────────┘   │
│         │                                                 │
│    ┌────▼────┐                                            │
│    │ Storage │                                            │
│    │(persist)│                                            │
│    └─────────┘                                            │
└──────────────────────────────────────────────────────────┘
```

Four core abstractions:
1. **FastA2A** — Starlette ASGI app. Owns routes, AgentCard generation, JSON-RPC dispatch
2. **TaskManager** — Coordinates between HTTP layer and broker/storage. Handles `message/send`, `tasks/get`, `tasks/cancel`
3. **Broker** — Queues tasks for workers. ABC with `run_task()`, `cancel_task()`, `receive_task_operations()`
4. **Storage** — Persists tasks and conversation context. Dual-purpose: A2A task state + agent-specific context
5. **Worker** — Executes tasks. ABC with `run_task()`, `cancel_task()`, `build_message_history()`, `build_artifacts()`

### 1.2 Dependencies

Minimal:
- `starlette>0.29.0` — ASGI framework
- `pydantic>=2.10` — Schema validation (TypedDicts + TypeAdapters)
- `opentelemetry-api>=1.28.0` — Tracing (API only, no SDK)
- `anyio` — Used by InMemoryBroker for in-process streams

**Does NOT depend on:** FastAPI, pydantic-ai, httpx (only in dev/client)

### 1.3 File Structure (811+149+99+132+175+70+84 ≈ ~1520 lines)

| File | Lines | Purpose |
|------|-------|---------|
| `schema.py` | 811 | Complete A2A protocol TypedDicts + JSON-RPC types |
| `task_manager.py` | 175 | TaskManager dataclass — coordinates broker/storage |
| `applications.py` | 149 | FastA2A Starlette app — routes, JSON-RPC dispatch |
| `storage.py` | 132 | Storage ABC + InMemoryStorage |
| `broker.py` | 99 | Broker ABC + InMemoryBroker (anyio streams) |
| `worker.py` | 70 | Worker ABC — agent execution abstraction |
| `client.py` | 84 | A2AClient — httpx-based JSON-RPC client |
| `__init__.py` | 7 | Re-exports |

---

## 2. A2A Protocol Implementation

### 2.1 Protocol Version

FastA2A targets **A2A v0.3.0** (latest released). The RC v1.0 adds gRPC bindings and ListTasks but the JSON-RPC binding remains backward-compatible.

### 2.2 Supported Operations

| A2A Operation | FastA2A Status | Implementation |
|---------------|----------------|----------------|
| `message/send` | ✅ Implemented | `TaskManager.send_message()` |
| `message/stream` | ❌ Not implemented | `raise NotImplementedError` |
| `tasks/get` | ✅ Implemented | `TaskManager.get_task()` |
| `tasks/cancel` | ✅ Implemented | `TaskManager.cancel_task()` |
| `tasks/pushNotification/set` | ❌ Not implemented | `raise NotImplementedError` |
| `tasks/pushNotification/get` | ❌ Not implemented | `raise NotImplementedError` |
| `tasks/resubscribe` | ❌ Not implemented | `raise NotImplementedError` |
| Agent Card discovery | ✅ Implemented | `GET /.well-known/agent-card.json` |

### 2.3 Protocol Design Decisions

- **Always creates a Task** — FastA2A never returns a direct Message response. Every `message/send` creates a task (state: `submitted`) and runs it asynchronously via the broker
- **JSON-RPC 2.0** — Single POST endpoint (`/`) with method dispatch via the `method` field
- **Opinionated**: No inline/synchronous execution. Broker always mediates between HTTP and Worker
- **context_id** — Auto-generated if not provided; groups related tasks across multi-turn conversations

### 2.4 Task Lifecycle

```
message/send → submitted → working → completed|failed|canceled
                                   → input-required (not yet implemented)
```

States: `submitted`, `working`, `input-required`, `completed`, `canceled`, `failed`, `rejected`, `auth-required`, `unknown`

### 2.5 ID Hierarchy

| ID | Scope | Generator | Purpose |
|----|-------|-----------|---------|
| `task_id` | Per-execution | Server (uuid4) | Tracks one agent run |
| `context_id` | Conversation | Client or Server | Groups tasks in a conversation thread |
| `message_id` | Per-message | Creator | Identifies individual messages |
| `artifact_id` | Per-artifact | Worker | Identifies output artifacts |

---

## 3. Key Component Details

### 3.1 FastA2A App (`applications.py`)

- Extends `Starlette` directly (not FastAPI)
- Constructor takes `storage` and `broker` as required params
- Creates `TaskManager(broker=broker, storage=storage)` internally
- Caches serialized AgentCard JSON on first request
- Routes: `/.well-known/agent-card.json` (HEAD/GET/OPTIONS), `/` (POST), `/docs` (GET, optional)
- Runtime check: if `TaskManager` not initialized (no lifespan ran), raises `RuntimeError`
- Lifespan pattern: `async with app.task_manager:` required during startup
- Supports custom Starlette routes, middleware, exception handlers via constructor

### 3.2 TaskManager (`task_manager.py`)

- Dataclass with `broker` and `storage` fields
- Async context manager (`__aenter__`/`__aexit__`) — manages broker lifecycle
- `send_message()`:
  1. Extract `context_id` from message (or generate uuid4)
  2. `storage.submit_task(context_id, message)` → creates Task with state `submitted`
  3. Build `TaskSendParams` and call `broker.run_task(params)`
  4. Return JSONRPC response with the submitted Task
- `get_task()`: Load from storage, return or 404
- `cancel_task()`: Delegate to broker, then load updated task

### 3.3 Broker (`broker.py`)

ABC with three methods:
- `run_task(params: TaskSendParams)` — Schedule execution
- `cancel_task(params: TaskIdParams)` — Request cancellation
- `receive_task_operations()` → `AsyncIterator[TaskOperation]` — Worker pulls from this

**InMemoryBroker**:
- Uses `anyio.create_memory_object_stream[TaskOperation]()` — async channel
- `run_task()`: wraps params into `_RunTask` TypedDict, captures current OTel span, sends to stream
- `receive_task_operations()`: yields from the read side of the stream
- Single-process only (no remote workers)

**OTel integration**: Captures `get_current_span()` when scheduling, propagates to worker via `_current_span` field

### 3.4 Storage (`storage.py`)

ABC with `Generic[ContextT]` — context type is parameterized:
- `load_task(task_id, history_length?)` → `Task | None`
- `submit_task(context_id, message)` → `Task` (creates new task)
- `update_task(task_id, state, new_artifacts?, new_messages?)` → `Task`
- `load_context(context_id)` → `ContextT | None`
- `update_context(context_id, context)` → None

**Dual storage model** — critical design pattern:
1. **Task storage**: A2A protocol-formatted tasks (status, artifacts, message history)
2. **Context storage**: Agent-implementation-specific format (e.g., Pydantic AI's `list[ModelMessage]`)

This separation means the worker can store rich internal state (tool calls, reasoning traces) in context, while only exposing A2A-compliant data in task history.

**InMemoryStorage**: Simple dict-based implementation. `submit_task()` generates `task_id` as uuid4.

### 3.5 Worker (`worker.py`)

ABC with `Generic[ContextT]`:
- `run_task(params: TaskSendParams)` — Execute the task
- `cancel_task(params: TaskIdParams)` — Handle cancellation
- `build_message_history(history: list[Message])` → `list[Any]` — Convert A2A messages to agent format
- `build_artifacts(result: Any)` → `list[Artifact]` — Convert agent output to A2A artifacts

**Worker._loop()**: Continuously pulls from `broker.receive_task_operations()` and dispatches to `_handle_task_operation()`.

**OTel integration**: Uses `use_span()` to restore the span captured by the broker, then creates child span for the operation.

**Error handling**: If `run_task()` raises, automatically sets task state to `failed` via storage.

**Worker.run()** is an async context manager using `anyio.create_task_group()`.

### 3.6 Pydantic AI Integration (`_a2a.py` in pydantic-ai)

`AgentWorker(Worker[list[ModelMessage]])` — the bridge between FastA2A and Pydantic AI:

```python
agent_to_a2a(agent, storage?, broker?, name?, ...) → FastA2A
```

Exposed as `Agent.to_a2a()` on the Pydantic AI Agent class.

**run_task flow:**
1. Load task from storage, verify state is `submitted`
2. Set state to `working`
3. Load conversation context (previous `list[ModelMessage]`)
4. Convert A2A message history to Pydantic AI format via `build_message_history()`
5. `result = await self.agent.run(message_history=message_history)`
6. Save `result.all_messages()` as new context
7. Convert new response messages to A2A `Message` objects
8. Build artifacts from `result.output`
9. Update task to `completed` with messages and artifacts

**Message conversion:**
- A2A → Pydantic AI: `TextPart` → `UserPromptPart`, `FilePart` → `BinaryContent`/URL types, `DataPart` → not supported
- Pydantic AI → A2A: `TextPart` → `A2ATextPart`, `ThinkingPart` → `A2ATextPart` with metadata, `ToolCallPart` → skipped

**Artifacts:** All outputs become artifacts. String results → `TextPart`, structured data → `DataPart` with JSON schema metadata.

### 3.7 Client (`client.py`)

`A2AClient` — thin httpx wrapper for JSON-RPC:
- `send_message(message, metadata?, configuration?)` — POST JSON-RPC `message/send`
- `get_task(task_id)` — POST JSON-RPC `tasks/get`
- Uses Pydantic TypeAdapters for serialization/validation

### 3.8 Interactive Docs (`/docs`)

Ships a 761-line HTML file (`static/docs.html`) — a built-in chat/debug interface for testing A2A interactions. Similar concept to FastAPI's `/docs` (Swagger UI).

---

## 4. Schema Design

### 4.1 TypedDict Pattern

All A2A types are `TypedDict`s with `@pydantic.with_config({'alias_generator': to_camel})` for camelCase JSON serialization. This is lighter than full Pydantic models:
- No runtime validation overhead on construction
- TypeAdapter used for serialization/deserialization at boundaries
- `NotRequired` for optional fields

### 4.2 JSON-RPC Types

Generic `JSONRPCRequest[Method, Params]` and `JSONRPCResponse[ResultT, ErrorT]` TypedDicts. Error types use literal type codes for type safety:

```python
TaskNotFoundError = JSONRPCError[Literal[-32001], Literal['Task not found']]
```

### 4.3 Discriminated Unions

- `A2ARequest` — discriminated on `method` field (9 request types)
- `Part` — discriminated on `kind` field (`text`, `file`, `data`)
- `TaskOperation` — discriminated on `operation` field (`run`, `cancel`)

### 4.4 TypeAdapters

Pre-built TypeAdapters for hot-path serialization:
```python
a2a_request_ta = TypeAdapter(A2ARequest)
a2a_response_ta = TypeAdapter(A2AResponse)
```

---

## 5. OpenTelemetry Integration

### 5.1 Scope

FastA2A uses `opentelemetry-api` only (no SDK dependency). Tracing is built into:
- **Broker**: Captures current span when scheduling, propagates to worker
- **Worker**: Restores broker span, creates child spans for task operations

### 5.2 Span Propagation

```python
# Broker: capture span when task is scheduled
_RunTask(operation='run', params=params, _current_span=get_current_span())

# Worker: restore and create child span
with use_span(task_operation['_current_span']):
    with tracer.start_as_current_span(f'{operation} task'):
        await self.run_task(params)
```

This ensures that HTTP request → broker → worker spans are connected in a single trace, even though broker → worker communication may be async.

### 5.3 Comparison with PAIS

| Feature | FastA2A | PAIS |
|---------|---------|------|
| OTel dependency | API only | API + SDK + exporters |
| Request spans | Via Starlette middleware | Custom `server-run` spans |
| Task execution spans | Worker creates child spans | Agent spans via Pydantic AI |
| Cross-service propagation | Span reference in TaskOperation | HTTP header injection (`inject`) |
| Metrics | None | `kaos.delegations` counter, `kaos.delegation.duration` histogram |
| Logging correlation | None built-in | KaosLoggingHandler adds trace_id/span_id |

---

## 6. Comparison: FastA2A vs PAIS Current Architecture

### 6.1 Transport Protocol

| Aspect | FastA2A | PAIS |
|--------|---------|------|
| Protocol | JSON-RPC 2.0 over HTTP POST | REST (OpenAI Chat Completions) |
| Streaming | Not implemented | SSE (Server-Sent Events) |
| Endpoint | Single POST `/` | `POST /v1/chat/completions` |
| Discovery | `GET /.well-known/agent-card.json` | `GET /.well-known/agent` |
| Health | None built-in | `GET /health`, `GET /ready` |
| Memory API | None (storage internal) | `GET /memory/events`, `/memory/sessions` |

### 6.2 Task Model

| Aspect | FastA2A | PAIS |
|--------|---------|------|
| Execution model | Async: submit → poll/subscribe | Synchronous: request → stream/response |
| Task tracking | Explicit Task with state machine | No task concept; single request-response |
| Task states | 9 states (submitted..failed) | None — success or HTTP error |
| Multi-turn | context_id groups tasks | session_id with memory |
| Result format | Artifacts (typed, named, multi-part) | OpenAI choices[].message.content |

### 6.3 Agent Card

| Field | FastA2A | PAIS |
|-------|---------|------|
| name | ✅ | ✅ |
| description | ✅ | ✅ |
| url | ✅ | ✅ |
| version | ✅ | ❌ |
| protocolVersion | ✅ (0.3.0) | ❌ |
| skills | ✅ (A2A Skill type) | ✅ (custom: name+description dicts) |
| capabilities | ✅ (streaming, push, history) | ✅ (custom: string list) |
| security | ✅ (HTTP, API Key, OAuth2, OIDC) | ❌ |
| provider | ✅ | ❌ |
| inputModes/outputModes | ✅ | ❌ |

### 6.4 Agent Communication (Delegation)

| Aspect | FastA2A | PAIS |
|--------|---------|------|
| Protocol | A2A JSON-RPC natively | OpenAI Chat Completions |
| Discovery | Agent card fetch at init | Agent card fetch at init |
| Client | `A2AClient` (httpx) | `RemoteAgent` (httpx) |
| Sub-agent model | Not built-in (app-level) | `DelegationToolset` dynamically exposes `delegate_to_*` tools |

### 6.5 Memory/Storage

| Aspect | FastA2A | PAIS |
|--------|---------|------|
| Purpose | Task state + agent context | Conversation history |
| Multi-backend | Storage ABC (pluggable) | Memory ABC (Local, Redis, Null) |
| Scope | Per-task + per-context | Per-session |
| History limit | `history_length` param | `memory_context_limit` |
| External API | `tasks/get` via JSON-RPC | `GET /memory/events`, `/memory/sessions` |

---

## 7. Key Insights for a Senior Engineer

### 7.1 FastA2A is a Transport Layer, Not an Agent Runtime

FastA2A does not execute agents. It provides:
- HTTP ↔ JSON-RPC routing
- Task lifecycle management
- Agent card serving
- Broker/worker mediation

The actual agent execution happens in `Worker.run_task()`, which the framework consumer implements. Pydantic AI provides `AgentWorker` as their implementation.

### 7.2 The Broker Pattern Enables Distributed Execution

The `Broker` abstraction is designed for workers that run in separate processes or machines. `InMemoryBroker` is a same-process implementation, but the interface supports:
- Remote task queues (Redis, RabbitMQ, etc.)
- Multi-worker round-robin
- Cancellation propagation

### 7.3 Dual Storage is Intentional

Context storage (agent-specific) vs task storage (A2A-protocol) separation means:
- Pydantic AI stores `list[ModelMessage]` (with tool calls, reasoning) in context
- Only user/agent text messages appear in A2A task history
- This is a deliberate information hiding pattern aligned with A2A's "opaque execution" principle

### 7.4 JSON-RPC is Fundamental to A2A

The A2A protocol spec is built on JSON-RPC 2.0. This is not optional — it's the primary binding. The RC v1.0 adds gRPC and REST as alternative bindings, but JSON-RPC remains the default.

### 7.5 Streaming is Not Yet Implemented

`message/stream` raises `NotImplementedError`. This is a significant gap — streaming is listed as an AgentCapability and is critical for interactive use cases.

### 7.6 No Health/Readiness Probes

FastA2A has no built-in health or readiness endpoints. For Kubernetes deployment, these would need to be added as custom Starlette routes.

### 7.7 Agent Card is Cached

The AgentCard JSON is serialized once on first request and cached (`_agent_card_json_schema`). This is efficient but means runtime changes (e.g., new tools) won't be reflected without a restart.

---

## 8. Code Quality and Maturity

- **Test coverage**: Minimal (1 test file, 4 tests — agent card and docs endpoint only)
- **Error handling**: Basic — exceptions in workers set task to `failed` state
- **Type safety**: Strong TypedDict-based types with TypeAdapter validation
- **Documentation**: Good README with architecture diagrams and usage examples
- **Maturity**: Beta (v0.x) — several operations not implemented
- **Code style**: Clean, well-documented, Ruff-formatted, pyright strict mode

---

## 9. Summary

FastA2A provides a clean, minimal A2A protocol transport layer. Its strengths are:
1. Framework-agnostic design (any agent framework can implement Worker)
2. Clean separation of concerns (HTTP ↔ task management ↔ execution)
3. Protocol-compliant A2A implementation
4. Minimal dependencies
5. Built-in OTel trace propagation

Its current limitations:
1. No streaming support
2. No health probes
3. No push notifications
4. Minimal test coverage
5. Only in-memory broker/storage shipped
6. No authentication/authorization implementation
