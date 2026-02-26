# PLAN: Restructure KAOS with Pydantic AI as Core Component

## Vision
Instead of wrapping Pydantic AI in a KAOS-specific Agent class, restructure so that:
1. Users create a standard `pydantic_ai.Agent` (or bring a custom one)
2. KAOS provides **composable utility functions** that add capabilities
3. The server works directly with a standard `pydantic_ai.Agent`
4. KAOS capabilities become opt-in additions, not mandatory wrappers

## Current Architecture (Wrapper Pattern)
```
AgentServer
  └── KAOS Agent (wrapper)
        ├── pydantic_ai.Agent (wrapped)
        ├── Memory system
        ├── Delegation tools
        ├── Model resolution
        └── OTEL configuration
```

## Proposed Architecture (Utility Pattern)
```
AgentServer (thin protocol adapter)
  └── pydantic_ai.Agent (standard, first-class)
        ├── kaos.resolve_model()     → Model
        ├── kaos.add_delegation()    → registers delegate_to_ tools
        ├── kaos.add_memory()        → sets up memory hooks + deps
        ├── kaos.enable_otel()       → configures InstrumentationSettings
        └── user tools/toolsets      → standard Pydantic AI patterns
```

## Module Structure

### `kaos/models.py` — Model Resolution
```python
def resolve_model(
    model_api_url: str,
    model_name: str,
    tool_call_mode: str = "auto",  # "auto" | "native" | "string"
    mock_responses: list[str] | None = None,
) -> tuple[Model, _MockResponseState | None]:
    """Resolve a Pydantic AI Model from KAOS operator configuration.
    
    Returns (model, mock_state). Mock state is used for DEBUG_MOCK_RESPONSES.
    """
```
- Extracts `_resolve_model()` from current `client.py` lines 211-247
- Extracts `_build_mock_model_function()` from lines 155-208
- Extracts `build_string_mode_handler()` from `string_mode.py` (unchanged)
- No Agent dependency — pure function

### `kaos/delegation.py` — Sub-Agent Delegation
```python
def add_delegation_tools(
    agent: pydantic_ai.Agent[AgentDeps, str],
    sub_agents: dict[str, RemoteAgent],
    memory_context_limit: int = 6,
) -> None:
    """Register delegate_to_{name} tools on a Pydantic AI agent.
    
    Each tool uses RunContext[AgentDeps] to access session_id and memory
    for forwarding conversation context to sub-agents.
    """
```
- Extracts `_register_delegation_tools()` and `_register_single_delegation_tool()` from `client.py`
- Extracts `_execute_delegation()` with tracing and metrics
- Uses `RunContext[AgentDeps]` pattern (same as current)
- `RemoteAgent` class stays as-is (HTTP client for sub-agent communication)

### `kaos/memory_utils.py` — Memory Integration
```python
async def build_message_history(
    memory: Memory,
    session_id: str,
    context_limit: int = 6,
) -> list[ModelMessage] | None:
    """Build Pydantic AI message_history from KAOS memory events.
    
    Converts MemoryEvents to ModelRequest/ModelResponse.
    Excludes the latest user prompt (it's passed as user_prompt to run()).
    """

async def store_run_messages(
    memory: Memory,
    session_id: str,
    messages: list[ModelMessage],
) -> None:
    """Store Pydantic AI messages as KAOS memory events.
    
    Called after agent.run() with result.new_messages().
    """

async def get_or_create_session(
    memory: Memory,
    session_id: str | None,
) -> str:
    """Get or create a memory session, returning the session_id."""
```
- Extracts `_build_message_history()` from `client.py` lines 511-542
- Extracts `_store_pydantic_message()` from lines 544-577
- Pure async functions operating on Memory + Pydantic AI message types
- Memory ABC, LocalMemory, RedisMemory, NullMemory stay as-is

### `kaos/otel.py` — OTEL Configuration
```python
def enable_otel(
    agent_name: str,
    instrumentation_version: int = 4,
    event_mode: str = "attributes",
) -> None:
    """Initialize KAOS OTEL and configure Pydantic AI instrumentation.
    
    Calls init_otel() and Agent.instrument_all() with explicit providers.
    """
```
- Wraps `init_otel()` + `PydanticAgent.instrument_all()` from `server.py` lines 629-646
- Keeps `telemetry/manager.py` as the core OTEL infrastructure

### `kaos/deps.py` — AgentDeps
```python
@dataclass
class AgentDeps:
    """Per-run dependencies for KAOS tools (delegation, memory)."""
    session_id: str = ""
    memory: Memory | None = None
```
- Same as current `AgentDeps` from `client.py` lines 51-56
- Shared by delegation tools, memory utils, and user tools

### `kaos/agent_card.py` — Agent Discovery
```python
async def build_agent_card(
    agent: pydantic_ai.Agent,
    name: str,
    description: str,
    sub_agents: dict[str, RemoteAgent],
    mcp_servers: list,
    base_url: str,
) -> AgentCard:
    """Generate A2A agent card by introspecting the Pydantic AI agent."""
```
- Extracts `get_agent_card()` from `client.py` lines 579-624
- Operates on a standard Pydantic AI agent (reads `_function_toolset`)

### `kaos/server.py` — AgentServer (Simplified)
The server becomes a thin protocol adapter:
```python
class AgentServer:
    """OpenAI-compatible server for any Pydantic AI agent."""
    
    def __init__(
        self,
        agent: pydantic_ai.Agent,
        memory: Memory,
        name: str,
        description: str = "AI Agent",
        sub_agents: dict[str, RemoteAgent] | None = None,
        port: int = 8000,
    ):
        # No KAOS Agent wrapper — works with standard pydantic_ai.Agent
        ...
```
Key changes:
- Accepts `pydantic_ai.Agent` directly (not KAOS `Agent` wrapper)
- `process_message()` logic moves into server methods
- Streaming via `agent.iter()` stays (for progress events)
- Non-streaming via `agent.run()` stays
- Memory session management and event storage done in server
- Health/ready/agent-card/memory-events endpoints stay

### `kaos/factory.py` — Convenience Factory
For users who want the "batteries included" experience:
```python
def create_kaos_agent(
    settings: AgentServerSettings | None = None,
) -> tuple[pydantic_ai.Agent, AgentServer]:
    """Create a fully configured Pydantic AI agent with KAOS utilities.
    
    This is equivalent to the current `create_agent_server()` but returns
    both the raw Pydantic AI agent and the server.
    """
    settings = settings or AgentServerSettings()
    
    # 1. Resolve model
    model, mock_state = resolve_model(...)
    
    # 2. Create standard Pydantic AI agent
    agent = pydantic_ai.Agent(
        model=model,
        instructions=settings.agent_instructions,
        name=settings.agent_name,
        deps_type=AgentDeps,
        defer_model_check=True,
    )
    
    # 3. Add KAOS capabilities
    add_delegation_tools(agent, sub_agents)
    
    # 4. Enable OTEL
    enable_otel(settings.agent_name)
    
    # 5. Create server
    server = AgentServer(agent, memory, ...)
    
    return agent, server
```

## Custom Agent Pattern
Users building custom images:
```python
from pydantic_ai import Agent
from kaos.deps import AgentDeps
from kaos.delegation import add_delegation_tools
from kaos.otel import enable_otel

# Create standard Pydantic AI agent with custom tools
agent = Agent(
    model='openai:gpt-4',
    deps_type=AgentDeps,
    instructions="You are a specialized agent.",
)

@agent.tool
async def my_custom_tool(ctx: RunContext[AgentDeps], query: str) -> str:
    return f"Result for {query}"

# Optionally add KAOS capabilities
# add_delegation_tools(agent, sub_agents)  # if needed
# enable_otel("my-agent")  # if needed

# The KAOS server handles the rest
```

## Server Process Flow (Proposed)

```
POST /v1/chat/completions
  ↓
1. Extract session_id, messages, trace context
2. session_id = await get_or_create_session(memory, session_id)
3. await memory.add_event(session_id, "user_message", prompt)
4. message_history = await build_message_history(memory, session_id)
5. deps = AgentDeps(session_id=session_id, memory=memory)
  ↓
[streaming]
6a. async with agent.iter(prompt, message_history=..., deps=deps):
      - Emit progress events for CallToolsNode
      - Yield final text
  ↓
[non-streaming]
6b. result = await agent.run(prompt, message_history=..., deps=deps)
  ↓
7. await memory.add_event(session_id, "agent_response", content)
8. await store_run_messages(memory, session_id, result.new_messages())
9. Return OpenAI-format response
```

## Pydantic AI Server Research

### Can we use `to_a2a()` from Pydantic AI?

**No, not as a replacement.** `to_a2a()` creates a **Starlette-based A2A JSON-RPC server** (via `fasta2a`), which is fundamentally different from KAOS's OpenAI-compatible server:
- A2A uses JSON-RPC protocol, not REST/SSE
- No `/v1/chat/completions` endpoint
- No health/ready probes
- No SSE streaming (tasks are async: submit → poll)
- No memory REST endpoints
- No OTEL HTTP instrumentation

**Could add as optional**: Mount `to_a2a()` at `/a2a/` for A2A interop alongside existing endpoints. But this is a separate effort and requires `fasta2a` dependency.

### Can we use their server as base?

Pydantic AI also has a `ui` module with web server capabilities, but these are for Pydantic AI's own web UI and not designed for extension. The conclusion: **keep our own FastAPI server** as the protocol adapter.

## Memory Integration Deep-Dive

### Option A: Keep Current Pattern (Recommended for v1)
- Memory stays external to Pydantic AI
- `build_message_history()` builds `message_history` parameter for each run
- `store_run_messages()` persists after each run
- Clean separation: KAOS owns persistence, Pydantic AI owns execution

### Option B: Pydantic AI history_processors (Future)
- Could use `history_processors` for trimming/filtering
- But `history_processors` transform existing history, they don't build it from external storage
- Better as an optimization layer ON TOP of Option A

### Option C: Custom Pydantic AI Message Serialization (Future)
- Use `ModelMessagesTypeAdapter` to serialize/deserialize full Pydantic AI message history
- Store complete Pydantic AI messages in KAOS memory (not event-based)
- Would preserve tool call/return pairs perfectly
- More complex but eliminates the KAOS event → Pydantic AI message conversion

## String Mode Deep-Dive

### Current: FunctionModel(handler)
- `build_string_mode_handler()` creates a closure that makes HTTP calls and parses tool call JSON from text
- Works but doesn't integrate with Pydantic AI instrumentation (no model-level tracing)

### Proposed: Keep FunctionModel for now
- Creating a proper `Model` subclass requires implementing the full `Model` interface
- `FunctionModel` is explicitly designed for this use case
- Pydantic AI's model interface may change (still evolving)
- Low priority for refactoring

## Migration Path

### Phase 1: Extract Utilities (Non-Breaking)
- Extract `kaos/models.py`, `kaos/delegation.py`, `kaos/memory_utils.py`, `kaos/otel.py`
- Keep existing `Agent` and `AgentServer` working
- New utilities are importable alongside existing code
- Tests pass with both old and new paths

### Phase 2: Simplify Server
- `AgentServer` accepts `pydantic_ai.Agent` directly
- Move `process_message()` logic into server
- Remove KAOS `Agent` wrapper
- Update `create_agent_server()` to use factory pattern

### Phase 3: Custom Agent Support
- Custom images use utility functions directly
- Template/example showing the pattern
- Documentation for building custom KAOS agents

### Phase 4: Advanced Memory (Deferred)
- Option C: Full Pydantic AI message serialization
- `history_processors` for advanced trimming
- External memory plugins (user-provided Memory implementations)

## Lines of Code Estimate

### Before (current)
- `client.py`: ~634 lines
- `server.py`: ~670 lines
- Total KAOS-specific: ~1304 lines

### After (proposed)
- `kaos/models.py`: ~90 lines
- `kaos/delegation.py`: ~80 lines
- `kaos/memory_utils.py`: ~80 lines
- `kaos/otel.py`: ~30 lines
- `kaos/deps.py`: ~15 lines
- `kaos/agent_card.py`: ~50 lines
- `kaos/server.py`: ~350 lines (simplified, includes process_message logic)
- `kaos/factory.py`: ~100 lines
- Total: ~795 lines (39% reduction)

Key reduction: eliminating the ~400 line Agent wrapper class by distributing its responsibilities into focused utility modules.

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Breaking custom agent image pattern | Phase 1 maintains backward compat; custom images adapt in Phase 3 |
| Pydantic AI API changes | `defer_model_check=True`, pin Pydantic AI version, abstract behind utilities |
| Memory consistency during migration | Keep same Memory ABC; only change how server calls it |
| OTEL span hierarchy changes | `enable_otel()` is a thin wrapper; hierarchy comes from Pydantic AI |
| E2E test breakage | Run 1-3 locally before CI push; mock responses are model-level (unchanged) |

## Decisions Deferred

1. **A2A via `to_a2a()`**: Could mount alongside existing endpoints. Separate effort.
2. **Full message serialization in memory**: Option C above. Higher fidelity but more complex.
3. **Proper Model adapter for string mode**: Wait for Pydantic AI model interface stability.
4. **history_processors integration**: Low value until message serialization is in place.
5. **Upstream contributions**: Health probes, OTEL middleware for fasta2a, OpenAI-compat endpoint.

## Conclusion

This refactoring makes KAOS a **thin orchestration layer** on top of Pydantic AI rather than a **wrapper around** it. Users get direct access to the full Pydantic AI API while KAOS adds Kubernetes-native capabilities (model resolution, delegation, memory, OTEL, server) as composable utilities.

The most impactful change is removing the `Agent` wrapper class. Everything it does can be expressed as utility functions that operate on a standard `pydantic_ai.Agent`. The server then becomes the only KAOS-specific infrastructure component, and even it becomes simpler because it delegates execution entirely to Pydantic AI.
