# PLAN: Extend PAIS to Adopt FastA2A Features

## Context

PAIS (Pydantic AI Server) currently provides an OpenAI-compatible chat completions API with custom A2A discovery, memory persistence, health probes, and delegation. FastA2A is Pydantic's official A2A protocol implementation built on Starlette.

This plan evaluates which FastA2A features PAIS should adopt, how they align with our Kubernetes-native architecture, and what the integration path looks like.

---

## 1. Feature Assessment Matrix

| Feature | FastA2A Has | PAIS Has | Should Adopt? | Priority | Rationale |
|---------|-------------|----------|---------------|----------|-----------|
| A2A-compliant Agent Card | ✅ Full spec | ⚠️ Custom format | **Yes** | High | Standards compliance; interop with A2A ecosystem |
| JSON-RPC transport | ✅ | ❌ (REST/SSE) | **No** (see §2.1) | — | Breaking change; OpenAI compat is our value prop |
| Task lifecycle model | ✅ 9 states | ❌ | **Partial** | Medium | Long-running tasks need this, but not all requests |
| Broker/Worker pattern | ✅ | ❌ | **No** | — | Over-engineering for K8s-native sidecar model |
| Dual storage (task+context) | ✅ | ❌ (session-based) | **Partial** | Low | Our memory model is simpler and sufficient |
| A2A protocol schema types | ✅ TypedDicts | ❌ | **Yes** (subset) | Medium | Reuse for agent card, skills, capabilities |
| Interactive docs UI | ✅ `/docs` | ❌ | **Defer** | Low | Nice-to-have; not critical for K8s |
| OTel span propagation | ✅ API-only | ✅ Full SDK | Already have | — | Our implementation is more complete |
| Health/readiness probes | ❌ | ✅ | Already have | — | FastA2A lacks this; we're ahead |
| Memory endpoints | ❌ | ✅ | Already have | — | Our memory API is unique value |
| Streaming (SSE) | ❌ Not impl | ✅ | Already have | — | We're ahead here |
| Push notifications | ❌ Not impl | ❌ | **Defer** | Low | Spec feature, low practical priority |
| Security schemes | ✅ Schema | ❌ | **Partial** | Medium | Agent card should declare auth |
| Client library | ✅ A2AClient | ✅ RemoteAgent | **No change** | — | Our OpenAI-compat client works for our needs |

---

## 2. Detailed Feature Decisions

### 2.1 Transport Protocol: Keep OpenAI-Compatible REST

**Decision: Do NOT adopt JSON-RPC**

**Rationale:**
- PAIS's value proposition is an **enterprise-ready Pydantic AI server** deployed on Kubernetes. The OpenAI Chat Completions API is the de-facto standard for LLM interaction — every SDK, UI, and tool speaks it
- Switching to JSON-RPC would break all existing integrations (E2E tests, operator, CLI, examples)
- The A2A protocol RC v1.0 adds HTTP/REST as an alternative binding — PAIS can become A2A-compatible without JSON-RPC
- FastA2A's JSON-RPC is a single POST endpoint — harder to route, debug, and monitor than REST
- Our streaming SSE implementation already works; FastA2A hasn't implemented streaming yet

**Future consideration:** If A2A adoption becomes critical for external interoperability, we could add a `/a2a` JSON-RPC endpoint alongside the existing REST API. But this is not a priority for the initial delivery.

### 2.2 Agent Card: Align with A2A Specification ✅ ADOPT

**Current state:** PAIS serves a custom agent card at `GET /.well-known/agent`:
```json
{
  "name": "...",
  "description": "...",
  "url": "...",
  "skills": [{"name": "...", "description": "..."}],
  "capabilities": ["message_processing", "tool_execution"]
}
```

**Target state:** A2A-compliant agent card at `GET /.well-known/agent.json` (standard path):
```json
{
  "name": "...",
  "description": "...",
  "url": "...",
  "version": "1.0.0",
  "protocolVersion": "0.3.0",
  "skills": [{"id": "...", "name": "...", "description": "...", "tags": [...], "inputModes": [...], "outputModes": [...]}],
  "capabilities": {"streaming": true, "pushNotifications": false, "stateTransitionHistory": false},
  "defaultInputModes": ["application/json"],
  "defaultOutputModes": ["application/json"]
}
```

**Implementation approach:**

**Option A: Use fasta2a schema types directly** (Recommended)
- Add `fasta2a` as an optional dependency
- Import `AgentCard`, `Skill`, `AgentCapabilities` from `fasta2a.schema`
- Construct proper A2A card using these types
- Use `agent_card_ta.dump_json(card, by_alias=True)` for camelCase serialization
- Pros: Standards-compliant, auto-updates with fasta2a, no maintenance burden
- Cons: Adds dependency; fasta2a is still beta

**Option B: Implement A2A card types inline**
- Define a subset of A2A AgentCard as Pydantic models or TypedDicts in PAIS
- Only the fields we need: name, description, url, version, protocolVersion, skills, capabilities, defaultInputModes, defaultOutputModes
- Pros: No dependency; full control
- Cons: Must maintain manually; may drift from spec

**Option C: Dual endpoints**
- Keep `/.well-known/agent` with current format for backward compat
- Add `/.well-known/agent.json` with A2A-compliant format
- Pros: No breaking change
- Cons: Two card formats to maintain; confusing

**Recommendation: Option B** — Implement a small A2A-compliant card inline. Our card is simple enough that importing fasta2a just for TypedDicts is overkill. We can define ~30 lines of TypedDict/dataclass. The endpoint path should be `/.well-known/agent.json` per the A2A spec, and we should keep `/.well-known/agent` as an alias for backward compatibility.

### 2.3 Task Lifecycle: Adopt Conceptually, Not Structurally

**Decision: Do NOT adopt the full Task model or Broker/Worker pattern**

**Rationale:**
- PAIS runs as a Kubernetes sidecar — one request maps to one agent execution
- The Broker/Worker indirection (anyio streams, async task queues) adds complexity with no benefit in our deployment model where each pod IS the worker
- Our synchronous request → stream/response model is simpler and works for the K8s use case
- Task states are useful for long-running operations, but our agents complete in seconds-to-minutes, not hours

**What to adopt conceptually:**
- **context_id pattern** — Our `session_id` already serves this purpose (groups related requests). After Phase 15, session_id is returned in the response `id` field, so clients can track conversations
- **Task state awareness** — We could consider adding a `status` concept to our memory model if we ever need to track failed/canceled sessions, but this is not currently needed

### 2.4 Dual Storage Pattern: Defer

**Decision: Do NOT adopt the dual storage pattern**

**Rationale:**
- FastA2A separates "A2A-formatted task history" from "agent-internal context". This makes sense when you need to serve A2A-protocol-compliant task queries to external clients
- PAIS stores Pydantic AI's message history directly in memory. External consumers use our custom `/memory/events` and `/memory/sessions` endpoints
- We don't serve A2A task queries, so there's no need for dual storage
- Our memory model (Local/Redis/Null) is well-tested and fits our needs

**If needed in future:** If we add A2A `tasks/get` support, we'd need to store tasks separately from agent context. At that point, the dual storage pattern would make sense.

### 2.5 A2A Schema Types: Adopt Subset

**Decision: Import or define a subset of A2A types**

The following A2A types would be useful in PAIS:

| Type | Use Case |
|------|----------|
| `AgentCard` | Serve A2A-compliant agent cards |
| `Skill` | Describe tools/delegation capabilities |
| `AgentCapabilities` | Declare streaming, etc. |
| `Part` (Text/File/Data) | Future: richer message content |
| `SecurityScheme` | Future: declare auth requirements |

**Implementation:** Define as inline TypedDicts (10-30 lines). No need for the full 811-line schema.

### 2.6 Security Scheme Declaration: Adopt in Agent Card

**Decision: Add security scheme info to agent card**

**Rationale:**
- PAIS agents deployed behind a gateway may have auth requirements
- The A2A spec defines standard security scheme types (HTTP, API Key, OAuth2, OIDC)
- Declaring these in the agent card enables automated client configuration
- This is metadata-only — no auth implementation change needed

**Implementation:** Add optional `security` and `securitySchemes` fields to the agent card, populated from environment variables or CRD config.

### 2.7 Interactive Docs: Defer

**Decision: Defer**

FastA2A's `/docs` endpoint is a nice development tool but not critical for K8s deployment. Could be added later as a debug-mode feature.

### 2.8 Client (A2AClient): Do NOT Adopt

**Decision: Keep RemoteAgent as-is**

**Rationale:**
- `RemoteAgent` speaks OpenAI Chat Completions, which is what our agents expose
- Switching to `A2AClient` (JSON-RPC) would require all sub-agents to speak A2A
- Our delegation model (DelegationToolset + RemoteAgent) works well
- If we ever need to delegate to external A2A agents, we could add an `A2ARemoteAgent` alongside `RemoteAgent`

---

## 3. Implementation Plan

### Phase 1: A2A-Compliant Agent Card (Small, High Value)

**Scope:** Upgrade the agent card endpoint to be A2A-spec-compliant while maintaining backward compatibility.

**Tasks:**

1. **Define A2A agent card types** — Add minimal TypedDicts for `AgentCard`, `Skill`, `AgentCapabilities` to `serverutils.py` (or a new `a2a_types.py` if warranted)

2. **Upgrade `_get_agent_card()`** — Return A2A-compliant card with:
   - `version` from `VERSION` file or env var
   - `protocolVersion: "0.3.0"`
   - `skills` as proper A2A `Skill` objects with `id`, `name`, `description`, `tags`, `inputModes`, `outputModes`
   - `capabilities` as A2A `AgentCapabilities` dict (streaming: true, pushNotifications: false, stateTransitionHistory: false)
   - `defaultInputModes` / `defaultOutputModes`

3. **Serve at correct path** — Add `/.well-known/agent.json` (A2A standard) alongside `/.well-known/agent` (backward compat, both return same data)

4. **Update RemoteAgent._init()** — Handle both old and new card formats when fetching sub-agent cards

5. **Update tests** — Agent card tests should validate A2A-compliant structure

6. **Update operator** — The Go controller may reference the card endpoint path; verify and update if needed

**Estimated impact:** ~50-80 lines changed across 2-3 files. No breaking changes.

### Phase 2: Agent Card Security Metadata (Optional Follow-up)

**Scope:** Add security scheme declarations to the agent card.

**Tasks:**

1. **Add env vars** — `AGENT_SECURITY_SCHEME` (e.g., `bearer`, `apiKey`), `AGENT_SECURITY_DESCRIPTION`

2. **Populate card fields** — Add `security` and `securitySchemes` to agent card when configured

3. **Update CRD** — Add `spec.security` to Agent CRD for declaring auth requirements

**Estimated impact:** ~30 lines Python + CRD type changes.

### Phase 3: A2A Endpoint (Future / Deferred)

**Scope:** If A2A interoperability becomes a requirement, add a JSON-RPC endpoint alongside the existing REST API.

**Tasks:**

1. **Add `/a2a` POST endpoint** — JSON-RPC router for `message/send` and `tasks/get`
2. **Implement task tracking** — Wrap `_process_message()` with task state management
3. **Consider using FastA2A** — At this point, using FastA2A as a library (mounting it alongside our FastAPI app) becomes viable

**This phase is explicitly deferred** — only implement if external A2A interop becomes a concrete requirement.

---

## 4. What We Should NOT Adopt

| Feature | Why Not |
|---------|---------|
| JSON-RPC as primary transport | Breaks all integrations; OpenAI compat is our value prop |
| Broker/Worker pattern | Over-engineering for K8s sidecar model |
| Starlette migration | We use FastAPI; Starlette is a downgrade in DX |
| Full Task lifecycle | Our request-response model is simpler and sufficient |
| InMemoryBroker (anyio streams) | We don't need async task queuing |
| fasta2a as dependency | Only ~30 lines of types needed; not worth the dep |
| Push notifications | Spec feature with no current use case |

---

## 5. Summary

### Adopt Now (Phase 1)
- ✅ A2A-compliant agent card format
- ✅ Standard discovery endpoint path (`/.well-known/agent.json`)
- ✅ Proper A2A Skill and Capability types
- ✅ Agent card version and protocol version fields

### Adopt Later (Phase 2)
- ⏳ Security scheme declarations in agent card
- ⏳ Interactive docs endpoint (debug mode)

### Explicitly Deferred (Phase 3)
- 🔮 JSON-RPC A2A endpoint (only if interop required)
- 🔮 Task lifecycle model (only if long-running tasks needed)
- 🔮 Mounting FastA2A alongside FastAPI

### Do Not Adopt
- ❌ JSON-RPC as primary transport
- ❌ Broker/Worker pattern
- ❌ Starlette migration
- ❌ fasta2a as runtime dependency
- ❌ Dual storage model

The key insight is: **PAIS and FastA2A solve different problems**. FastA2A is a generic A2A transport layer. PAIS is a Kubernetes-native enterprise server for Pydantic AI agents. The overlap is in agent card discovery — adopt that cleanly, and keep everything else as-is.
