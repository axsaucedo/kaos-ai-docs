# KAOS Pydantic AI Integration — Feature Roadmap

This roadmap lists **every feature** required for the Pydantic AI integration to be feature-complete (parity with the current custom framework, plus new capabilities).

---

## Feature Status Legend

- ✅ **Done** — Implemented and tested
- 🔵 **Deferred** — Planned for follow-up
- 🔴 **Not started** — Must be built

---

## R1. Core Agent Framework

| ID   | Feature                                | Status | Notes                                                                                          |
| ---- | -------------------------------------- | ------ | ---------------------------------------------------------------------------------------------- |
| R1.1 | Pydantic AI Agent as core runtime      | ✅     | `Agent` wraps `pydantic_ai.Agent` in `client.py`                                              |
| R1.2 | Env-var driven agent creation          | ✅     | `AgentServerSettings` reads all env vars                                                       |
| R1.3 | OpenAI-compatible model provider       | ✅     | `OpenAIChatModel(provider=OpenAIProvider(base_url=...))`                                       |
| R1.4 | Tool calling (native function calling) | ✅     | Pydantic AI handles natively                                                                   |
| R1.5 | String-mode tool calling fallback      | ✅     | `FunctionModel` wrapper in `string_mode.py` — injects tool descriptions into system prompt |
| R1.6 | Max steps / loop control               | ✅     | `UsageLimits(request_limit=max_steps)`                                                         |
| R1.7 | System prompt configuration            | ✅     | `Agent(system_prompt=...)`                                                                     |

## R2. MCP Tool Integration

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R2.1 | MCP server connection via Streamable HTTP | ✅ | `MCPServerStreamableHTTP(url=...)` |
| R2.2 | Multi-MCP server support | ✅ | Passed as `toolsets` to Pydantic AI |
| R2.3 | Tool discovery from MCP servers | ✅ | Automatic via Pydantic AI |
| R2.4 | Graceful degradation on MCP failure | ✅ | Error handling in agent card discovery |
| R2.5 | Env-var MCP server configuration | ✅ | `MCP_SERVERS` + `MCP_SERVER_*_URL` env vars |

## R3. Sub-Agent Delegation

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R3.1 | Peer agent discovery via agent cards | ✅ | `PEER_AGENTS` + `PEER_AGENT_*_CARD_URL` |
| R3.2 | Delegation as tool functions | ✅ | `delegate_to_{name}` registered as `@tool_plain` |
| R3.3 | A2A-compatible delegation (send/receive tasks) | ✅ | Custom OpenAI-compat A2A (not fasta2a) |
| R3.4 | Unavailable agent exclusion | ✅ | Failed discovery → excluded from tools |

## R4. A2A Protocol

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R4.1 | A2A discovery endpoint (`/.well-known/agent`) | ✅ | Custom implementation on FastAPI |
| R4.2 | A2A task execution (fasta2a tasks/send) | 🔵 | Deferred — current custom A2A is sufficient |
| R4.3 | A2A streaming (fasta2a tasks/sendSubscribe) | 🔵 | Deferred — current custom A2A is sufficient |
| R4.4 | Backward-compatible `/v1/chat/completions` | ✅ | Full OpenAI-compatible streaming + non-streaming |

## R5. HTTP Server / API Surface

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R5.1 | FastAPI/ASGI server | ✅ | FastAPI with lifespan |
| R5.2 | Health endpoint (`GET /health`) | ✅ | Kubernetes liveness probe |
| R5.3 | Readiness endpoint (`GET /ready`) | ✅ | Kubernetes readiness probe |
| R5.4 | `/v1/chat/completions` (streaming + non-streaming) | ✅ | SSE streaming, OpenAI format |
| R5.5 | `GET /memory/events` | ✅ | With session_id and limit filters |
| R5.6 | `GET /memory/sessions` | ✅ | List all session IDs |
| R5.7 | `GET /.well-known/agent` | ✅ | A2A agent card |
| R5.8 | Session ID from header/body | ✅ | `X-Session-ID` header or body field |

## R6. Memory System

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R6.1 | Memory event model (MemoryEvent, SessionMemory) | ✅ | Unchanged from previous |
| R6.2 | LocalMemory (in-process, bounded sessions) | ✅ | With Pydantic AI bridge |
| R6.3 | RedisMemory (distributed, TTL, session index) | ✅ | With Pydantic AI bridge |
| R6.4 | NullMemory (stateless) | ✅ | No-op, used when `memory_enabled=False` |
| R6.5 | Message history bridging (Pydantic AI ↔ KAOS events) | ✅ | `_memory_events_to_messages()` + `_extract_and_persist_events()` |
| R6.6 | Memory HTTP endpoints | ✅ | `/memory/events` and `/memory/sessions` |
| R6.7 | Memory env var configuration | ✅ | `MEMORY_*` env vars |

## R7. Telemetry / Observability

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R7.1 | OTel tracing (spans for agent runs, tool calls) | ✅ | Custom spans + Pydantic AI native |
| R7.2 | OTel metrics (request count, duration, tool usage) | ✅ | Custom metrics in telemetry module |
| R7.3 | W3C Trace Context propagation | ✅ | Via OpenTelemetry SDK |
| R7.4 | OTLP export configuration | ✅ | `OTEL_*` env vars |
| R7.5 | User-facing `otel.enable()` utility | 🔵 | Deferred — part of custom image support |

## R8. Testing Infrastructure

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R8.1 | Mock model for unit tests (FunctionModel) | ✅ | `FunctionModel` + `_MockResponseState` |
| R8.2 | DEBUG_MOCK_RESPONSES env var support for E2E | ✅ | 2-entry pattern (tool call + final) |
| R8.3 | Unit tests for agent creation/configuration | ✅ | 75 tests passing |
| R8.4 | Unit tests for agentic loop (tool calling, delegation) | ✅ | Tool + delegation tests |
| R8.5 | Unit tests for memory bridge | ✅ | Memory bridge tests |
| R8.6 | Unit tests for server endpoints | ✅ | Server endpoint tests |
| R8.7 | E2E test compatibility | ✅ | All 7 E2E shards green |

## R9. Custom Image Support

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R9.1 | Standard wrapper mode (KAOS wraps user's Pydantic AI agent) | ✅ | `create_agent_server(custom_agent=...)` |
| R9.2 | Template mode (user builds from KAOS base) | ✅ | `examples/custom-agent/Dockerfile` |
| R9.3 | Utility functions (`kaos.enable_otel()`, `kaos.enable_memory()`, `kaos.serve()`) | 🔵 | Separate PR — convenience wrappers |
| R9.4 | Agent CRD custom image support | ✅ | `spec.container.image` override + controller support |

## R10. Build / Packaging

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R10.1 | Updated pyproject.toml with Pydantic AI dependencies | ✅ | `pydantic-ai-slim[openai,mcp]` |
| R10.2 | Updated Dockerfile | ✅ | Updated CMD, dependencies |
| R10.3 | Updated Makefile (lint, test, format) | ✅ | black + ty check |
| R10.4 | CI pipeline compatibility | ✅ | All CI green |

## R11. Documentation

| ID | Feature | Status | Notes |
|----|---------|--------|-------|
| R11.1 | Update `.github/instructions/python.instructions.md` | ✅ | Pydantic AI architecture |
| R11.2 | Update `.github/instructions/e2e.instructions.md` | ✅ | 2-entry mock pattern |
| R11.3 | Update `.github/copilot-instructions.md` | ✅ | Updated project structure |
| R11.4 | Update docs/ (VitePress) | ✅ | All pages rewritten |

---

## Feature Count Summary

| Category | Total | Done (✅) | Deferred (🔵) |
|----------|-------|-----------|---------------|
| Core Agent | 7 | 7 | 0 |
| MCP Tools | 5 | 5 | 0 |
| Delegation | 4 | 4 | 0 |
| A2A Protocol | 4 | 2 | 2 |
| HTTP Server | 8 | 8 | 0 |
| Memory | 7 | 7 | 0 |
| Telemetry | 5 | 4 | 1 |
| Testing | 7 | 7 | 0 |
| Custom Images | 4 | 3 | 1 |
| Build/Packaging | 4 | 4 | 0 |
| Documentation | 4 | 4 | 0 |
| **TOTAL** | **59** | **55** | **4** |

---

## Deferred Features

| Feature | Reason | Upstream Opportunity |
|---------|--------|---------------------|
| R4.2-R4.3 fasta2a A2A | Current custom A2A works well; fasta2a v0.1.0 is pre-release with JSON-RPC protocol | Wait for fasta2a maturity |
| R7.5 otel.enable() utility | Part of custom image support | — |
| R9.3 Utility functions | Convenience wrappers for custom images | Separate PR |

*Roadmap version: 3.0*
*Based on KAOS v0.2.8-dev, Pydantic AI v0.2.x*
