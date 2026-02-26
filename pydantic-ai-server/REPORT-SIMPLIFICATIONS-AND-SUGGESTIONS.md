# KAOS Python Framework Simplification Report

## Implemented Simplifications

### 1. Remove `memory_enabled` flag — NullMemory as no-op (`b4b99e4`)
- **Before**: 7 `if self.memory_enabled` conditional checks scattered through `client.py`; `memory_enabled` parameter threaded through Agent constructor and server creation
- **After**: All memory calls are unconditional; NullMemory silently absorbs calls when memory is disabled
- **Impact**: -16 lines, eliminated branching complexity in the hot path

### 2. Memory ABC with shared logic (`c63ae22`)
- **Before**: `create_event()` duplicated in LocalMemory and RedisMemory (both had identical OTEL trace context injection); `build_conversation_context()` duplicated with identical formatting logic; `Memory` was a Union type alias in client.py; `InMemorySessionService` alias unused
- **After**: `Memory` ABC defines canonical interface with `create_event()` and `build_conversation_context()` as shared implementations; NullMemory, LocalMemory, RedisMemory all extend `Memory`; removed `InMemorySessionService` alias
- **Impact**: -44 lines, single source of truth for event creation and context building

### 3. Replace `_current_session_id` with RunContext deps (`7479044`)
- **Before**: Mutable `self._current_session_id` field set per-request, read by delegation tool closures; not concurrency-safe
- **After**: `AgentDeps(session_id=...)` passed via Pydantic AI `RunContext` dependency injection; delegation tools receive `ctx: RunContext[AgentDeps]` and read `ctx.deps.session_id`
- **Impact**: Concurrency-safe delegation, eliminated mutable per-Agent state

### 4. Extract `_resolve_model()` factory function (`5b02590`)
- **Before**: 30 lines of model resolution branching (mock/string/native) inside `Agent.__init__`
- **After**: Standalone `_resolve_model()` function returns `(model, mock_state)` tuple; `__init__` calls it in one line
- **Impact**: Cleaner separation of concerns, testable model resolution

### 5. Remove unused imports (`f40997e`)
- Removed from `client.py`: `is_otel_enabled`, `ATTR_SESSION_ID`, `NullMemory`
- Removed from `server.py`: `model_validator`, top-level `LocalMemory` import (duplicated by local import in `create_agent_server`)

### 6. Consolidate RemoteAgent HTTP clients (`3956fe0`)
- **Before**: Two `httpx.AsyncClient` instances — `_discovery_client` (5s timeout) and `_request_client` (60s timeout)
- **After**: Single `_client` with request timeout; discovery is a one-time call that doesn't need a dedicated client
- **Impact**: Fewer open connections, simpler lifecycle management

### 7. Remove unused OTEL constants (`47d3d20`)
- Removed `ATTR_AGENT_NAME` and `ATTR_SESSION_ID` from telemetry manager (not referenced anywhere)

### 8. Compact startup logging (`c899c72`)
- **Before**: 46 lines with decorative banners and per-field logging
- **After**: 5 structured log lines with all information
- **Impact**: -30 lines, faster startup, cleaner log output

## Summary

| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| Total lines (5 core files) | ~2659 | ~2587 | -72 |
| `if memory_enabled` checks | 7 | 0 | -7 |
| Duplicated methods (memory) | 2 | 0 | -2 |
| httpx clients (RemoteAgent) | 2 | 1 | -1 |
| Mutable per-Agent state | 1 | 0 | -1 |
| Unused imports | 6 | 0 | -6 |

---

## Future Radical Simplifications

These are larger changes deferred to follow-up work:

### F1. Replace manual `_build_message_history` with Pydantic AI history processors
**Status: DEFERRED** — History processors are `Callable[[list[ModelMessage]], list[ModelMessage]]` which only transform existing Pydantic AI messages. They cannot replace the KAOS→Pydantic message conversion step (the actual complexity). The conversion from memory events to ModelRequest/ModelResponse objects must happen before a history processor runs. Current implementation is ~30 clean lines. A deeper refactor (storing raw Pydantic AI messages in memory) would be needed to eliminate this entirely.

### F2. Unify streaming + persistence via agent-run event streaming
**Status: DEFERRED** — Analysis shows minimal duplication between streaming/non-streaming paths. Both persist memory events after completion via the same methods. The streaming path uniquely requires `agent.iter()` for progress events (frontend reasoning status). Forcing both through `iter()` would add complexity to the non-streaming path for no gain.

### F3. Make server a pure OpenAI-compat protocol adapter
**Status: IMPLEMENTED** — Extracted `_build_span_attrs()` helper and `_format_sse_chunk()` module-level function. Removed unused `ChatCompletionRequest` model and `BaseModel` import. Server is now ~660 lines with clean separation between protocol formatting and agent interaction.

### F4. Standardize `max_steps` on Pydantic AI `UsageLimits`
**Status: DEFERRED** — Current mapping `UsageLimits(request_limit=max_steps)` is already clean and simple. Exposing full UsageLimits configuration in AgentServerSettings would add configuration complexity with little user benefit in alpha. Can revisit when users need token-level limits.

### F5. Move memory to AgentDeps for full RunContext access
**Status: IMPLEMENTED** — Added `memory: Optional[Memory]` to `AgentDeps` dataclass. Delegation tools now receive memory via `ctx.deps.memory`. This decouples tools from the Agent instance and enables future custom tools to access memory through standard Pydantic AI dependency injection.

### F6. Remove KAOS `Agent` wrapper entirely
**Status: DEFERRED** — The wrapper provides clean encapsulation for model resolution, delegation registration, memory integration, progress events, and cleanup lifecycle. Converting to a factory function would scatter the same logic into module-level functions or closures without reducing total code. The wrapper is already thin (~640 lines including all functionality). Removing it would reduce readability and testability.

### F7. Replace `string_mode.py` with a proper Pydantic AI Model adapter
**Status: DEFERRED** — `FunctionModel(handler)` is a documented and legitimate Pydantic AI pattern for custom model behavior. A full Model subclass would require implementing a complex interface (multiple methods for streaming, token counting, etc.) for marginal benefit. Current handler is ~100 clean lines. Can revisit when Pydantic AI's Model interface stabilizes.

---

## Additional Simplifications (F-series round)

### 9. Add memory to AgentDeps for RunContext access (`272d4a1`)
- **Before**: Delegation tools accessed memory via `self.memory` closure
- **After**: `AgentDeps(session_id, memory)` passes memory through Pydantic AI's `RunContext`
- **Impact**: Decoupled tools from Agent instance, enabled future custom tool memory access

### 10. Extract shared span attrs and SSE formatting (`b444199`)
- **Before**: Both completion methods manually built identical span attributes and SSE formatting was inline
- **After**: `_build_span_attrs()` method and `_format_sse_chunk()` module function
- **Impact**: -21 lines, cleaner protocol adapter

### 11. Remove unused imports from create_agent_server (`d61584c`)
- Removed duplicate `import os` (already at module level) and unused `import re`

### 12. Extract _format_progress_event from streaming loop (`c6cbcfc`)
- **Before**: 15-line inline JSON construction for progress events in the streaming loop
- **After**: Dedicated `_format_progress_event()` method
- **Impact**: Streaming loop reduced from 18 to 8 lines

### 13. Add close() to Memory ABC (`ab7cd2d`)
- **Before**: `hasattr(self.memory, "close")` check in Agent.close()
- **After**: No-op `close()` on Memory ABC; all implementations have it

### 14. Remove unused ChatCompletionRequest model (`92704ba`)
- Removed unused Pydantic model and BaseModel import from server.py

### 15. Simplify AgentCard.to_dict with asdict (`174a424`)
- **Before**: Manual 6-line dict construction
- **After**: `dataclasses.asdict(self)` — 1 line

## Updated Summary

| Metric | Original | After M1-8 | After F-series | Delta |
|--------|----------|-----------|----------------|-------|
| client.py (lines) | ~635 | ~587 | ~635* | ~0 |
| server.py (lines) | ~703 | ~679 | ~657 | -46 |
| memory.py (lines) | ~682 | ~682 | ~686 | +4 (ABC close) |
| Total (5 core files) | ~2659 | ~2587 | ~2545 | -114 |
| Dead code classes/models | 2 | 0 | 0 | -2 |
| Duplicate imports | 3 | 0 | 0 | -3 |
| hasattr checks | 1 | 1 | 0 | -1 |

*client.py slightly increased due to new `_format_progress_event()` and `AgentDeps.memory` but offset by other reductions.
