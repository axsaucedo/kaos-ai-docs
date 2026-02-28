# COMMIT_FLOW_FULL.md — Narrative Commit Flow

This document groups the 93 relevant commits from the `feat/exploration-pydantic-ai` branch into logical "chapters" that tell the story of how the Pydantic AI Server was built. Each chapter preserves traceability to commit hashes while emphasizing the *why* behind design decisions, iterations, and reversals.

---

## Chapter 1: Foundation — Replacing the Agent Core with Pydantic AI

**Commits:** `e8e26a2`, `c544569`, `8f9b07c`, `fdbd11e`, `e103a36`, `1051944`, `3758b23`

The journey begins with a deliberate architectural decision: replace the custom-built agent framework with [Pydantic AI](https://ai.pydantic.dev) as the first-class runtime. The rationale, documented in research reports (`e8e26a2`), evaluated multiple frameworks — ADK (too tightly coupled to GCP/Vertex), LangChain/CrewAI (too complex), and Pydantic AI (best balance of structured validation, clean interfaces, and extensibility).

The core swap (`c544569`) replaced the entire agent execution engine. Where the old framework had a custom agentic loop with manual tool dispatch, Pydantic AI's `Agent` class takes over: model calls, tool invocation, and the agentic loop are all handled by the framework. KAOS adds the server shell — HTTP endpoints, memory persistence, delegation between agents, and operational infrastructure.

An immediate fix (`8f9b07c`) addressed Pydantic v2 settings compatibility — `model_config` needs to be a dict, not a class attribute. The Dockerfile was updated (`fdbd11e`) and the new framework was validated end-to-end in a KIND cluster (`e103a36`). Documentation and a final integration report followed (`1051944`, `3758b23`).

**Key design decision:** Pydantic AI owns the agentic loop. KAOS never reimplements it — it wraps, configures, and extends.

---

## Chapter 2: Hardening — Bug Fixes and OpenAI Compatibility

**Commits:** `5e8e4d8`, `a38c05d`, `e07128c`, `e8d141c`, `f98cf5f`, `bf7b1d4`

With the core swapped, real-world testing exposed friction points. The first critical fix (`5e8e4d8`) was OpenAI API compatibility: `MODEL_API_URL` needed `/v1` auto-appended for Ollama's OpenAI-compat endpoint. Mock response patterns were updated (`e07128c`) — Pydantic AI needs only 2 mock responses for tool calls (not 3 like the old framework), because it stops automatically after text follows tool execution.

A code review (`e8d141c`) addressed 7 bugs ranging from tool result serialization to memory event types. These are the kinds of integration bugs that only surface when running real multi-turn conversations with actual model backends.

**Lesson:** Framework migrations surface subtle behavioral differences — mock testing alone isn't enough; you need real model and E2E validation.

---

## Chapter 3: String-Mode Tool Calling — Supporting Limited Models

**Commits:** `43b45f7`, `29ca5f9`, `cac95b0`, `9023c9e`, `eca4abb`, `187dabe`, `cd78fdc`

Not all models support native function calling (tool_calls in the OpenAI API). For smaller models (e.g., Ollama's smollm2), PAIS introduces "string mode" (`43b45f7`): tool descriptions are injected into the system prompt, and the model's text response is parsed for JSON `{"tool_calls": [...]}` patterns. This uses Pydantic AI's `FunctionModel` as a wrapper that intercepts model calls and adapts the protocol.

This chapter also introduces custom agent image support (`cac95b0`) — users can package their own Pydantic AI agents with custom tools and deploy them via the Agent CRD's `container.image` override. The operator was refactored (`eca4abb`) to use strategic merge instead of cherry-picked container overrides, making the customization pattern cleaner.

**Key design decision:** String mode is a transport adapter, not a different execution model. The same DelegationToolset and MCP tools work regardless of whether the model uses native or string-mode calling.

---

## Chapter 4: Streaming and Progress Events

**Commits:** `3dfb890`, `dbecda0`, `4c004fd`

Streaming required a fix (`3dfb890`) — Pydantic AI's `run_stream` was replaced with `iter()` for proper SSE streaming support. Progress events were enriched with `step` and `max_steps` counters (`dbecda0`), giving clients visibility into the agentic loop's progress. A further refinement (`4c004fd`) distinguishes delegation progress events from tool_call events, so clients can render multi-agent orchestration differently from single-agent tool use.

**Design pattern:** PAIS wraps Pydantic AI's streaming output in OpenAI-compatible SSE format (`data: {...}\n\n`), adding KAOS-specific progress events as additional SSE messages during tool execution and delegation.

---

## Chapter 5: OpenTelemetry — From Custom Manager to Native Integration

**Commits:** `5944de2`, `381e625`, `b853ad2`, `5e8b7f8`, `82cdf1a`, `9f901a5`, `09c43e1`, `2df52dc`, `d4f1406`, `b8b720b`, `194b202`

This is the most iterative chapter — OpenTelemetry integration went through several architectural phases:

1. **Initial integration** (`5944de2`): Enabled Pydantic AI's built-in OTEL instrumentation via `Agent.instrument_all()`. This was a key discovery — Pydantic AI has first-party tracing support.

2. **Bug fix — delegation closure leak** (`b853ad2`): The delegation tool closure was accidentally exposing internal `ctx` and `_toolset` parameters to the model. This is a subtle but critical bug — the model sees these as required tool parameters and fails.

3. **Architecture overhaul** (`5e8b7f8`): Replaced the custom `KaosOtelManager` singleton with standard OTEL context managers (`tracer.start_as_current_span()`). This was a significant simplification — no more manual span lifecycle management.

4. **Span lifecycle fix** (`9f901a5`): The `server-run` span needed to be created inside `generate_stream()` (streaming) and `_complete_chat_completion()` (non-streaming) to stay active during processing. Creating it in the route handler caused it to close before streaming completed.

5. **Provider passing** (`09c43e1`): Explicit `TracerProvider` and `EventLoggerProvider` passed to Pydantic AI's `instrument_all()` instead of relying on global state.

6. **Instrumentation versioning** (`2df52dc`): `OTEL_INSTRUMENTATION_VERSION` (1-4) and `OTEL_EVENT_MODE` (attributes/logs) control Pydantic AI's instrumentation behavior. Version 1 + logs mode emits LogRecord events; version 2+ stores data as span attributes.

7. **Critical fix** (`d4f1406`): Do NOT pass `instrument=True` to the Agent constructor — it creates fresh defaults, ignoring `instrument_all()` settings.

**Key lesson:** Pydantic AI handles agent/model/tool spans internally. KAOS adds: delegation spans, `server-run` request span, `kaos.delegations` counter, `kaos.delegation.duration` histogram, and log correlation via `KaosLoggingHandler`.

**Reversal note:** An attempt to add per-tool OTEL spans (`6c07b4e`) was immediately reverted (`1b31b60`) — Pydantic AI already provides tool-level tracing, so custom spans were redundant and caused double-counting.

---

## Chapter 6: Deep Refactor — Server-Centric Architecture

**Commits:** `b4b99e4`, `c63ae22`, `7479044`, `5b02590`, `f40997e`, `3956fe0`, `47d3d20`, `c899c72`, `a012423`, `272d4a1`, `b444199`, `d61584c`, `c6cbcfc`, `ab7cd2d`, `92704ba`, `174a424`, `7c5ffea`, `39c12f4`, `25ea96d`, `75165db`, `85b5118`, `f529932`, `c894d52`, `8480a7c`, `8e7939e`

This is the most extensive chapter — a systematic refactoring that transforms the architecture from "Agent-centric" (where a wrapper `Agent` class mediated between server and Pydantic AI) to "Server-centric" (where `AgentServer` directly owns the Pydantic AI agent):

**Memory refactoring:**
- NullMemory replaces `memory_enabled` flag (`b4b99e4`) — no branching needed, just call memory methods unconditionally
- Memory ABC extracted with shared utilities (`c63ae22`) — `build_message_history()` and `store_pydantic_message()` convert between KAOS events and Pydantic AI messages
- `close()` added to Memory ABC (`ab7cd2d`)

**Dependency injection:**
- `_current_session_id` ContextVar replaced with Pydantic AI's `RunContext` deps (`7479044`)
- `AgentDeps(session_id, memory)` passed through RunContext — concurrency-safe per-request state
- Memory added to AgentDeps (`272d4a1`) for tool access

**Extraction and decomposition:**
- `DelegationToolset` extracted to `tools.py` (`25ea96d`) — implements Pydantic AI's `AbstractToolset` interface, exposing sub-agents as `delegate_to_{name}` tools dynamically
- Message history utilities moved to Memory base class (`75165db`)
- Model resolution, deps, and remote agent consolidated into `serverutils.py` (`85b5118`)
- `_process_message()` moved into AgentServer (`f529932`)

**The big removal** (`c894d52`): The `Agent` wrapper class and `client.py` are deleted entirely. `AgentServer` now directly owns the `pydantic_ai.Agent` instance. This was the culmination of all the preceding refactoring — each step made the wrapper thinner until it could be removed.

**Key design decision:** `AgentServer` is the one class. No wrappers, no intermediaries. It creates the Pydantic AI agent, configures tools via toolsets (DelegationToolset, MCPServerStreamableHTTP), manages memory, and serves HTTP endpoints.

---

## Chapter 7: Code Health — Decomposition and Cleanup

**Commits:** `47d33b2`, `87b79b8`, `2562504`, `c9517fc`, `8d32d37`, `f2fd396`, `b7e51a4`, `92a477a`, `3fd5473`, `35813c1`, `e75003a`, `bce94ec`, `626a177`, `18e2afc`, `8bae64a`, `db7381e`

A thorough cleanup pass that brings the codebase from "working refactor" to "maintainable production code":

- Dead code removed: `mock_model_server.py` (188 lines, `87b79b8`), verbose docstrings (-337 lines, `2562504`)
- `server.py` decomposed from 1107→754 lines (`c9517fc`) by extracting `models.py`
- `create_agent_server` decomposed into focused helper functions (`8d32d37`)
- Response builders, logging, and probe helpers extracted into small focused functions
- Telemetry folder flattened from directory to single file (`bce94ec`)
- Settings stored directly, removing 6 duplicate `__init__` fields (`626a177`)

**Theme:** Every function should do one thing. If `server.py` is >750 lines, extract. If a function builds responses AND manages spans, split them.

---

## Chapter 8: Naming and Identity — Birth of PAIS

**Commits:** `d70a83a`, `8f4f798`, `0ffa8d5`, `af3b48d`, `7ebd90b`, `1972016`, `7cea2e8`, `7a286d9`, `33eeccf`, `0f6c397`

The package undergoes two naming passes:
1. `agent/` → `kaos_server/`, `kaos-framework/` → `kaos-server/` (`d70a83a`)
2. `kaos-server/` → `pai-server/` (Pydantic AI Server) (`7a286d9`)

Along the way, internal modules are refined: `config.py` → `serverutils.py` (`8f4f798`), trace context wrappers inlined (`0ffa8d5`), SSE/response builders consolidated (`1972016`). The PAIS README is created (`33eeccf`), establishing the package as a standalone entity with its own identity.

**Design decision:** PAIS is the standalone server package. KAOS is the Kubernetes orchestration platform that deploys it. Clean separation of concerns.

---

## Chapter 9: A2A Discovery and Session Consolidation

**Commits:** `61437ef`, `5aa0914`, `df3359c`, `56af538`, `8f03caf`

The final feature work introduces A2A (Agent-to-Agent) protocol compliance:
- `chat_id` and `session_id` consolidated into a single identifier (`61437ef`) — the old distinction was legacy from the pre-Pydantic AI architecture
- A2A-compliant agent card format implemented (`5aa0914`) at `/.well-known/agent.json` — includes agent name, description, skills, capabilities (streaming, state transition history), and version
- CLI and examples updated to use the standard A2A path (`df3359c`, `56af538`, `8f03caf`)

**Key pattern:** The agent card enables zero-configuration agent discovery. Delegation targets advertise their capabilities, and orchestrating agents can inspect what tools and skills are available before delegating.

---

## Chapter 10: Final Polish and Pre-Merge Review

**Commits:** `264a22d`, `dcd579e`, `5b28a69`

Documentation updates and a comprehensive pre-merge review fixing pyproject.toml, Makefile targets, docs, and residual TODO comments. The branch is ready for merge.

---

## Summary: The Arc

The 93 commits tell a clear architectural story:

1. **Replace** the custom agent core with Pydantic AI (keep KAOS as the server shell)
2. **Harden** the integration with real-world bug fixes
3. **Extend** with string-mode tool calling, streaming progress events, and OTEL
4. **Refactor** from Agent-centric to Server-centric architecture (AgentServer owns everything)
5. **Clean** the codebase to production quality
6. **Name** and establish PAIS as a standalone package
7. **Add** A2A protocol compliance for agent interoperability

The thread through all of it: **Pydantic AI does the agent work. PAIS does the server work. KAOS does the Kubernetes work.** Three clear layers, each owning its domain.
