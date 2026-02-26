# Full OTEL Refactor Plan: Decommission KaosOtelManager

## Overview

This document outlines the full scope of work required to migrate from the custom `KaosOtelManager` (625 LOC) to a minimal OTEL setup that delegates most tracing to Pydantic AI's native instrumentation.

## Current Architecture

```
┌─────────────────────────────────────────────┐
│ KaosOtelManager (telemetry/manager.py)      │
│ ┌─────────────────────────────────────────┐ │
│ │ SDK Init (TracerProvider, MeterProvider, │ │
│ │          LoggerProvider, OTLP exporters) │ │
│ ├─────────────────────────────────────────┤ │
│ │ Propagators (W3C TraceContext + Baggage) │ │
│ ├─────────────────────────────────────────┤ │
│ │ Custom span API (span_begin/success/    │ │
│ │ failure with ContextVar stack)           │ │
│ ├─────────────────────────────────────────┤ │
│ │ Custom metrics (kaos.requests,          │ │
│ │ kaos.delegations, etc.)                 │ │
│ ├─────────────────────────────────────────┤ │
│ │ Context injection/extraction (A2A)      │ │
│ ├─────────────────────────────────────────┤ │
│ │ HTTP instrumentation (FastAPI + HTTPX)  │ │
│ ├─────────────────────────────────────────┤ │
│ │ Log correlation (KaosLoggingHandler)    │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Target Architecture

```
┌─────────────────────────────────────────────┐
│ kaos_otel.py (~100 LOC)                     │
│ ┌─────────────────────────────────────────┐ │
│ │ init_otel() — SDK init (same as now)    │ │
│ ├─────────────────────────────────────────┤ │
│ │ Propagators (W3C TraceContext + Baggage) │ │
│ ├─────────────────────────────────────────┤ │
│ │ HTTP instrumentation (FastAPI + HTTPX)  │ │
│ ├─────────────────────────────────────────┤ │
│ │ Log correlation (LoggingInstrumentor)   │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Pydantic AI (instrument=True)               │
│ ┌─────────────────────────────────────────┐ │
│ │ Agent run spans (GenAI conventions)     │ │
│ │ Model call spans (token usage, cost)    │ │
│ │ Tool execution spans (per-tool)         │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ client.py (direct OTEL API)                 │
│ ┌─────────────────────────────────────────┐ │
│ │ Delegation spans (tracer.start_as_*)    │ │
│ │ A2A context inject/extract              │ │
│ │ Custom metrics (kaos.delegations, etc.) │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Migration Steps

### Phase 1: Remove Agent-Level Span Duplication

**Files**: `agent/client.py`

1. Remove `otel.span_begin("agent.agentic_loop")` — replaced by Pydantic AI's `agent run: {name}` span
2. Remove corresponding `otel.span_success()` / `otel.span_failure()` in process_message()
3. Remove per-tool OTEL spans from iter() loop (`otel.span_begin(f"agent.tool_call.{name}")`)
4. Keep delegation spans (`otel.span_begin(f"delegate.{name}")`)

**Risk**: Low — Pydantic AI's spans are richer (token usage, cost, GenAI conventions)

**Verification**: Compare trace trees before and after; Pydantic AI's `agent run` span should cover everything the old `agent.agentic_loop` span did, plus model call and tool call details.

### Phase 2: Replace Custom Span API with Direct OTEL

**Files**: `agent/client.py`, `telemetry/manager.py`

1. Replace `otel.span_begin(f"delegate.{name}")` with direct `tracer.start_as_current_span()`
2. Remove `span_begin()`, `span_success()`, `span_failure()` from KaosOtelManager
3. Remove `SpanState`, `_span_stack` ContextVar
4. Use standard OTEL context managers instead:
   ```python
   with tracer.start_as_current_span("delegate.worker", attributes={...}) as span:
       try:
           result = await self._request_client.post(...)
           return result
       except Exception as e:
           span.set_status(StatusCode.ERROR, str(e))
           span.record_exception(e)
           raise
   ```

**Risk**: Medium — the ContextVar-based span stack is complex; direct context managers are simpler

### Phase 3: Simplify Context Propagation

**Files**: `agent/client.py`, `agent/server.py`, `telemetry/manager.py`

1. Replace `KaosOtelManager.inject_context(headers)` with direct `opentelemetry.propagate.inject(headers)`
2. Replace `KaosOtelManager.extract_and_attach_context(request.headers)` with direct `opentelemetry.propagate.extract()`
3. Remove static methods from KaosOtelManager

**Risk**: Low — these are thin wrappers over standard OTEL API

### Phase 4: Extract SDK Init + Rename Module

**Files**: `telemetry/manager.py` → `telemetry/init.py`

1. Keep `init_otel()` function (TracerProvider, MeterProvider, LoggerProvider, propagators)
2. Keep `should_enable_otel()` and `is_otel_enabled()`
3. Keep HTTP instrumentor setup
4. Keep log correlation setup
5. Remove KaosOtelManager class entirely
6. Rename module from `manager.py` to `init.py` (or keep as `manager.py`)

**Risk**: Low — this is mostly file reorganization

### Phase 5: Simplify Custom Metrics

**Files**: `telemetry/manager.py`, `agent/client.py`

1. Keep `kaos.delegations` counter + histogram (KAOS-specific, not in Pydantic AI)
2. Remove `kaos.requests` counter + histogram (replaced by Pydantic AI's agent run spans)
3. Remove `kaos.model.calls` (never used; Pydantic AI has native model metrics)
4. Remove `kaos.tool.calls` (replaced by Pydantic AI's per-tool spans)
5. Create metrics directly in `client.py` instead of through the manager

**Risk**: Low — unused metrics being removed

## LOC Reduction Estimate

| Component | Current LOC | Target LOC | Reduction |
|-----------|-------------|------------|-----------|
| KaosOtelManager class | ~300 | 0 | -300 |
| SDK init function | ~80 | ~80 | 0 |
| Context propagation | ~60 | 0 (use OTEL API directly) | -60 |
| Custom metrics | ~80 | ~20 | -60 |
| HTTP instrumentation | ~40 | ~40 | 0 |
| Log correlation | ~65 | ~65 | 0 |
| **Total** | **~625** | **~205** | **~420 (-67%)** |

## Dependencies to Remove After Migration

None of the OpenTelemetry dependencies change — we're just removing the custom wrapper layer.

## Testing Strategy

1. Unit tests: Update `test_telemetry.py` — remove tests for span_begin/success/failure, keep init/config tests
2. Integration tests: Verify trace trees with in-memory span exporter
3. E2E tests: Verify spans appear correctly in OTEL collector (if configured in cluster)

## Prerequisites

- [x] Pydantic AI instrumentation enabled (`instrument=True`)
- [ ] Verify span correlation in live cluster with OTEL collector
- [ ] Confirm Pydantic AI per-tool spans provide sufficient granularity
- [ ] Assess whether `kaos.requests` metrics are needed (or if Pydantic AI's agent run spans suffice)
