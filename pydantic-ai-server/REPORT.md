# KAOS Pydantic AI Integration — Comprehensive Report

## Overview

This report covers the complete refactoring of the KAOS data-plane Python framework from a custom implementation to a Pydantic AI-based architecture across three phases of development.

---

## Phase 1: Core Framework Rewrite (12 tasks + 7 bug fixes)

### Tasks Completed

| # | Task | Status | Commit |
|---|------|--------|--------|
| 1 | Pydantic AI Agent wrapper (`client.py`) | ✅ Done | `6f93d1e` |
| 2 | Mock model (FunctionModel + _MockResponseState) | ✅ Done | `6f93d1e` |
| 3 | MCP integration (MCPServerStreamableHTTP) | ✅ Done | `6f93d1e` |
| 4 | Sub-agent delegation (delegate_to_ tools) | ✅ Done | `6f93d1e` |
| 5 | Memory bridge (KAOS events ↔ Pydantic AI messages) | ✅ Done | `6f93d1e` |
| 6 | AgentServer (FastAPI endpoints) | ✅ Done | `6f93d1e` |
| 7 | Unit tests (75 tests) | ✅ Done | `6f93d1e` |
| 8 | E2E test updates (24 tests) | ✅ Done | Multiple |
| 9 | Telemetry (OpenTelemetry) | ✅ Done | `9ecaee5` |
| 10 | CI pipeline updates | ✅ Done | `f98cf5fc` |
| 11 | Copilot instructions | ✅ Done | `9ecaee5` |
| 12 | pyproject.toml + Dockerfile | ✅ Done | `6f93d1e` |

### Bug Fixes

| # | Bug | Status |
|---|-----|--------|
| B1 | Memory persistence for tool calls | ✅ Fixed |
| B2 | Streaming response format | ✅ Fixed |
| B3 | Agent card tool discovery | ✅ Fixed |
| B4 | Delegation context forwarding | ✅ Fixed |
| B5 | Mock response ContextVar issue | ✅ Fixed |
| B6 | E2E mock response patterns (2-entry vs 3) | ✅ Fixed |
| B7 | Memory endpoint session filtering | ✅ Fixed |

---

## Phase 2: Documentation (11 tasks)

| # | Task | Status | Commit |
|---|------|--------|--------|
| 1 | PR #88 description update | ✅ Done | — |
| 2 | VitePress overview.md | ✅ Done | `bf7b1d4` |
| 3 | VitePress agent.md | ✅ Done | `bf7b1d4` |
| 4 | VitePress agentic-loop.md | ✅ Done | `bf7b1d4` |
| 5 | VitePress memory.md | ✅ Done | `bf7b1d4` |
| 6 | VitePress mcp-tools.md | ✅ Done | `bf7b1d4` |
| 7 | VitePress server.md + delete model-api.md | ✅ Done | `bf7b1d4` |
| 8 | Copilot instructions (.github/instructions) | ✅ Done | Prior commits |
| 9 | ROADMAP.md | ✅ Done | `bf7b1d4` |
| 10 | Docs build validation | ✅ Done | `bf7b1d4` |
| 11 | REPORT.md | ✅ Done | — |

---

## Phase 3: String Mode, A2A Assessment, Custom Agent Image (6 tasks)

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| 1 | String-mode tool calling (`string_mode.py`) | ✅ Done | `43b45f7` | FunctionModel wrapper with system prompt injection |
| 2 | String-mode unit tests (23 tests) | ✅ Done | `43b45f7` | Tool descriptions, JSON parsing, agent integration |
| 3 | String-mode E2E test update | ✅ Done | `29ca5f9` | `toolCallMode: string` in Agent CRD |
| 4 | fasta2a A2A assessment | ✅ Assessed | — | **Deferred**: JSON-RPC protocol incompatible with OpenAI-compat |
| 5 | Custom agent image support | ✅ Done | `cac95b0` | Image override + custom tools + E2E test |
| 6 | Documentation & REPORT.md update | ✅ Done | Current | Instructions, VitePress, ROADMAP updated |

### fasta2a Assessment Summary

**Decision: Deferred**

fasta2a v0.1.0 is a Starlette-based application implementing the A2A JSON-RPC protocol. Assessment findings:
- Uses JSON-RPC protocol (not OpenAI-compatible) — fundamentally different from our current `/v1/chat/completions` endpoint
- Requires a separate `Worker` class with 4 abstract methods
- `streaming=False` in current capabilities
- Pre-release API (v0.1.0) — breaking changes likely
- Would require running dual apps (FastAPI + Starlette) or replacing the server
- Our custom A2A implementation works well with the OpenAI-compatible format

**Recommendation**: Wait for fasta2a maturity and consider integrating when the protocol stabilizes.

---

## Phase 4: Code Cleanup, Docs & Live Validation

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| 1 | Remove unused modelapi/ and mcptools/ | ✅ Done | `187dabe` | Deleted modules + stale logger names |
| 2 | Custom agent docs (docs/examples/custom-agent.md) | ✅ Done | `cd78fdc` | VitePress sidebar entry added |
| 3 | Strategic merge for container overrides | ✅ Done | `eca4abb` | Replaced cherry-picked overrides |
| 4 | CI fix for custom-agent image | ✅ Done | `698dd0b` | Docker build + KIND load in CI |
| 5 | String-mode live validation | ✅ Verified | — | Tested against DeepSeek V3 via port-forward |

### String-Mode Live Validation

Verified string-mode works against live DeepSeek V3 model (Nebius) in `kaos-hierarchy` namespace:
- Port-forwarded to ModelAPI and tested with local Pydantic AI code
- Echo MCP tool calls work correctly (tool_calls JSON parsed from response text)
- Models without native function calling (DeepSeek) correctly use string-mode when `toolCallMode: string`
- All hierarchy agents patched to string mode for DeepSeek compatibility

---

## Architecture Summary

### Before (Custom Framework)
- Custom two-phase agentic loop
- Manual tool calling with string-mode fallback
- Custom MCP client (`mcptools/`)
- Custom model API client (`modelapi/`)
- `litellm.supports_function_calling()` for model detection

### After (Pydantic AI)
- `pydantic_ai.Agent` handles agentic loop natively
- Native function calling via Pydantic AI tools API
- String-mode via `FunctionModel` wrapper for models without native support
- MCP via `MCPServerStreamableHTTP` (Pydantic AI built-in)
- Model via `OpenAIChatModel` + `OpenAIProvider`
- Custom agent images with `create_agent_server(custom_agent=...)`

### Key Metrics
- **101 unit tests** (75 original + 23 string-mode + 3 streaming)
- **25+ E2E tests** across 6 test files
- **55 of 59** roadmap features complete (93%)
- **4 features deferred** (fasta2a A2A × 2, otel utility, convenience wrappers)

---

## Commit History

| Commit | Message |
|--------|---------|
| `6f93d1e` | feat(framework): rewrite data-plane to Pydantic AI agent framework |
| `9ecaee5` | feat(framework): add telemetry, instructions, and polish |
| `f98cf5fc` | fix(e2e): update all E2E tests for Pydantic AI mock pattern |
| `bf7b1d4` | docs: update python framework docs for pydantic-ai architecture |
| `43b45f7` | feat(framework): add string-mode tool calling |
| `29ca5f9` | test(e2e): update string-mode E2E test to use toolCallMode: string |
| `cac95b0` | feat(agent): add custom agent image support |
| `eca4abb` | refactor(operator): replace cherry-picked container overrides with strategic merge |
| `698dd0b` | fix(ci): build and load custom-agent image for E2E tests |
| `187dabe` | refactor(framework): remove unused modelapi and mcptools modules |
| `cd78fdc` | docs: add custom agent image example to VitePress docs |
| `3dfb890` | fix(framework): replace run_stream with iter() for streaming support |
| `dbecda0` | feat(framework): add step and max_steps to streaming progress events |
| `4c004fd` | feat(framework): distinguish delegate vs tool_call in progress events |
| `6c07b4e` | feat(framework): add per-tool OTEL spans and unknown tool error detection |

---

## Phase 5: Streaming Fix & OTEL Enhancements

### Streaming Fix
**Problem**: `stream: true` requests to string-mode agents failed with `FunctionModel must receive a stream_function to support streamed requests`. Pydantic AI's `run_stream()` calls `node.stream()` for ALL model request nodes (including intermediate tool-calling steps), requiring `stream_function` on FunctionModel.

**Solution**: Replaced `run_stream()` with `agent.iter()` for streaming mode. This provides node-by-node control:
- **Progress events**: Tool calls emit `{"type":"progress","action":"tool_call","target":"..."}` SSE chunks
- **Step tracking**: `step` and `max_steps` fields match old two-phase behavior
- **Delegate detection**: `action` distinguishes `delegate` vs `tool_call` based on `delegate_to_` prefix
- **Final response**: Text yielded as single chunk after agentic loop completes
- **No stream_function needed**: `iter()` uses `function` (non-streaming) for all model calls

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| 1 | Replace run_stream with iter() | ✅ Done | `3dfb890` | Node-by-node streaming |
| 2 | Add step/max_steps to progress events | ✅ Done | `dbecda0` | Step counter per CallToolsNode |
| 3 | Distinguish delegate vs tool_call action | ✅ Done | `4c004fd` | DELEGATION_TOOL_PREFIX check |
| 4 | Per-tool OTEL spans + unknown tool detection | ✅ Done | `6c07b4e` | Interim until upstream ships |

### OTEL Per-Tool Spans
Added per-tool OTEL spans in the iter() streaming loop:
- `otel.span_begin(f"agent.tool_call.{tool_name}")` for each ToolCallPart
- `RetryPromptPart` detection → `otel.span_failure()` + warning log for unknown tools
- `ToolReturnPart` detection → `otel.span_success()` for successful tools
- Comments reference upstream pydantic-ai contribution (see REPORT-PYDANTIC-PR.md)

### Upstream Pydantic AI Contribution
- **Fork**: https://github.com/axsaucedo/pydantic-ai/tree/feat/per-tool-otel-spans
- **Change**: Added `tracer` param to `_call_tool()`, per-tool span with `gen_ai.*` attributes
- **Files**: `pydantic_ai_slim/pydantic_ai/_agent_graph.py` (~15 lines logic, rest re-indent)
- **Reports**: See REPORT-PYDANTIC-PR.md and REPORT-PYDANTIC-ISSUE.md (gitignored)

### Verification
- 101 unit tests pass (new progress, delegation, unknown tool tests)
- Live tested against KIND cluster with DeepSeek V3 (string-mode)
- Tool calls emit progress SSE with step tracking, final text follows correctly

---

## Phase 6: Pydantic AI OTEL Instrumentation

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| 1 | Enable Pydantic AI instrumentation | ✅ Done | `5944de2` | `instrument=True` + `Agent.instrument_all(True)` |
| 2 | Verify span correlation | ✅ Done | — | KAOS spans → Pydantic AI spans (same tracer provider) |
| 3 | Create PLAN-FULL-OTEL-REFACTOR.md | ✅ Done | — | 5-phase migration plan (gitignored) |
| 4 | Create REPORT-PYDANTIC-TELEMETRY-FULL.md | ✅ Done | — | Deep-dive guide for senior engineers (gitignored) |
| 5 | Update copilot instructions | ✅ Done | `381e625` | OTEL section in python.instructions.md |

### Key Finding: NoOpTracer Without instrument=True

Pydantic AI uses `NoOpTracer()` when `instrument` is not set — ALL internal spans (agent run, model call, per-tool) are silently disabled. After enabling `instrument=True`:

- **7 span types** now active: agent run, model call, running tools, per-tool, output tool, concurrency wait, graph run
- **Span correlation confirmed**: KAOS's `span_begin()` → `context.attach()` → Pydantic AI spans auto-parent
- **Metrics enabled**: token usage and cost histograms recorded by Pydantic AI
- **Redundancy identified**: KAOS per-tool spans in iter() overlap with Pydantic AI's `_tool_manager` per-tool spans

### Future Migration (PLAN-FULL-OTEL-REFACTOR.md) — ✅ EXECUTED (Phases 1-3)

Phases 1-3 of the 5-phase plan executed. KaosOtelManager removed (~625 → ~175 LOC):

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | Remove agent.agentic_loop span (Pydantic AI covers it) | ✅ Done |
| 2 | Replace delegation spans with `tracer.start_as_current_span()` | ✅ Done |
| 3 | Simplify server context propagation (no manual attach/detach) | ✅ Done |
| 4 | Extract SDK init to minimal bootstrap (already minimal) | Deferred |
| 5 | Simplify metrics (delegation metrics only remain) | ✅ Done |

**Commits**: `5e8b7f8` (OTEL refactor), `82cdf1a` (docs update)

**Bugs Fixed**:
- Context detach ValueError ("token was created in a different Context") — caused by `otel_context.attach/detach` conflicting with `asyncio.create_task()` in Pydantic AI
- Delegation closure bug (tool schema exposed `_s`/`_n` params) — fixed in `b853ad2`
- No Pydantic AI log correlation — confirmed as expected behavior (Pydantic AI uses OTEL Logger API, not Python logging)

**Observability Report**: `REPORT-PYDANTIC-OBSERVABILITY-RECOMMENDATIONS.md` (gitignored) — 6 recommendations for further improvements

---

## Phase 7: Framework Simplification (15 tasks across 2 rounds)

### Round 1: Targeted Simplifications (M1-M3, #3-#8)

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| M1 | Remove `memory_enabled` flag, use NullMemory | ✅ Done | `b4b99e4` | -16 lines, eliminated 7 conditional checks |
| M2+M3 | Memory ABC with shared logic | ✅ Done | `c63ae22` | -44 lines, single create_event/build_conversation_context |
| #3 | Replace `_current_session_id` with RunContext deps | ✅ Done | `7479044` | Concurrency-safe delegation |
| #4 | Extract `_resolve_model()` factory function | ✅ Done | `5b02590` | Clean separation of model resolution |
| #5 | Remove unused imports | ✅ Done | `f40997e` | client.py + server.py cleanup |
| #6 | Consolidate RemoteAgent HTTP clients | ✅ Done | `3956fe0` | 2 clients → 1 client |
| #7 | Remove unused OTEL constants | ✅ Done | `47d3d20` | ATTR_AGENT_NAME, ATTR_SESSION_ID |
| #8 | Compact startup logging | ✅ Done | `c899c72` | 46 lines → 5 structured lines |

### Round 2: F-Series Radical Simplifications

**Evaluated**: F1-F7 proposals were thoroughly researched and assessed against actual Pydantic AI APIs.

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| F5 | Add memory to AgentDeps | ✅ Done | `272d4a1` | Tools access memory via ctx.deps.memory |
| F3 | Extract shared span/SSE helpers | ✅ Done | `b444199` | _build_span_attrs() + _format_sse_chunk() |
| — | Remove duplicate imports | ✅ Done | `d61584c` | os/re in create_agent_server |
| — | Extract _format_progress_event | ✅ Done | `c6cbcfc` | Streaming loop 18→8 lines |
| — | Add close() to Memory ABC | ✅ Done | `ab7cd2d` | Eliminated hasattr check |
| — | Remove dead ChatCompletionRequest | ✅ Done | `92704ba` | Unused model + BaseModel import |
| — | Simplify AgentCard.to_dict | ✅ Done | `174a424` | asdict() replaces manual dict |
| F1 | History processors | Deferred | — | history_processors can't replace KAOS→Pydantic conversion |
| F2 | Unify streaming paths | Deferred | — | Minimal duplication; streaming needs iter() for progress |
| F4 | UsageLimits config | Deferred | — | Current mapping already clean |
| F6 | Remove Agent wrapper | Deferred | — | Wrapper provides clean encapsulation |
| F7 | Model adapter for string mode | Deferred | — | FunctionModel is the correct Pydantic AI pattern |

### Cumulative Impact

| Metric | Original (pre-Phase 7) | After All Simplifications |
|--------|------------------------|--------------------------|
| Total lines (5 core files) | ~2659 | ~2545 |
| Conditional branches (memory) | 7 | 0 |
| Duplicate imports/dead code | 5+ | 0 |
| Mutable per-Agent state | 1 | 0 |
| httpx clients (RemoteAgent) | 2 | 1 |

---

## Deferred Items (for follow-up PRs)

| Feature | Reason |
|---------|--------|
| fasta2a A2A integration (R4.2-R4.3) | Pre-release API, JSON-RPC incompatible with OpenAI-compat |
| `otel.enable()` utility (R7.5) | Part of custom image convenience layer |
| Utility functions (R9.3) | `kaos.enable_otel()`, `kaos.enable_memory()` wrappers |
| String-mode upstream contribution | Contribute text-based tool calling to Pydantic AI |
| OTEL per-tool spans upstream | Contribute per-tool OTEL spans to Pydantic AI (fork ready) |
| F1: History processors | history_processors API can't replace KAOS→Pydantic conversion |
| F2: Streaming unification | Minimal duplication; streaming needs iter() for progress |
| F4: Full UsageLimits config | Current max_steps→request_limit mapping is clean |
| F6: Remove Agent wrapper | Wrapper provides clean encapsulation (~640 lines) |
| F7: String mode Model adapter | FunctionModel is the correct documented pattern |

---

## Pydantic AI Limitations

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| No string-mode tool calling | Models without native tools can't use tools | Custom `FunctionModel` wrapper (`string_mode.py`) |
| No built-in memory persistence | No cross-session conversation memory | KAOS memory bridge (LocalMemory/RedisMemory) |
| `FunctionModel` runs in copied context | `ContextVar` state doesn't persist across calls | `_MockResponseState` closure pattern |
| fasta2a pre-release | A2A protocol may change | Custom A2A on top of OpenAI-compat endpoints |
| No A2A streaming in fasta2a | `streaming=False` in capabilities | Custom streaming via SSE |
| `run_stream()` requires `stream_function` | FunctionModel can't use `run_stream()` without it | Use `iter()` for streaming with progress events |
| No Python logging output | Pydantic AI uses OTEL Logger API, not `logging` module | KAOS logs are correlated; Pydantic AI data is in span attributes |

---

## Phase 8: OTEL Settings, Startup Logging, CI & Research

### Configurable OTEL Instrumentation

Moved `OTEL_INSTRUMENTATION_VERSION` and `OTEL_EVENT_MODE` into `AgentServerSettings` as proper typed fields with defaults and validation:
- `otel_instrumentation_version: int = 4` (1-4, v4 = latest with multimodal support)
- `otel_event_mode: Literal["attributes", "logs"] = "attributes"` (only relevant for v1)

Fixed critical bug: `instrument=True` in Agent constructor overrode `instrument_all()` settings. Removed explicit `instrument=True` so agents fall back to `_instrument_default` (the custom `InstrumentationSettings` from `instrument_all()`).

### Tasks Completed

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| 1 | Startup logging (INFO + DEBUG) | ✅ Done | `39c12f4` | Compact INFO summary + DEBUG settings dump |
| 2 | CI custom-agent image optimization | ✅ Done | `bca36e4` | Only build for `core` E2E shard |
| 3 | Remove unused fasta2a dependency | ✅ Done | `b35e27d` | Zero imports in codebase |
| 4 | REPORT-PYDANTIC-SERVER.md | ✅ Done | — | Thorough analysis of to_a2a() / fasta2a (gitignored) |
| 5 | REPORT-PYDANTIC-AGENT.md | ✅ Done | — | Thorough analysis of Pydantic AI agent internals (gitignored) |
| 6 | PLAN-REFACTOR-AGENT.md | ✅ Done | — | Proposal: restructure KAOS with Pydantic AI as core (gitignored) |
| 7 | REPORT.md update | ✅ Done | — | This section |

### Startup Logging Improvement

Rewrote `_log_startup_config()` in `server.py`:
- **INFO level**: Single compact line with agent name, port, model, memory type, max_steps, MCP server count, sub-agent names. Separate OTEL status line.
- **DEBUG level**: Full `AgentServerSettings.model_dump()`, per-sub-agent status/description, per-MCP-server details, access_log setting.

### CI Custom-Agent Image Optimization

The custom-agent Docker image was being built for ALL 7 E2E shards but is only used by the `core` shard (`test_custom_agent_image` in `test_base_func_e2e.py`). Added `if: matrix.shard.name == 'core'` condition to both the build and KIND load steps, saving ~30 seconds per non-core shard.

### fasta2a Dependency Removal

`fasta2a>=0.1.0` was declared in pyproject.toml but had zero imports anywhere in the codebase. Removed cleanly. If A2A via `to_a2a()` is needed in the future, it can be re-added as an optional dependency.

### Research Reports (gitignored)

Three research documents created for the next phase of KAOS evolution:

1. **REPORT-PYDANTIC-SERVER.md** (~414 lines): Analysis of Pydantic AI's server implementation:
   - `to_a2a()` architecture (FastA2A, AgentWorker, Storage, Broker)
   - Comparison with KAOS AgentServer (11 feature dimensions)
   - Three adoption options: replace (not viable), add alongside, keep current
   - Upstream contribution opportunities

2. **REPORT-PYDANTIC-AGENT.md** (~535 lines): Analysis of Pydantic AI agent internals:
   - Core classes (AbstractAgent, Agent, graph-based execution)
   - Tool registration, RunContext[Deps], message history
   - 5 run methods (run, run_sync, run_stream, run_stream_events, iter)
   - What KAOS Agent adds vs what Pydantic AI provides natively

3. **PLAN-REFACTOR-AGENT.md**: Comprehensive proposal for restructuring KAOS:
   - Vision: Pydantic AI as first-class, KAOS as composable utilities
   - 8 utility modules: models, delegation, memory_utils, otel, deps, agent_card, server, factory
   - Server-centric process flow (server calls agent directly, no wrapper)
   - Custom agent pattern (bring-your-own pydantic_ai.Agent)
   - 4-phase migration path (extract → simplify server → custom agents → advanced memory)
   - Estimated 39% code reduction (~1304 → ~795 lines)
   - Risks, mitigations, and deferred decisions

---

## Phase 9: Agent Wrapper Removal — Pydantic AI Native Refactor

This phase executed the plan from PLAN-REFACTOR-AGENT.md to make Pydantic AI the core agent component. The `Agent` wrapper class in `client.py` was deleted, and all its functionality was restructured into composable modules with `AgentServer` as the single entry point.

### Module Restructure

| Before | After |
|--------|-------|
| `client.py` (634 lines): Agent wrapper, AgentDeps, AgentCard, RemoteAgent, model resolution | **Deleted** |
| `string_mode.py`: Separate module | **Merged** into `tools.py` |
| `server.py`: Thin HTTP adapter | `server.py`: Central orchestrator (AgentServer, create_agent_server, routes, model resolution, RemoteAgent, process_message) |
| — | `tools.py`: DelegationToolset (AbstractToolset), string-mode handler, progress events |
| `memory.py`: Memory ABC + backends | `memory.py`: Same + `build_message_history()` / `store_pydantic_message()` utilities on base class |

### Tasks Completed

| # | Task | Status | Commit | Notes |
|---|------|--------|--------|-------|
| 1 | Create tools.py — DelegationToolset + string mode | ✅ Done | `25ea96d` | AbstractToolset subclass, 35 lines of delegation logic |
| 2 | Add build_message_history + store_pydantic_message to Memory | ✅ Done | `75165db` | Concrete methods on Memory ABC, all backends inherit |
| 3 | Move model resolution + AgentDeps + RemoteAgent to server.py | ✅ Done | `85b5118` | server.py owns all construction concerns |
| 4 | Move process_message into AgentServer | ✅ Done | `f529932` | Server calls pydantic_ai.Agent directly |
| 5 | Delete Agent wrapper — remove client.py | ✅ Done | `c894d52` | 381 insertions, 624 deletions |
| 6 | Update docs and instructions | ✅ Done | — | This commit |

### Key Design Decisions

1. **DelegationToolset over @agent.tool**: Uses `AbstractToolset[AgentDeps]` (same pattern as MCPServerStreamableHTTP) — dynamic per-run filtering, clean lifecycle, no agent mutation
2. **Memory utilities on base class**: `build_message_history()` and `store_pydantic_message()` as concrete methods on Memory ABC — all backends inherit, NullMemory no-ops naturally
3. **Server owns everything**: AgentServer is the single orchestrator. `create_agent_server()` builds PydanticAgent directly with toolsets, model, memory
4. **Custom agents**: `create_agent_server(custom_agent=my_agent)` overrides model and adds DelegationToolset to user's PydanticAgent
5. **Tests via make_test_server()**: `tests/helpers.py` provides factory that creates AgentServer with optional delegation, mock model, memory

### Impact

| Metric | Before | After |
|--------|--------|-------|
| Core files | 4 (client.py, server.py, string_mode.py, memory.py) | 3 (server.py, tools.py, memory.py) |
| Agent wrapper class | 634 lines | 0 (deleted) |
| Unit tests | 96 passing | 96 passing |
| Lint | Clean | Clean |

---

## Overall Project Status

### Metrics Summary

| Metric | Value |
|--------|-------|
| Total unit tests | 96 passing, 10 skipped |
| E2E tests | 25+ across 6 test files |
| Roadmap completion | 55 of 59 features (93%) |
| Core files | 3 (server.py, tools.py, memory.py) |
| PR | #88 on `feat/exploration-pydantic-ai` |
| Reports created | 7 (RESEARCH, ROADMAP, REPORT, 2× Pydantic analysis, SIMPLIFICATIONS, OTEL-RECS) |
| Plans created | 2 (PLAN-FULL-OTEL-REFACTOR, PLAN-REFACTOR-AGENT) |

### Deferred to Follow-Up

| Feature | Reason | Reference |
|---------|--------|-----------|
| A2A via `to_a2a()` | JSON-RPC incompatible with OpenAI-compat | REPORT-PYDANTIC-SERVER.md |
| Full Pydantic AI message serialization in memory | Higher fidelity but complex | PLAN-REFACTOR-AGENT.md Phase 4 |
| Proper Model adapter for string mode | Wait for Pydantic AI API stability | F7 in REPORT-SIMPLIFICATIONS |
| Upstream contributions | Health probes, OTEL middleware for fasta2a | REPORT-PYDANTIC-SERVER.md §6 |

---

## Phase 10: E2E Delegation Fix

### Root Cause
`DelegationToolset.get_tools()` filtered sub-agents by `remote._active`, but `RemoteAgent._active` starts as `False` and only becomes `True` after lazy initialization in `process_message()`. This prevented delegation tools from being exposed to the model in CI E2E tests.

### Fix
Removed the `_active` filter from `get_tools()`. Delegation tools are now always exposed — `RemoteAgent` handles lazy init on first call.

| Commit | Description |
|--------|-------------|
| `8e7939e` | `fix(delegation): remove _active filter from DelegationToolset.get_tools()` |

### CI Results (all passing)
- ✅ Python Tests (96 pass)
- ✅ Go Tests
- ✅ Docs Lint
- ✅ E2E Tests (all 3 suites: core, mcp, multi-agent)

---

## Phase 11: Custom Agent Test Migration + Complexity Analysis

### Custom Agent Test Migration (T1-T3)
Moved `test_custom_agent_image` from the core E2E test file to a Jupytext-executable example format, matching the existing example test patterns (kaos-monkey, custom-mcp-server).

| Commit | Description |
|--------|-------------|
| `47d33b2` | `refactor(e2e): move custom agent test to examples shard` |

Changes:
- Converted `docs/examples/custom-agent.md` to Jupytext-executable format with `%%writefile`, Python subprocess for docker build (graceful CI failure), Python polling for agent readiness
- Fixed mock response YAML quoting: replaced `%%bash` heredoc with Python `json.dumps()` to ensure proper JSON escaping in YAML env vars (root cause of missing `tool_call`/`tool_result` memory events)
- Added `test_custom_agent_example` to `test_examples_e2e.py`
- Added `example-custom-agent` shard to `reusable-tests.yaml` with Docker build/load
- Removed `test_custom_agent_image` from `test_base_func_e2e.py` (~93 lines)
- Moved Docker build from `core` shard to dedicated `example-custom-agent` shard

**CI Status**: All 13 checks passing ✅ (commit `47d33b2`)

### Complexity Analysis (T4)
Created `PLAN-IMPLEMENTATION-SIMPLIFICATION-AND-THOROUGH-ANALYSIS.md` (gitignored, local reference) with:
- Commit-by-commit audit of Phase 9 changes with line counts
- Module size comparison across 3 snapshots (main, pre-Phase9, HEAD)
- Assessment of each architectural decision (DelegationToolset ✅, Memory ABC ✅, Agent deletion ⚠️ mixed)
- 4 options for splitting server.py with pros/cons (recommended: Option 4)
- 6 further simplification opportunities identified
- Retrospective assessment: Agent wrapper removal was correct, server.py needs splitting

**Key finding**: Total agent module size unchanged (2240→2241 lines). server.py bloated to 1107 lines (+61%) but this is a file organization issue, not an architectural one. Recommendation: extract `models.py` and decompose `create_agent_server` factory.

---

## Phase 12: Server.py Decomposition + Codebase Simplification

### Task Summary

| # | Task | Status | Commit | Lines Impact |
|---|------|--------|--------|--------------|
| T9 | Remove dead `mock_model_server.py` | ✅ Done | `87b79b8` | -188 lines |
| T4 | Trim verbose docstrings | ✅ Done | `2562504` | -337 lines |
| T1 | Extract `models.py` from server.py | ✅ Done | `c9517fc` | 286 lines moved, server 1107→754 |
| T2 | Decompose `create_agent_server` into helpers | ✅ Done | `8d32d37` | Factory 174→50 lines readable |
| T3 | Simplify `configure_logging` | ✅ Done | `f2fd396` | 52→38 lines |
| T5 | Extract `_prepare_run()` from `_process_message` | ✅ Done | `b7e51a4` | DRY pre-run setup |
| T6 | Fix `_get_agent_card` private API access | ✅ Done | `92a477a` | Remove `_function_toolset` access |
| T7 | Simplify `_log_startup_config` | ✅ Done | `3fd5473` | 34→13 lines |
| T8 | Flatten `_setup_telemetry` | ✅ Done | `35813c1` | 30→18 lines |
| T10 | Create PLAN-SERVER-APP-STATE-REFACTOR.md | ✅ Done | N/A (gitignored) | Design document |

### Results

**server.py**: 1107 → 730 lines (34% reduction)
**New models.py**: 286 lines (clean data/config module)
**Dead code removed**: 188 lines (mock_model_server.py)
**Docstrings trimmed**: ~337 lines across package

Key architectural improvements:
- Broke bidirectional server↔tools dependency (tools.py now imports from models.py)
- Factory decomposed into 4 focused helpers (parse_mcp, parse_sub_agents, create_memory, setup_otel)
- Removed access to Pydantic AI internals (`_function_toolset`) — custom tools tracked at creation time
- Shared pre-run setup extracted into `_prepare_run()` method

**CI Status**: All tests passing ✅ (96 pass, 10 skipped). Pushed to PR #88.

---

## Phase 13: Further Simplification + Package Rename

### Tasks Completed

| # | Task | Status | Commit | Lines Impact |
|---|------|--------|--------|--------------|
| T7 | Rename `models.py` → `config.py` | ✅ Done | `e75003a` | Clearer module name |
| T8 | Flatten `telemetry/` → `agent/telemetry.py` | ✅ Done | `bce94ec` | -1 file, -1 folder |
| T1 | Store settings directly, remove 6 dup fields | ✅ Done | `626a177` | -12 lines init, simplified factory |
| T2 | Extract `_build_chat_response` helper | ✅ Done | `18e2afc` | DRY response dict |
| T3 | Combine health/ready into `_probe_response` | ✅ Done | `8bae64a` | -7 lines |
| T4 | Simplify `_get_agent_card` with comprehensions | ✅ Done | `db7381e` | -7 lines |
| T9 | Rename `agent/` → `kaos_server/`, `kaos-framework/` → `kaos-server/` | ✅ Done | `d70a83a` | 40 files, coherent naming |
| Fix | Regenerate notebook after rename | ✅ Done | `0999393` | Sync custom-agent.ipynb |

### Results

**server.py**: 730 → 704 lines (further 4% reduction, 36% total from 1107)
**Package rename**: `data-plane/kaos-framework/agent/` → `data-plane/kaos-server/kaos_server/`
**Module rename**: `models.py` → `config.py`, `telemetry/manager.py` → `kaos_server/telemetry.py`

Key improvements:
- `AgentServer.__init__` takes `AgentServerSettings` directly — no field duplication
- Telemetry is now a single file inside the package (no separate folder)
- Module names reflect content: `config.py` for configuration/helpers
- Package name `kaos_server` is descriptive and consistent
- All Dockerfiles, workflows, docs, examples, instructions updated

**CI Status**: Python ✅, Go ✅, E2E ✅, Docs Lint ✅ (after notebook regen). PR #88.

---

## Phase 14: pai-server Rename + Telemetry/SSE Simplifications

### Tasks Completed

| # | Task | Status | Commit | Impact |
|---|------|--------|--------|--------|
| T1 | Rename `config.py` → `serverutils.py` | ✅ Done | `8f4f798` | Clearer module naming |
| T2 | Inline `inject/extract_trace_context` wrappers | ✅ Done | `0ffa8d5` | -13 lines, use OTel directly |
| T3 | Replace `get_tracer()`/`_get_service_name()` with `SERVICE_NAME` global | ✅ Done | `af3b48d` | -5 lines, no per-call env reads |
| T4 | Convert `_resolve_model` to keyword-only params | ✅ Done | `7ebd90b` | Self-documenting call sites |
| T5 | Consolidate `_format_sse_chunk` → `_build_streaming_chunk`, move response builders to serverutils | ✅ Done | `1972016` | -7 lines, unified response building |
| T6 | Add `chat_id` vs `session_id` clarifying comments | ✅ Done | `7cea2e8` | Documentation only |
| T7 | Rename `kaos-server` → `pai-server`, `kaos_server` → `pai_server` | ✅ Done | `7a286d9` | 40 files, full package rename |
| T8 | Create `pai-server/README.md` (PAIS branding) | ✅ Done | `33eeccf` | Standalone package docs |
| T9 | Fix `ty check` diagnostics | ✅ Done | N/A (already passing) | 0 issues |
| Fix | Regenerate notebook after rename | ✅ Done | `8ca87dc` | Sync custom-agent.ipynb |

### Results

**Package rename**: `data-plane/kaos-server/kaos_server/` → `data-plane/pai-server/pai_server/`
**pyproject.toml**: `kaos-runtime` → `pai-server`
**Branding**: PAIS (Pydantic AI Server) with standalone README

Key improvements:
- Telemetry simplified: `SERVICE_NAME` global replaces per-call `os.getenv()` + wrapper functions
- `inject_trace_context`/`extract_trace_context` wrappers removed — use `opentelemetry.propagate` directly
- `_build_streaming_chunk` handles both content and final stop chunks (was 2 separate code paths)
- `_build_chat_response` moved to `serverutils.py` alongside streaming builder
- `_resolve_model` uses keyword-only params — self-documenting at call sites
- All response building logic consolidated in `serverutils.py`

**CI Status**: Python ✅, ty check ✅, Docker image verified ✅. PR #88.

*Report updated for PR #88 on branch `feat/exploration-pydantic-ai`*
