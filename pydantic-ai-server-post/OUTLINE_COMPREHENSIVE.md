# OUTLINE_COMPREHENSIVE.md — Blog Post Outline

## Meta

- **Title (working):** "Building PAIS: A Production Server for Pydantic AI Agents on Kubernetes"
- **Subtitle:** How we replaced a custom agent framework with Pydantic AI and built a thin server wrapper for Kubernetes-native AI agent orchestration
- **Target audience:** Application developers building agentic AI systems who want practical production patterns, not just demos
- **Reading time:** ~20 minutes
- **Tone:** Technical-practical, story-driven (following the PR's progression), opinionated but grounded

## Target Audience Assumptions

1. **Familiar with Python** — comfortable with async/await, decorators, type hints
2. **Aware of LLM APIs** — has called OpenAI or similar APIs, understands chat completions
3. **Knows basic Kubernetes** — understands pods, services, deployments (not necessarily CRDs)
4. **Interested in production concerns** — not just "hello world" agents but observability, memory, multi-agent patterns
5. **May or may not know Pydantic AI** — the post should be accessible to newcomers while providing depth for experienced users

## Narrative Arc (Following the PR's Feature Progression)

The blog post mirrors the commit flow — from framework evaluation to production-ready server:

```
Why → Swap → Harden → Extend → Refactor → Ship
```

---

## Section-by-Section Outline

### 1. Introduction: The Problem with Custom Agent Frameworks (500 words)

**Key points:**
- Everyone building AI agents eventually writes the same infrastructure: HTTP API, streaming, memory, tool dispatch, observability
- The temptation is to build everything custom. We did that. Here's why we stopped.
- Framework evaluation: ADK (GCP lock-in), LangChain (complexity), CrewAI (opinionated), Pydantic AI (just right)
- The thesis: **Use a framework for the agent. Build a thin server for the rest.**

**Suggested diagram:** Framework comparison radar chart (type safety, vendor lock-in, complexity, extensibility, observability)

**Code snippet:** None (narrative section)

---

### 2. What is PAIS? (600 words)

**Key points:**
- PAIS = Pydantic AI Server — an enterprise wrapper for Pydantic AI agents
- Three-layer architecture: Pydantic AI (runtime) → PAIS (server) → KAOS (orchestrator)
- What PAIS adds: HTTP API, memory, delegation, health probes, A2A discovery, OTEL
- What PAIS doesn't do: agentic loop, model calls, tool dispatch (that's Pydantic AI)

**Suggested diagram:** Architecture mermaid diagram (from README — Client → AgentServer → Agent → ModelAPI/SubAgents/MCPServers)

**Code snippet:** Minimal agent (5 lines of pure Pydantic AI)
```python
from pydantic_ai import Agent

agent = Agent("test", instructions="You are helpful.", defer_model_check=True)

@agent.tool_plain
def greet(name: str) -> str:
    return f"Hello, {name}!"
```

---

### 3. The Core Swap: Replacing the Agent Engine (800 words)

**Key points:**
- Before: Custom agentic loop with manual tool dispatch, string parsing, response building
- After: `pydantic_ai.Agent` handles everything — one line replaces hundreds
- The `/v1` auto-append lesson (Ollama compatibility)
- Mock response pattern change: Pydantic AI needs 2 responses (not 3) for tool calls
- ContextVar bug with FunctionModel — mutable closure workaround

**Commits referenced:** `e8e26a2`, `c544569`, `5e8e4d8`, `e07128c`, `e8d141c`

**Code snippet:** Before/after comparison of agentic loop invocation
```python
# Before (custom framework) — ~50 lines of loop management
while not done:
    response = await model.chat(messages)
    if response.tool_calls:
        results = await execute_tools(response.tool_calls)
        messages.append(tool_results)
    else:
        done = True

# After (Pydantic AI) — 1 line
result = await agent.run(prompt, deps=deps, usage_limits=limits)
```

---

### 4. String Mode: When Models Can't Call Functions (500 words)

**Key points:**
- Problem: Small models (Ollama smollm2, etc.) don't support `tool_calls` in their API
- Solution: Inject tool descriptions into system prompt, parse JSON from text response
- Implementation: `FunctionModel` wrapper that intercepts model calls
- Same tools, same DelegationToolset, different transport — no code changes for the agent developer

**Commits referenced:** `43b45f7`, `29ca5f9`

**Code snippet:** String mode configuration
```yaml
spec:
  config:
    toolCallMode: string  # or "auto" (default) or "native"
```

**Sidebar:** "What is string-mode tool calling?" — explanation box

---

### 5. Streaming: Real-Time Agent Responses (600 words)

**Key points:**
- OpenAI-compatible SSE streaming via `/v1/chat/completions`
- Progress events: clients see tool execution and delegation in real time
- The span lifecycle gotcha: creating OTEL spans inside generators, not route handlers
- `step` and `max_steps` counters in progress events

**Commits referenced:** `3dfb890`, `dbecda0`, `4c004fd`

**Suggested diagram:** Sequence diagram showing streaming flow:
```
Client → PAIS → Pydantic AI → Model
                     ↓
              SSE: delta chunks
                     ↓
              SSE: progress (tool call)
                     ↓
              SSE: delta chunks (final)
                     ↓
              SSE: [DONE]
```

**Code snippet:** Curl streaming example
```bash
curl -N http://localhost:8000/v1/chat/completions \
  -d '{"model":"agent","messages":[{"role":"user","content":"Run the echo tool"}],"stream":true}'
```

---

### 6. Memory: Stateful Agents Without Stateful Code (700 words)

**Key points:**
- LLMs are stateless. Memory bridges the gap.
- Three backends: LocalMemory (dev), RedisMemory (production), NullMemory (stateless)
- The NullMemory pattern: no `if memory_enabled` branching — always call memory methods
- Conversion layer: KAOS events ↔ Pydantic AI messages
- Context windowing: `memory_context_limit` prevents unbounded history
- Memory as a first-class API: `GET /memory/events`, `GET /memory/sessions`

**Commits referenced:** `b4b99e4`, `c63ae22`, `ab7cd2d`, `75165db`

**Code snippet:** Memory configuration and inspection
```bash
# Deploy with Redis memory
kaos agent deploy my-agent --modelapi api -m llama3.2 \
  --env MEMORY_TYPE=redis --env MEMORY_REDIS_URL=redis://redis:6379

# Inspect memory events
curl http://localhost:8000/memory/events?session_id=my-session
```

---

### 7. Multi-Agent Delegation (700 words)

**Key points:**
- `DelegationToolset` implements Pydantic AI's `AbstractToolset` — sub-agents appear as tools
- Dynamic tool creation: `delegate_to_{agent_name}` for each accessible agent
- The delegation closure bug: accidentally exposing `ctx` and `_toolset` parameters to the model
- A2A agent cards: `/.well-known/agent.json` for zero-config discovery
- Memory events: delegation uses `delegation_request`/`delegation_response` types (not `tool_call`)

**Commits referenced:** `25ea96d`, `b853ad2`, `5aa0914`, `8e7939e`

**Suggested diagram:** Multi-agent delegation flow
```
Orchestrator Agent
  ├── delegate_to_worker_a(task="...")
  │   └── Worker A → ModelAPI → response
  └── delegate_to_worker_b(task="...")
      └── Worker B → ModelAPI → response
```

**Code snippet:** Agent CRD with delegation
```yaml
spec:
  agentNetwork:
    access: [worker-a, worker-b]
    expose: true
```

---

### 8. Observability: OTEL for Non-Deterministic Systems (800 words)

**Key points:**
- Why agents need tracing more than APIs: non-deterministic, multi-step, multi-service
- Layered approach: Pydantic AI spans (automatic) + KAOS spans (delegation, server-run)
- The `instrument=True` trap: don't pass it to the constructor
- Instrumentation versioning: v1 logs vs v2+ attributes
- Metrics: delegation counter, duration histogram
- Log correlation: trace_id/span_id in every log line
- The reverted per-tool spans: Pydantic AI already does this

**Commits referenced:** `5944de2`, `5e8b7f8`, `9f901a5`, `d4f1406`, `2df52dc`

**Suggested diagram:** Span hierarchy visualization (screenshot of Jaeger/SigNoz trace)

**Code snippet:** Enable OTEL
```bash
kaos agent deploy my-agent --modelapi api -m llama3.2 \
  --otel-endpoint http://otel-collector:4317 --expose --wait
```

**Sidebar:** "Recommended local stacks" — SigNoz, Jaeger, Grafana Tempo comparison table

---

### 9. The Deep Refactor: From Agent-Centric to Server-Centric (600 words)

**Key points:**
- Initial architecture: Server → Agent wrapper → Pydantic AI Agent (three layers)
- Problem: The Agent wrapper added nothing — just passed through to Pydantic AI
- Solution: AgentServer directly owns the Pydantic AI Agent
- How it was done: 25+ incremental commits, each one extracting, consolidating, removing
- The payoff: ~1100 lines → ~750 lines, clearer ownership, fewer indirections

**Commits referenced:** Chapter 6 of COMMIT_FLOW_FULL.md (c894d52 is the culmination)

**Suggested diagram:** Before/after architecture comparison
```
Before:  Server → Agent → PydanticAI Agent → Model
After:   AgentServer → PydanticAI Agent → Model
```

---

### 10. KAOS Integration: From Local Dev to Kubernetes (600 words)

**Key points:**
- The same code runs everywhere: `pais run` locally, KAOS in Kubernetes
- Environment variables are the interface between KAOS operator and PAIS
- CRD → env vars → PAIS configuration (no config files, no secrets in code)
- End-to-end deployment example: install → model → agent → invoke

**Code snippet:** Full end-to-end
```bash
# Install
kaos system install --wait

# Deploy model
kaos modelapi deploy api --mode Hosted -m smollm2:135m --wait

# Build and deploy agent
pais init my-agent && cd my-agent
pais build --name my-agent --tag v1 --kind-load
kaos agent deploy my-agent -a api -m smollm2:135m --expose --wait

# Test
kaos agent invoke my-agent --message "Hello!"
```

**Suggested diagram:** Deployment flow (developer → CLI → operator → Kubernetes → running agent)

---

### 11. Pitfalls and Lessons Learned (500 words)

**Key points (derived from PR iterations):**
1. **Don't reimplement what the framework gives you** — Pydantic AI handles the agentic loop, tool tracing, etc.
2. **Mock testing isn't enough** — real model + E2E validation catches integration bugs
3. **Streaming span lifecycle matters** — create spans where they're consumed, not where the route is defined
4. **ContextVar + copied contexts = pain** — use mutable closures for FunctionModel state
5. **NullMemory > boolean flags** — no-op implementations eliminate branching
6. **The wrapper that adds nothing should be removed** — the deep refactor was the biggest win
7. **Framework lock-in is real** — ADK's Vertex dependency would have been a dealbreaker for KAOS

---

### 12. Conclusion: The Three-Layer Rule (300 words)

**Key points:**
- **Pydantic AI**: Agent runtime (reasoning, tools, validation)
- **PAIS**: Server infrastructure (HTTP, memory, delegation, observability)
- **KAOS**: Kubernetes orchestration (CRDs, operator, gateway, CLI)
- Each layer owns one concern. The interfaces between them are clean (env vars, HTTP, ASGI).
- What's next: PyPI publish, full A2A task lifecycle, pais CLI improvements

**Call to action:** Links to repo, docs, getting started

---

## Proposed Diagrams Summary

| # | Diagram | Type | Source |
|---|---------|------|--------|
| 1 | Architecture overview | Mermaid graph | README.md diagram |
| 2 | Streaming SSE flow | Mermaid sequence | Custom |
| 3 | Multi-agent delegation | Mermaid graph | Custom |
| 4 | Span hierarchy | Screenshot | Jaeger/SigNoz (or mermaid approximation) |
| 5 | Before/after refactor | Side-by-side boxes | Custom |
| 6 | Local-to-K8s deployment flow | Mermaid flowchart | Custom |
| 7 | Framework comparison | Table or radar chart | Custom |

## Estimated Length

| Section | Words |
|---------|-------|
| 1. Introduction | 500 |
| 2. What is PAIS? | 600 |
| 3. Core Swap | 800 |
| 4. String Mode | 500 |
| 5. Streaming | 600 |
| 6. Memory | 700 |
| 7. Delegation | 700 |
| 8. Observability | 800 |
| 9. Deep Refactor | 600 |
| 10. KAOS Integration | 600 |
| 11. Pitfalls | 500 |
| 12. Conclusion | 300 |
| **Total** | **~7,200** |
