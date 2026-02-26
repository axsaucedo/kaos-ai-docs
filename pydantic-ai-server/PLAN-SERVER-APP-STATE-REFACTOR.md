# PLAN: AgentServer → AppState Functional Refactor

## Summary

This document analyzes replacing the `AgentServer` class with a plain `AppState` dataclass + module-level functions. After the models.py extraction, server.py is 730 lines with the `AgentServer` class spanning lines 92-534 (442 lines). This refactor would eliminate the class entirely, using FastAPI closure-based route patterns instead.

## Current Architecture (post-extraction)

```
agent/models.py (286 lines)
  AgentDeps, AgentCard, RemoteAgent, AgentServerSettings
  _resolve_model, _build_mock_model_function, _extract_user_prompt, _format_sse_chunk

agent/server.py (730 lines)
  configure_logging()                     # Module function (37 lines)
  AgentServer class                       # 442 lines
    __init__                              # 47 lines — stores 14 fields
    _setup_telemetry                      # 18 lines
    _lifespan                             # 8 lines
    _log_startup_config                   # 13 lines
    _setup_routes                         # 73 lines (7 route closures)
    _build_span_attrs                     # 5 lines
    _get_agent_card                       # 35 lines
    _prepare_run                          # 28 lines
    _process_message                      # 60 lines
    _complete_chat_completion             # 38 lines
    _stream_chat_completion               # 52 lines (incl. generate_stream closure)
    run                                   # 4 lines
  _parse_mcp_servers()                    # 26 lines
  _parse_sub_agents()                     # 26 lines
  _create_memory()                        # 21 lines
  _setup_otel_instrumentation()           # 19 lines
  create_agent_server()                   # 86 lines
  create_app() / get_app()               # 10 lines
```

## Proposed Architecture

### Option D: AppState + Standalone Functions

```python
# agent/server.py — ~520 lines (vs current 730)

@dataclass
class AppState:
    agent: PydanticAgent[AgentDeps]
    name: str
    description: str
    memory: Memory
    max_steps: int
    memory_context_limit: int
    mock_state: Optional[_MockResponseState]
    sub_agents: Dict[str, RemoteAgent]
    mcp_servers: list
    model: Any
    port: int
    custom_tools: list

async def prepare_run(state: AppState, message, session_id=None):
    """Shared pre-run setup."""
    ...

async def process_message(state: AppState, message, session_id=None, stream=False):
    """Core agentic loop. Yields content chunks."""
    session_id, user_prompt, history, deps, limits = await prepare_run(state, message, session_id)
    ...

async def complete_chat(state: AppState, messages, model_name, session_id=None, parent_ctx=None):
    """Non-streaming chat completion."""
    ...

async def stream_chat(state: AppState, messages, model_name, session_id=None, parent_ctx=None):
    """Streaming chat completion."""
    ...

async def get_agent_card(state: AppState, base_url: str) -> AgentCard:
    """Build A2A agent card."""
    ...

def create_app(settings=None, custom_agent=None) -> FastAPI:
    """Build AppState + FastAPI app with all routes."""
    state = _build_state(settings, custom_agent)
    app = FastAPI(title=f"Agent: {state.name}", lifespan=partial(_lifespan, state))

    @app.get("/health")
    async def health():
        return {"status": "healthy", "name": state.name, "timestamp": int(time.time())}

    @app.post("/v1/chat/completions")
    async def chat_completions(request: Request):
        ...  # calls complete_chat(state, ...) or stream_chat(state, ...)

    # ... other routes close over `state`
    return app
```

### Lines Saved

| Section | Current | After | Saved |
|---------|---------|-------|-------|
| `__init__` (47 lines) | 47 | ~15 (dataclass) | 32 |
| `self.` references (~80) | 80 occurrences | 0 | Verbosity reduction |
| `_setup_routes` method wrapper | 73 | 0 (routes in create_app) | 73 |
| `_setup_telemetry` in __init__ | 18 | 18 (same, in create_app) | 0 |
| `_lifespan` bound method | 8 | 8 (closure) | 0 |
| `run()` method | 4 | 0 (callers use uvicorn directly) | 4 |
| Class boilerplate | ~10 | 0 | 10 |
| **Total estimated** | **730** | **~520** | **~210** |

### Migration Impact

**Tests**: All 96 tests reference `AgentServer`, `server._process_message()`, etc.
- `make_test_server()` in helpers.py creates AgentServer → returns AppState
- Test calls: `server._process_message(msg)` → `process_message(state, msg)`
- ~15 test files, ~50 call sites. Mechanical find-and-replace.

**Dockerfile**: `agent.server:get_app` → still works (get_app calls create_app)

**Custom agents**: `create_agent_server(custom_agent=...)` → `create_app(custom_agent=...)` returns FastAPI app directly. Simpler API.

**E2E tests**: No change (they hit HTTP endpoints, not Python internals).

### Pros

1. **~210 lines removed** — less code to maintain
2. **No class boilerplate** — no `self.` everywhere, no `__init__` assignment block
3. **Functions are independently testable** — pass AppState, get result
4. **More idiomatic FastAPI** — closure routes are the standard pattern
5. **AppState is a plain dataclass** — trivially constructible in tests
6. **No method naming concerns** — `process_message` not `_process_message`

### Cons

1. **Breaking change** — all test patterns need updating (mechanical)
2. **Loss of encapsulation** — AppState fields are public (but they already were via `server.memory`, etc.)
3. **Functional style less familiar** — to some Python developers
4. **No `server.run()` convenience** — must call `uvicorn.run(create_app(), ...)` directly
5. **IDE navigation** — no single class to "jump to definition" on

### Risk Assessment

**Low risk** — this is purely structural refactoring with zero behavioral changes. Every function does exactly what the corresponding method did, but receives `state` instead of `self`. The test migration is mechanical.

### Recommendation

**Defer to next sprint.** The models.py extraction (already done) achieved the primary goal of breaking bidirectional dependencies. The AppState refactor provides ~210 lines of reduction but requires updating all 96 tests. This is worth doing but not urgent given the current decomposition already brought server.py from 1107 → 730 lines (34% reduction).

When executed, recommend combining with any other test-touching changes to amortize the migration cost.
