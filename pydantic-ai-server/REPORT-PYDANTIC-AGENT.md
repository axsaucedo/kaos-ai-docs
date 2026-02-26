# Pydantic AI Agent Architecture & KAOS Integration Analysis

> **Audience**: Senior engineers working on the KAOS agent framework and its Pydantic AI integration.
> **Purpose**: Detailed reference for understanding Pydantic AI's agent internals and how KAOS wraps them, to inform refactoring decisions.

---

## 1. Pydantic AI Agent Architecture

### Core Classes

The agent hierarchy is split across two files:

- **`AbstractAgent[AgentDepsT, OutputDataT]`** in `agent/abstract.py` (~71KB): Base class providing the execution API — `run()`, `run_sync()`, `run_stream()`, `run_stream_events()`, `iter()`. Defines the contract but defers graph construction to subclasses.
- **`Agent[AgentDepsT, OutputDataT]`** in `agent/__init__.py` (~79KB): Concrete implementation, declared as a `@dataclass`. This is what users instantiate directly.

Key constructor parameters on `Agent`:

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | `str \| Model \| None` | Model to use (can be deferred) |
| `instructions` | `str \| Callable` | System prompt (static or dynamic via deps) |
| `deps_type` | `type[AgentDepsT]` | Type of dependencies |
| `output_type` | `type[OutputDataT]` | Expected output type (validated) |
| `tools` | `list[Tool \| Callable]` | Tool functions or Tool instances |
| `toolsets` | `list[Toolset]` | Toolset instances (e.g., `MCPServerStreamableHTTP`) |
| `history_processors` | `list[Callable]` | Transform message list before each model request |
| `end_strategy` | `EndStrategy` | When to stop (`"early"` or `"exhaustive"`) |
| `retries` | `int` | Default retry count for failed tool calls |
| `usage_limits` | `UsageLimits` | Token/request limits |
| `model_settings` | `ModelSettings` | Temperature, top_p, etc. |
| `defer_model_check` | `bool` | Delay model validation to run time |
| `name` | `str \| None` | Agent name (used in telemetry spans) |

### Execution Model (Graph-Based)

Agent runs are implemented as a **graph execution** via `_agent_graph.py` (~67KB). The graph is not user-facing but drives all execution:

```
UserPromptNode → ModelRequestNode → CallToolsNode ─┐
                       ▲                            │
                       └────── (more tool calls) ───┘
                                                    │
                                                    ▼
                                                   End
```

Key graph nodes:

- **`UserPromptNode`** — Injects the user prompt into the message list, resolves system prompts (including dynamic ones that access deps).
- **`ModelRequestNode`** — Sends messages to the model, receives `ModelResponse`. Checks usage limits. If the response contains tool calls, transitions to `CallToolsNode`; otherwise transitions to `End`.
- **`CallToolsNode`** — Executes tool calls (MCP tools, native tools, delegation tools) in parallel via `asyncio`. Appends `ToolReturnPart` results to messages. Transitions back to `ModelRequestNode`.
- **`End`** — Terminal node, captures the final output.

The `iter()` method exposes this graph for **step-by-step control**:

```python
async with agent.iter(prompt, deps=deps, message_history=history) as run:
    async for node in run:
        if isinstance(node, CallToolsNode):
            # KAOS uses this to emit progress events
            ...
result = run.result
```

This is critical for KAOS's streaming architecture — it intercepts `CallToolsNode` to emit tool-call progress JSON to the client before the final response.

### Tool Registration

Four mechanisms for registering tools:

1. **`@agent.tool`** — Decorator. Tool function receives `RunContext[DepsT]` as first parameter:
   ```python
   @agent.tool
   async def lookup(ctx: RunContext[MyDeps], query: str) -> str:
       return await ctx.deps.db.search(query)
   ```

2. **`@agent.tool_plain`** — Decorator. Tool function has NO `RunContext`:
   ```python
   @agent.tool_plain
   def add(a: int, b: int) -> int:
       return a + b
   ```

3. **`tools=[...]` constructor param** — List of functions or `Tool` instances:
   ```python
   agent = Agent(model, tools=[lookup, Tool(add, retries=3)])
   ```

4. **`toolsets=[...]`** — List of `Toolset` instances (including MCP servers):
   ```python
   agent = Agent(model, toolsets=[MCPServerStreamableHTTP(url="http://...")])
   ```

Tools are managed internally by `_tool_manager.py`, which handles schema generation, argument validation, and parallel execution.

### Dependencies (RunContext)

Dependencies provide typed, runtime-injected context to tools and system prompts:

```python
@dataclass
class MyDeps:
    session_id: str
    db: Database

agent = Agent(model, deps_type=MyDeps)
result = await agent.run("query", deps=MyDeps(session_id="abc", db=db))
```

Tools access deps via `RunContext`:
```python
@agent.tool
async def search(ctx: RunContext[MyDeps], query: str) -> str:
    return await ctx.deps.db.search(query)
    # ctx.deps.session_id also available
```

System prompts can also be dynamic and access deps:
```python
@agent.system_prompt
async def sys(ctx: RunContext[MyDeps]) -> str:
    return f"Session: {ctx.deps.session_id}"
```

**KAOS uses** `AgentDeps(session_id: str, memory: Optional[Memory])` as its deps type, threading session identity and memory access through to tools and prompts.

### Message History

The message system is the backbone of multi-turn conversations:

- **`message_history`** parameter on `run()` / `run_stream()` / `iter()` — type: `list[ModelMessage]`
- **`ModelMessage`** is a union: `ModelRequest | ModelResponse`

**`ModelRequest` parts** (what we send to the model):
- `UserPromptPart` — User's text input
- `ToolReturnPart` — Result from a tool call (includes `tool_call_id`)
- `RetryPromptPart` — Retry instruction after a failed tool call
- `SystemPromptPart` — System-level instructions

**`ModelResponse` parts** (what the model returns):
- `TextPart` — Generated text
- `ToolCallPart` — Tool call request (name, arguments, tool_call_id)
- `ThinkingPart` — Model's chain-of-thought (if supported)

**History retrieval**:
```python
result = await agent.run("hello", message_history=previous_messages)
result.all_messages()   # Full history (previous + new)
result.new_messages()   # Only messages from this run
```

**Serialization**: `ModelMessagesTypeAdapter` handles JSON serialization/deserialization of message lists.

**`history_processors`** — List of callables that transform the message list before each model request:
```python
# Type signature (sync or async, with or without RunContext):
Callable[[list[ModelMessage]], list[ModelMessage]]
```
These run before every model call in the loop, enabling message trimming, summarization, or filtering.

### Streaming

Multiple streaming APIs with different granularity levels:

1. **`run_stream()`** — Returns `StreamedRunResult` with:
   - `stream_text()` — Yields text chunks as they arrive
   - `stream_output()` — Yields validated output chunks
   - `stream()` — Yields raw stream events

2. **`run_stream_events()`** — Yields typed `AgentStreamEvent` objects:
   - `PartStartEvent` — A new response part begins
   - `PartDeltaEvent` — Incremental content for a part
   - `PartEndEvent` — A response part completes
   - `FunctionToolCallEvent` — A tool is being called
   - `FunctionToolResultEvent` — A tool returned a result
   - `FinalResultEvent` — The agent's final output

3. **`iter()`** — Node-by-node graph control (most granular). KAOS uses this to:
   - Detect `CallToolsNode` → emit progress JSON to the client
   - Detect `End` → stream the final response
   - Control exactly when streaming starts/stops

4. **`event_stream_handler`** — Callback for processing tool/delegate events during streaming.

### Model Resolution

Models can be specified in multiple ways:

- **String format**: `"provider:model_name"` — e.g., `"openai:gpt-4"`, `"anthropic:claude-3-5-sonnet"`
- **Explicit construction**:
  ```python
  from pydantic_ai.models.openai import OpenAIChatModel
  from pydantic_ai.providers.openai import OpenAIProvider

  model = OpenAIChatModel(
      model_name="gpt-4",
      provider=OpenAIProvider(base_url="http://localhost:8080/v1", api_key="key")
  )
  ```
- **Function model** (for testing/mocking):
  ```python
  from pydantic_ai.models.function import FunctionModel
  model = FunctionModel(my_handler_function)
  ```
- **`defer_model_check=True`** — Delays model validation until `run()` time, useful when the model is set dynamically.

### Instrumentation

Pydantic AI has built-in OpenTelemetry support:

```python
Agent.instrument_all(InstrumentationSettings(
    tracer_provider=tracer_provider,
    meter_provider=meter_provider,
    logger_provider=logger_provider,
    version=4,          # Span naming convention version
    event_mode="logs",  # "attributes" (default) or "logs"
))
```

**`InstrumentationSettings` fields**:
- `tracer_provider` — OpenTelemetry TracerProvider
- `meter_provider` — OpenTelemetry MeterProvider
- `logger_provider` — OpenTelemetry LoggerProvider
- `version` — Integer 1-4 controlling span naming conventions:
  - v1-v2: Legacy naming
  - v3+: Uses `gen_ai.*` semantic conventions (e.g., `gen_ai.invoke_agent`, `gen_ai.execute_tool`)
- `event_mode` — Controls how message events are recorded:
  - `"attributes"`: Events stored as span attributes
  - `"logs"`: Events emitted as log events correlated to spans (requires `logger_provider`)

**Generated spans**:
- Agent run: `agent run` (v1-v2) / `invoke_agent {name}` (v3+)
- Tool execution: `running tool` (v1-v2) / `execute_tool {name}` (v3+)

### Usage Limits

```python
from pydantic_ai import UsageLimits

limits = UsageLimits(
    request_limit=10,         # Max model requests per run
    tool_calls_limit=50,      # Max total tool calls
    input_tokens_limit=50000, # Max input tokens
    output_tokens_limit=10000,# Max output tokens
    total_tokens_limit=60000, # Max total tokens
)

result = await agent.run("prompt", usage_limits=limits)
```

KAOS maps its `max_steps` parameter to `UsageLimits(request_limit=max_steps)`.

---

## 2. KAOS Agent Wrapper Analysis

### What KAOS Agent Adds on Top of Pydantic AI Agent

KAOS's `Agent` class (in `data-plane/kaos-framework/agent/client.py`, ~634 lines) wraps a `pydantic_ai.Agent` and adds six capabilities:

#### 1. Model Resolution from Environment Variables

`_resolve_model()` creates the appropriate model based on env vars:

```python
def _resolve_model(self) -> Model:
    # Reads MODEL_API_URL, MODEL_NAME, DEBUG_MOCK_RESPONSES, TOOL_CALL_MODE
    if mock_responses:
        return FunctionModel(mock_handler)       # Testing mode
    if tool_call_mode == "string":
        return FunctionModel(string_mode_handler) # String tool calling
    return OpenAIChatModel(model_name, provider=OpenAIProvider(base_url=url, api_key=key))
```

This abstracts away model construction so K8s operators only need to set env vars.

#### 2. Delegation Tools

`_register_delegation_tools()` creates `delegate_to_{name}` tools for each sub-agent:

```python
def _register_delegation_tools(self):
    for name, remote_agent in self.sub_agents.items():
        tool_name = f"delegate_to_{name}"
        # Creates a pydantic_ai Tool that calls remote_agent.process_message()
        # Registers it with the underlying pydantic_ai Agent
```

The delegation tool function:
- Accepts a `task: str` argument
- Calls `RemoteAgent.process_message(task, session_id)` via HTTP
- Returns the response text as the tool result
- Creates OTEL spans for delegation tracking

#### 3. Memory Integration

Two-way conversion between KAOS memory events and Pydantic AI messages:

**`_build_message_history()`** (~31 lines): Converts KAOS `MemoryEvent` objects → Pydantic AI `ModelRequest`/`ModelResponse` list:
```python
async def _build_message_history(self, session_id: str) -> list[ModelMessage]:
    events = await self.memory.get_events(session_id, limit=self.memory_context_limit)
    messages = []
    for event in events:
        if event.role == "user":
            messages.append(ModelRequest(parts=[UserPromptPart(content=event.content)]))
        elif event.role == "assistant":
            messages.append(ModelResponse(parts=[TextPart(content=event.content)]))
    return messages
```

**`_store_pydantic_message()`**: Converts Pydantic AI result messages → KAOS `MemoryEvent` objects and stores them.

#### 4. String Tool Calling Mode

`build_string_mode_handler()` wraps a real model to handle text-based tool calling via `FunctionModel`. This enables tool calling with models that don't support native function calling (the model receives tool descriptions in the system prompt and returns JSON in text).

#### 5. Streaming with Progress Events

`process_message(stream=True)` uses `iter()` for fine-grained control:

```python
async with self._agent.iter(prompt, deps=deps, message_history=history) as run:
    async for node in run:
        if isinstance(node, CallToolsNode):
            # Emit progress JSON: {"type": "tool_call", "name": "...", ...}
            yield json.dumps(progress_event) + "\n"
    # After loop: stream final response
```

This gives clients real-time visibility into tool execution before the final answer.

#### 6. Agent Card Generation

`get_agent_card()` discovers tools from three sources (MCP servers, native tools, delegation tools) and generates an A2A-compatible agent card with capabilities metadata.

### Key Methods and Their Sizes

| Method | Lines | Purpose |
|--------|-------|---------|
| `process_message()` | ~96 | Main entry point: builds history, runs agent, stores results, handles streaming |
| `_execute_delegation()` | ~38 | Calls RemoteAgent with OTEL span + metrics |
| `_build_message_history()` | ~31 | KAOS events → Pydantic AI messages |
| `_register_delegation_tools()` | ~30 | Creates delegate_to_ tools |
| `_resolve_model()` | ~34 | Env var → Model instance |

### Additional KAOS Infrastructure

- **OTEL delegation spans** — `_execute_delegation()` creates spans with `kaos.delegation.*` attributes and counter metrics
- **Error handling** — All processing wrapped in try/except with memory event storage for debugging

---

## 3. What Could Be Extracted as Utilities

### Utility Functions That Extend a Standard `pydantic_ai.Agent`

These could live in a `kaos.agent.utils` or `kaos.extensions` module:

#### 1. `kaos.resolve_model()`
```python
def resolve_model(
    model_api_url: str,
    model_name: str,
    tool_call_mode: str = "auto",    # "auto" | "native" | "string"
    mock_responses: list[str] | None = None,
) -> tuple[Model, MockState | None]:
    """Returns (Model, optional MockState) from environment-style config."""
```

#### 2. `kaos.add_delegation_tools()`
```python
def add_delegation_tools(
    agent: pydantic_ai.Agent,
    sub_agents: dict[str, RemoteAgent],
    memory: Memory | None = None,
    memory_context_limit: int = 50,
) -> None:
    """Registers delegate_to_{name} tools on the agent for each sub-agent."""
```

#### 3. `kaos.add_memory_hooks()`
```python
def add_memory_hooks(
    agent: pydantic_ai.Agent,
    memory: Memory,
) -> None:
    """Adds history_processor that integrates KAOS memory into the message pipeline."""
```

#### 4. `kaos.build_message_history()`
```python
async def build_message_history(
    memory: Memory,
    session_id: str,
    context_limit: int = 50,
) -> list[ModelMessage]:
    """Converts KAOS memory events to Pydantic AI message history."""
```

#### 5. `kaos.store_run_messages()`
```python
async def store_run_messages(
    memory: Memory,
    session_id: str,
    result: RunResult | StreamedRunResult,
) -> None:
    """Stores Pydantic AI run result messages as KAOS memory events."""
```

#### 6. `kaos.enable_otel()`
```python
def enable_otel(
    agent: pydantic_ai.Agent,
    settings: InstrumentationSettings | None = None,
) -> None:
    """Configures OTEL instrumentation with KAOS's provider setup."""
```

### What Stays as KAOS Infrastructure (Not Extractable)

These are server/infra concerns, not agent-level utilities:

- **`AgentServer`** — FastAPI server with OpenAI-compatible `/v1/chat/completions` endpoint, health probes (`/health`, `/ready`), A2A agent card (`/.well-known/agent`), memory REST API
- **`AgentServerSettings`** — `pydantic-settings` based configuration from env vars (`AGENT_NAME`, `MODEL_API_URL`, `MODEL_NAME`, `MEMORY_TYPE`, etc.)
- **`Memory` ABC + implementations** — `LocalMemory` (in-process dict), `RedisMemory` (Redis-backed), `NullMemory` (no-op)
- **`RemoteAgent`** — HTTP client for sub-agent communication over the network
- **OTEL manager** (`telemetry/manager.py`) — `TracerProvider`, `MeterProvider`, `LoggerProvider` setup and lifecycle

---

## 4. Key Considerations for Refactor

### Memory as Pydantic AI `history_processors`

**The `history_processors` type**:
```python
Callable[[list[ModelMessage]], list[ModelMessage]]
```

**Why it CANNOT replace `_build_message_history()`**:
- `history_processors` receive an existing `list[ModelMessage]` and return a transformed version
- KAOS's `_build_message_history()` builds the list *from scratch* using KAOS `MemoryEvent` objects, not from existing Pydantic messages
- There's a semantic mismatch: processors transform; KAOS needs to construct

**Where `history_processors` CAN help**:
- Trimming/filtering messages after conversion (e.g., sliding window, token budget)
- Injecting context or instructions before each model call

**Recommended approach**: Pass `message_history=` to `run()` / `iter()` with a pre-built history from memory, then optionally use `history_processors` for trimming:

```python
history = await kaos.build_message_history(memory, session_id, limit=50)
result = await agent.run(
    prompt,
    message_history=history,
    # history_processors could do token-budget trimming here
)
await kaos.store_run_messages(memory, session_id, result)
```

### Pydantic AI `to_a2a()` and fasta2a Storage

Pydantic AI's `fasta2a` module provides A2A protocol support with a `Storage` abstraction that stores tasks AND conversation context.

**Opportunity**: Implement a KAOS `Storage` backend that delegates to KAOS Memory (Local/Redis):
- Task metadata → stored as memory events with special type
- Conversation context → already stored as memory events
- This would enable A2A protocol support alongside existing KAOS memory

**Consideration**: fasta2a's `Storage` interface may not align 1:1 with KAOS's event-based memory model. Evaluate whether the impedance mismatch justifies the integration vs. keeping them separate.

### Custom Pydantic AI Agent Pattern

KAOS already supports a custom agent pattern (see `examples/custom-agent/`):

```python
# Current pattern
custom_agent = PydanticAgent(model, tools=[...])
server = create_agent_server(custom_agent=custom_agent)
```

**Refactored pattern** using extracted utilities:

```python
# Step 1: Create standard Pydantic AI agent
agent = pydantic_ai.Agent(
    model=kaos.resolve_model(url, name, mode),
    instructions="You are a helpful assistant.",
    deps_type=AgentDeps,
)

# Step 2: Enhance with KAOS capabilities (composable)
kaos.add_delegation_tools(agent, sub_agents)
kaos.add_memory_hooks(agent, memory)
kaos.enable_otel(agent)

# Step 3: Serve
server = AgentServer(agent)
```

This pattern:
- Keeps `pydantic_ai.Agent` as the primary abstraction users interact with
- Makes KAOS enhancements **opt-in and composable** (use only what you need)
- Allows users to bring their own Pydantic AI agent with custom tools/instructions
- Reduces KAOS's `Agent` class from a monolithic wrapper to a set of focused utilities

### Migration Path

1. Extract utilities one at a time, keeping the existing `Agent` class working
2. Refactor `Agent` to use the extracted utilities internally (verify no behavior change)
3. Expose utilities as public API for advanced users
4. Eventually deprecate the monolithic `Agent` in favor of the composable pattern (optional, based on user feedback)

---

## Appendix: File Reference

| File | Size | Key Contents |
|------|------|-------------|
| `pydantic_ai/agent/abstract.py` | ~71KB | `AbstractAgent` base class |
| `pydantic_ai/agent/__init__.py` | ~79KB | `Agent` concrete implementation |
| `pydantic_ai/_agent_graph.py` | ~67KB | Graph nodes: `UserPromptNode`, `ModelRequestNode`, `CallToolsNode`, `End` |
| `pydantic_ai/_tool_manager.py` | — | Tool schema generation, execution, validation |
| `pydantic_ai/messages.py` | — | `ModelMessage`, `ModelRequest`, `ModelResponse`, all part types |
| `pydantic_ai/settings.py` | — | `ModelSettings`, `UsageLimits` |
| `pydantic_ai/models/openai.py` | — | `OpenAIChatModel`, `OpenAIProvider` |
| `pydantic_ai/models/function.py` | — | `FunctionModel` for testing/mocking |
| `kaos-framework/agent/client.py` | ~634 lines | KAOS `Agent` wrapper |
| `kaos-framework/agent/server.py` | — | `AgentServer` FastAPI application |
| `kaos-framework/agent/memory.py` | — | `Memory` ABC, `LocalMemory`, `RedisMemory`, `NullMemory` |
| `kaos-framework/agent/telemetry/` | — | OTEL instrumentation (tracing, metrics) |
