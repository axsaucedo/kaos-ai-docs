
### Implement limits on Agent CRD

```python
limits = UsageLimits(
    request_limit=10,         # Max model requests per run
    tool_calls_limit=50,      # Max total tool calls
    input_tokens_limit=50000, # Max input tokens
    output_tokens_limit=10000,# Max output tokens
    total_tokens_limit=60000, # Max total tokens
)
```


### Potential Upstream Issues

1) `instrument=True` Ignores `instrument_all()` Settings

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

2) OTEL Log Events Only Available in Deprecated Version 1

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



#### Ask question about implementation on toolset

## Decision 2: How to Handle Delegation Tools

### Option A: Utility function that registers tools on the agent

A standalone function in `tools.py` that takes a `pydantic_ai.Agent` and registers `delegate_to_{name}` tool functions on it using `@agent.tool()`.

```python
# tools.py
DELEGATION_TOOL_PREFIX = "delegate_to_"

def register_delegation_tools(
    agent: PydanticAgent,
    sub_agents: dict[str, RemoteAgent],
    memory_context_limit: int = 6,
) -> None:
    """Register delegate_to_{name} tools on a Pydantic AI agent.
    
    Each tool is a closure that captures the RemoteAgent instance.
    Tools receive AgentDeps via RunContext for session_id and memory access.
    """
    for name, remote in sub_agents.items():
        desc = f"Delegate a task to the {name} agent."
        if remote.agent_card:
            desc = f"Delegate to {remote.agent_card.name}: {remote.agent_card.description}"
        
        _register_single_delegation_tool(
            agent, f"{DELEGATION_TOOL_PREFIX}{name}",
            desc, name, remote, memory_context_limit,
        )


def _register_single_delegation_tool(agent, tool_name, description, agent_name, sub_agent, ctx_limit):
    """Register one delegation tool, capturing variables via closure."""
    @agent.tool(name=tool_name, description=description)
    async def _delegate(ctx: RunContext[AgentDeps], task: str) -> str:
        return await execute_delegation(
            agent_name, task, sub_agent,
            ctx.deps.session_id, ctx.deps.memory, ctx_limit,
        )


async def execute_delegation(
    agent_name: str,
    task: str,
    sub_agent: RemoteAgent,
    session_id: str,
    memory: Optional[Memory],
    memory_context_limit: int,
) -> str:
    """Execute delegation to a sub-agent. Contains OTEL span + metrics."""
    import time
    tracer = get_tracer()
    delegation_counter, delegation_duration = get_delegation_metrics()
    start_time = time.perf_counter()
    success = False

    with tracer.start_as_current_span(
        f"delegate.{agent_name}",
        attributes={ATTR_DELEGATION_TARGET: agent_name},
    ) as span:
        try:
            messages = []
            if session_id and memory:
                events = await memory.get_session_events(session_id)
                for event in (events[-memory_context_limit:] if events else []):
                    if event.event_type in ("user_message", "task_delegation_received"):
                        messages.append({"role": "user", "content": str(event.content)})
                    elif event.event_type == "agent_response":
                        messages.append({"role": "assistant", "content": str(event.content)})
            messages.append({"role": "task-delegation", "content": task})
            result = await sub_agent.process_message(messages)
            success = True
            return result
        except Exception as e:
            logger.error(f"Delegation to {agent_name} failed: {e}")
            from opentelemetry.trace import StatusCode, Status
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.record_exception(e)
            return f"[Delegation failed: {e}]"
        finally:
            duration_ms = (time.perf_counter() - start_time) * 1000
            if delegation_counter and delegation_duration:
                labels = {"target": agent_name, "success": str(success).lower()}
                delegation_counter.add(1, labels)
                delegation_duration.record(duration_ms, labels)
```

**Pros**:
- Works with any `pydantic_ai.Agent` instance, including custom user agents
- Simple — just call `register_delegation_tools(agent, sub_agents)` after construction
- Tools get `AgentDeps` via `RunContext` naturally (concurrency-safe)
- Easy to test: create a PydanticAgent, register tools, verify tool list

**Cons**:
- **Mutates** the agent after construction (adds tools to its internal toolset)
- If called twice with the same sub_agents, tools would be registered twice (guard needed)
- Requires the agent to use `deps_type=AgentDeps` — fails silently if not set

### Option B: Delegation as a Pydantic AI Toolset (AbstractToolset subclass)

Implement `DelegationToolset` extending `AbstractToolset[AgentDeps]`. The toolset is passed to the agent at construction time via `toolsets=[...]`.

```python
# tools.py
from pydantic_ai.toolsets.abstract import AbstractToolset, ToolsetTool
from pydantic_ai.tools import ToolDefinition
from pydantic_core import SchemaValidator, core_schema

class DelegationToolset(AbstractToolset[AgentDeps]):
    """KAOS delegation tools as a native Pydantic AI Toolset."""
    
    def __init__(
        self,
        sub_agents: dict[str, RemoteAgent],
        memory_context_limit: int = 6,
        toolset_id: str = "kaos-delegation",
    ):
        self._sub_agents = sub_agents
        self._memory_context_limit = memory_context_limit
        self._id = toolset_id

    @property
    def id(self) -> str:
        return self._id

    async def get_tools(self, ctx: RunContext[AgentDeps]) -> dict[str, ToolsetTool[AgentDeps]]:
        """Return delegation tools for all active sub-agents."""
        tools = {}
        for name, remote in self._sub_agents.items():
            if not remote._active:
                continue  # Skip unavailable sub-agents
            desc = f"Delegate a task to the {name} agent."
            if remote.agent_card:
                desc = f"Delegate to {remote.agent_card.name}: {remote.agent_card.description}"
            
            tool_name = f"{DELEGATION_TOOL_PREFIX}{name}"
            tool_def = ToolDefinition(
                name=tool_name,
                description=desc,
                parameters_json_schema={
                    "type": "object",
                    "properties": {
                        "task": {"type": "string", "description": "The task to delegate"}
                    },
                    "required": ["task"],
                },
            )
            # Build a schema validator for the args
            validator = SchemaValidator(core_schema.typed_dict_schema({
                "task": core_schema.typed_dict_field(core_schema.str_schema()),
            }))
            tools[tool_name] = ToolsetTool(
                toolset=self,
                tool_def=tool_def,
                max_retries=1,
                args_validator=validator,
            )
        return tools
    
    async def call_tool(
        self, name: str, tool_args: dict, ctx: RunContext[AgentDeps], tool: ToolsetTool
    ) -> Any:
        """Execute delegation when tool is called."""
        agent_name = name.removeprefix(DELEGATION_TOOL_PREFIX)
        sub_agent = self._sub_agents[agent_name]
        return await execute_delegation(
            agent_name, tool_args["task"], sub_agent,
            ctx.deps.session_id, ctx.deps.memory, self._memory_context_limit,
        )

# Usage at construction time:
delegation_toolset = DelegationToolset(sub_agents)
mcp_servers = [MCPServerStreamableHTTP(...), ...]
agent = PydanticAgent(
    model=model,
    toolsets=[delegation_toolset] + mcp_servers,
    deps_type=AgentDeps,
)
```

**Pros**:
- **Native Pydantic AI pattern** — toolsets are the official abstraction for dynamic tool collections
- **No mutation** — tools are declared at construction, not bolted on after
- **Composable** — works alongside MCP toolsets, function toolsets, etc. in the `toolsets=[...]` array
- **Dynamic tool availability** — `get_tools()` is called per-run, so unavailable sub-agents are automatically excluded
- **Lifecycle management** — `__aenter__`/`__aexit__` for init/cleanup (e.g., fetching agent cards at startup)
- **Durable execution** — Pydantic AI uses `id` for Temporal integration; delegation would "just work"

**Cons**:
- More code (~60 lines vs ~30 for Option A)
- Depends on `AbstractToolset` interface which is newer (but documented and stable in current Pydantic AI)
- Schema validation boilerplate (need to build `SchemaValidator` for args)
- If `AbstractToolset` API changes, refactoring is needed (low risk given it's documented public API)
- Custom agents can't easily add delegation after construction (toolsets are set at `__init__`)


## Decision 3: How to Handle Memory in the New Architecture

### Option A: KAOS MemoryEvent-based storage with utility methods on Memory class

Current approach: KAOS stores its own `MemoryEvent` objects (user_message, agent_response, tool_call, etc.). Utility methods on the `Memory` base class convert between KAOS events and Pydantic AI `ModelRequest`/`ModelResponse` for `message_history`.

```python
# memory.py — add these methods to the Memory ABC

class Memory(ABC):
    # ... existing abstract methods ...

    async def build_message_history(
        self, session_id: str, context_limit: int = 6
    ) -> Optional[list]:
        """Build Pydantic AI message_history from stored KAOS events.
        
        Returns None if no history. Excludes the latest prompt event
        (which is the current user message). Respects context_limit.
        """
        events = await self.get_session_events(session_id)
        if not events or len(events) <= 1:
            return None

        # Exclude latest prompt event (current message already sent as prompt)
        prompt_types = ("user_message", "task_delegation_received")
        exclude_idx = None
        for i in range(len(events) - 1, -1, -1):
            if events[i].event_type in prompt_types:
                exclude_idx = i
                break

        replayable = [e for i, e in enumerate(events) if i != exclude_idx]

        # Apply context limit
        if context_limit and len(replayable) > context_limit:
            replayable = replayable[-context_limit:]

        history = []
        for event in replayable:
            if event.event_type in prompt_types:
                history.append(ModelRequest(parts=[UserPromptPart(content=str(event.content))]))
            elif event.event_type == "agent_response":
                history.append(PydanticModelResponse(parts=[TextPart(content=str(event.content))]))
        return history if history else None

    async def store_pydantic_message(self, session_id: str, msg: Any) -> None:
        """Store a Pydantic AI message as KAOS memory events.
        
        Converts ModelResponse tool calls and ModelRequest tool returns
        into structured KAOS events.
        """
        if isinstance(msg, PydanticModelResponse):
            for part in msg.parts:
                if isinstance(part, ToolCallPart):
                    is_delegation = part.tool_name.startswith("delegate_to_")
                    event_type = "delegation_request" if is_delegation else "tool_call"
                    await self.add_event(session_id, event_type, {
                        "tool": part.tool_name, "arguments": part.args,
                    })
        elif isinstance(msg, ModelRequest):
            for part in msg.parts:
                if isinstance(part, ToolReturnPart):
                    is_delegation = part.tool_name.startswith("delegate_to_")
                    event_type = "delegation_response" if is_delegation else "tool_result"
                    # Parse JSON strings for cleaner storage
                    result = part.content
                    if isinstance(result, str):
                        try:
                            result = json.loads(result)
                        except (json.JSONDecodeError, ValueError):
                            pass
                    await self.add_event(session_id, event_type, {
                        "tool": part.tool_name, "result": result,
                    })
```

**How it works in practice** (server.py flow):
```python
# 1. User sends message
await memory.add_event(session_id, "user_message", "What's the weather?")

# 2. Build history for pydantic_ai (converts events → ModelRequest/ModelResponse)
history = await memory.build_message_history(session_id, context_limit=6)
# Returns: [ModelRequest(UserPromptPart("Previous question")), 
#           PydanticModelResponse(TextPart("Previous answer")), ...]

# 3. Run agent with history
result = await agent.run("What's the weather?", message_history=history, deps=deps)

# 4. Store response
await memory.add_event(session_id, "agent_response", str(result.output))

# 5. Store intermediate messages (tool calls, tool returns)
for msg in result.new_messages():
    await memory.store_pydantic_message(session_id, msg)
```

**Pros**:
- Minimal change from current architecture — same data model, same REST endpoints
- Memory events are human-readable (tool_call, delegation_request, agent_response)
- REST API (`/memory/sessions/{id}/events`) returns structured, filterable events
- Works identically with LocalMemory, RedisMemory, and NullMemory (NullMemory absorbs calls)
- Context limit is straightforward (just take last N events)
- No schema migration needed

**Cons**:
- **Lossy conversion**: Tool call IDs, thinking parts, retry attempts are not preserved in events
- Tool-call/tool-return pairing can become misaligned when context_limit trims history
- Message history doesn't include system prompts or multi-part tool calls faithfully
- Dual storage: Pydantic AI maintains its own message history internally; we duplicate in KAOS events

### Option B: Store full Pydantic AI messages natively

Replace KAOS events with serialized Pydantic AI `ModelMessage` objects. Use `ModelMessagesTypeAdapter` for JSON serialization. Keep KAOS events only for structured queries (tool_call, agent_response) as secondary indices.

```python
# memory.py — new methods added to Memory ABC

from pydantic_ai.messages import (
    ModelMessage, ModelMessagesTypeAdapter,
    ModelRequest, ModelResponse as PydanticModelResponse,
)

class Memory(ABC):
    # ... existing abstract methods ...

    @abstractmethod
    async def store_messages(
        self, session_id: str, messages: list[ModelMessage]
    ) -> bool:
        """Store complete Pydantic AI messages for a session.
        
        Replaces (not appends) the full message history.
        """
        ...

    @abstractmethod
    async def get_messages(
        self, session_id: str
    ) -> Optional[list[ModelMessage]]:
        """Retrieve stored Pydantic AI messages for a session."""
        ...


class LocalMemory(Memory):
    def __init__(self, ...):
        self._sessions: Dict[str, SessionMemory] = {}
        self._message_store: Dict[str, bytes] = {}  # session_id → serialized messages

    async def store_messages(self, session_id: str, messages: list) -> bool:
        self._message_store[session_id] = ModelMessagesTypeAdapter.dump_json(messages)
        return True

    async def get_messages(self, session_id: str) -> Optional[list]:
        raw = self._message_store.get(session_id)
        if not raw:
            return None
        return ModelMessagesTypeAdapter.validate_json(raw)


class RedisMemory(Memory):
    async def store_messages(self, session_id: str, messages: list) -> bool:
        key = f"{self._prefix}:pydantic_messages:{session_id}"
        raw = ModelMessagesTypeAdapter.dump_json(messages)
        await self._redis.set(key, raw, ex=self._session_ttl)
        return True

    async def get_messages(self, session_id: str) -> Optional[list]:
        key = f"{self._prefix}:pydantic_messages:{session_id}"
        raw = await self._redis.get(key)
        if not raw:
            return None
        return ModelMessagesTypeAdapter.validate_json(raw)
```

**How it works in practice** (server.py flow):
```python
# 1. User sends message — still store as event for REST API / debugging
await memory.add_event(session_id, "user_message", user_prompt)

# 2. Get stored Pydantic AI messages directly (no conversion)
message_history = await memory.get_messages(session_id)
# Returns: exact Pydantic AI ModelMessage objects — tool calls, returns, thinking parts included

# 3. Run agent
result = await agent.run(user_prompt, message_history=message_history, deps=deps)

# 4. Store full message history (all_messages includes previous + new)
await memory.store_messages(session_id, result.all_messages())

# 5. Also store key events for REST API queries
await memory.add_event(session_id, "agent_response", str(result.output))
for msg in result.new_messages():
    await memory.store_pydantic_message(session_id, msg)  # KAOS events for API
```

**Pros**:
- **Perfect fidelity**: Tool call IDs, thinking parts, retry content, multi-part messages all preserved
- No "orphaned tool call" bugs — Pydantic AI handles its own message pairing
- No manual conversion code for `build_message_history` — just deserialize and return
- Context limiting can be done properly: Pydantic AI's history_processors or just slice the message list
- Future-proof: as Pydantic AI adds new message types, they're automatically preserved

**Cons**:
- **Dual storage overhead**: Still need KAOS events for REST API (`/memory/sessions/{id}/events`), plus raw Pydantic AI messages for history. Two writes per request.
- **Storage size**: Serialized Pydantic AI messages can be large (include full tool call args/returns, thinking content). RedisMemory would need more storage.
- **Schema coupling**: KAOS memory becomes coupled to Pydantic AI's message schema. Version changes in Pydantic AI could break deserialization. Need `ModelMessagesTypeAdapter.validate_json()` to handle schema evolution gracefully.
- **Migration**: Existing sessions lose history (old KAOS events can't be converted to Pydantic AI messages retroactively). Only affects in-memory sessions, not persistent.
- **NullMemory**: Needs to implement both store_messages/get_messages as no-ops (trivial).




RemoteAgent:

Question whether we can actually have the remoteAgent removed, and instead just have the pydantic ai agent instead, as these are compatible with the openai completiosn endpoint.

