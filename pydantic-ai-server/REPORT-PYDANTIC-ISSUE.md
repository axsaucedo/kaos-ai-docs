# Suggested Issue for Pydantic AI: Per-Tool OTEL Spans

## Issue Title

`feat: Add per-tool OpenTelemetry spans for individual tool call observability`

## Issue Body

### Problem

Currently, pydantic-ai creates a single `'running tools'` span in `_call_tools()` that wraps all tool executions together. This makes it impossible to:

1. **Distinguish slow tools from fast ones** — A single tool that takes 5s is indistinguishable from 5 tools that take 1s each
2. **Attribute errors to specific tools** — Unknown tool errors (`ModelRetry` raised in `_tool_manager.py`) happen before the `'running tools'` span is entered, so they produce zero OTEL spans
3. **Profile tool performance** — No way to see which specific tool is the bottleneck in a multi-tool agent

### Proposed Solution

Add per-tool OTEL spans in `_call_tool()` (`_agent_graph.py`), wrapping each individual tool execution:

```python
async def _call_tool(
    ...,
    tracer: Tracer | None = None,
):
    cm = tracer.start_as_current_span(
        f'tool: {tool_call.tool_name}',
        attributes={
            'gen_ai.tool.name': tool_call.tool_name,
            'gen_ai.tool.call_id': tool_call.tool_call_id or '',
        },
    ) if tracer else nullcontext()
    with cm as span:
        # existing logic
        # set gen_ai.tool.status on success/retry
```

This is backward compatible — `tracer` defaults to `None`, using `nullcontext()` when no tracer is active.

### Implementation

I have a working implementation in my fork: https://github.com/axsaucedo/pydantic-ai/tree/feat/per-tool-otel-spans

The change is ~15 lines of logic (rest is re-indentation from the new `with` block). It follows the existing pattern of passing `tracer` from `_call_tools()` to `_call_tool()`.

### Use Case

We're building [KAOS](https://github.com/axsaucedo/kaos) (Kubernetes Agent Orchestration System) which uses pydantic-ai as its agent runtime. Our agents use MCP tools and sub-agent delegation, and we need per-tool observability for debugging multi-agent workflows. Currently we have interim KAOS-side spans that we'd like to remove once pydantic-ai supports this natively.

Related: https://github.com/axsaucedo/kaos/issues/89

### Labels

`enhancement`, `observability`
