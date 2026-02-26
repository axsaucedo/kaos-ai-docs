# Upstream Pydantic AI Contribution: Per-Tool OTEL Spans

## Summary

This document describes the upstream contribution to [pydantic-ai](https://github.com/pydantic/pydantic-ai) for adding per-tool OpenTelemetry spans.

## Fork & Branch

- **Fork**: https://github.com/axsaucedo/pydantic-ai
- **Branch**: `feat/per-tool-otel-spans`
- **Target upstream**: `pydantic/pydantic-ai:main`

## Problem

Pydantic AI currently creates a single `'running tools'` span in `_call_tools()` that wraps ALL tool executions. There is no per-tool span, which means:

1. A single slow tool is indistinguishable from many fast tools in traces
2. Unknown tool errors (raised as `ModelRetry` in `_tool_manager.py`) create zero OTEL spans — the error occurs before entering the `'running tools'` span
3. Tool validation errors follow the same path — no observability

## Changes Made

**File**: `pydantic_ai_slim/pydantic_ai/_agent_graph.py`

### 1. Import `nullcontext`
```python
from contextlib import asynccontextmanager, contextmanager, nullcontext
```

### 2. Add `tracer` parameter to `_call_tool()`
```python
async def _call_tool(
    tool_manager: ToolManager[DepsT],
    tool_call: _messages.ToolCallPart,
    tool_call_result: DeferredToolResult | None,
    tool_call_metadata: dict[str, dict[str, Any]] | None,
    tracer: Tracer | None = None,  # NEW
) -> ...:
```

### 3. Wrap function body with per-tool span
```python
span_attrs = {
    'gen_ai.tool.name': tool_call.tool_name,
    'gen_ai.tool.call_id': tool_call.tool_call_id or '',
}
cm = tracer.start_as_current_span(f'tool: {tool_call.tool_name}', attributes=span_attrs) if tracer else nullcontext()
with cm as span:
    # ... existing function body, now indented ...
```

### 4. Record retry/error status
```python
except ToolRetryError as e:
    if span is not None:
        span.set_attribute('gen_ai.tool.status', 'retry')
        span.set_attribute('gen_ai.tool.error', str(e.tool_retry.content))
    return e.tool_retry, None
```

### 5. Record success status
```python
if span is not None:
    span.set_attribute('gen_ai.tool.status', 'success')
return return_part, tool_return.content or None
```

### 6. Pass `tracer` from `_call_tools()` to `_call_tool()`
Both the sequential and parallel call sites now pass `tracer`:
```python
_call_tool(tool_manager, call, tool_call_results.get(call.tool_call_id), tool_call_metadata, tracer)
```

## Semantic Conventions

Uses [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/):
- `gen_ai.tool.name`: Tool name
- `gen_ai.tool.call_id`: Tool call ID
- `gen_ai.tool.status`: `success` or `retry`
- `gen_ai.tool.error`: Error message on retry

## Backward Compatibility

- `tracer` parameter defaults to `None`
- When `None`, uses `nullcontext()` — zero overhead, no behavior change
- Existing `'running tools'` span is preserved (per-tool spans are children)

## Suggested PR Title & Body

**Title**: `feat(otel): add per-tool OTEL spans for individual tool calls`

**Body**:
> This PR adds per-tool OpenTelemetry spans in `_call_tool()`, providing fine-grained observability for individual tool executions.
>
> **What it does:**
> - Creates a span `tool: <name>` for each individual tool call
> - Records `gen_ai.tool.status` (success/retry) and error details
> - Uses OpenTelemetry GenAI semantic conventions for attribute names
> - Backward compatible: spans only created when tracer is active
>
> **Why:**
> The existing `'running tools'` span wraps all tools together. This makes it impossible to distinguish slow tools from fast ones, or to see which specific tool failed. Per-tool spans enable:
> - Performance profiling of individual tools
> - Error attribution to specific tool calls
> - Unknown tool detection in traces (ModelRetry → retry status)
>
> **Size:** ~15 lines of logic changes, rest is re-indentation.
>
> Related: https://github.com/axsaucedo/kaos/issues/89

## KAOS Impact

Once this ships upstream, KAOS can remove its interim per-tool spans in `agent/client.py` (the `otel.span_begin/span_success/span_failure` calls in the iter() loop) and rely entirely on pydantic-ai's native spans.
