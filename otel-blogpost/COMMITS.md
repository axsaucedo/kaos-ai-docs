# OpenTelemetry Integration Commits - KAOS Framework

This document contains all commits related to OpenTelemetry integration in the KAOS (Kubernetes Agent Orchestration System) framework.

**Branch:** `39-opentelemetry-extension`  
**Total Commits:** 52 OTEL-related commits  
**Excludes:** Commits from `fix-helm-images-and-version` branch (infrastructure fixes unrelated to OTEL)

---

## Post-Merge Commits (After fix-helm-images-and-version merge)

These commits represent the continued OTEL development after merging infrastructure fixes.

### 1f02aa265c9e6c68ca14e1cdd472cd50365dc000
**Date:** 2026-01-28 19:23:10 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): exclude health check endpoints from traces

- Add OTEL_PYTHON_FASTAPI_EXCLUDED_URLS env var to Agent/MCPServer deployments
- Excludes /health and /ready endpoints from FastAPI instrumentation
- Reduces noise from Kubernetes liveness/readiness probes
- LiteLLM already excludes health endpoints (callback-based OTEL)
- Update telemetry documentation with health check exclusion details
- Add unit test for new env var

```

---

### 47eee1930a17d43c073960889a306e770b110135
**Date:** 2026-01-28 19:47:04 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(operator): add OTEL health check exclusions for ModelAPI/LiteLLM

- Add OTEL_PYTHON_EXCLUDED_URLS='^/health.*' env var to LiteLLM deployments
- Uses generic exclusion (not FastAPI-specific) for broader coverage
- Excludes /health/liveliness, /health/liveness, /health/readiness
- Update telemetry docs with separate env var tables for each component type

```

---

### cba50bceb153746d4d0120b0536268a3feaa01c1
**Date:** 2026-01-28 20:00:28 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(operator): simplify OTEL excluded URL patterns for health checks

- Remove regex anchors (^, $) - patterns use search() not match()
- Agent/MCPServer: /health,/ready (matches anywhere in URL)
- ModelAPI: /health (matches all health endpoints)
- Update tests and documentation to match new patterns

The OpenTelemetry Python util uses re.search() which finds patterns
anywhere in the string, so anchors are not needed for simple path
matching.

```

---

### 22b0fd27ee69a1924bc479a86563cf7626758d73
**Date:** 2026-01-28 20:48:27 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add OTLP log export and fix LiteLLM trace export

- Add OTLPLogExporter and LoggingHandler to Python telemetry manager
- Fix LiteLLM OTEL_EXPORTER: 'otlp' -> 'otlp_grpc' (LiteLLM requires explicit exporter type)
- Add OpenTelemetry packages to Dockerfile.litellm (api, sdk, exporter-otlp)
- Update docs to reflect logs export and correct env var values

```

---

### fb9dcef69c48932e664ecc129e9ceaa488234b2f
**Date:** 2026-01-29 07:38:46 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(helm): add logLevel configuration for control and data plane

Add global logLevel setting to Helm chart values.yaml with support for:
- TRACE, DEBUG, INFO (default), WARNING, ERROR levels
- DEFAULT_LOG_LEVEL env var passed to operator via ConfigMap
- Documentation for log level mapping across components

The operator will propagate this to all data plane components
(Agent, MCPServer, ModelAPI) in subsequent commits.

```

---

### b0fea19c487459c6b7224fd366ba5668472ae7d7
**Date:** 2026-01-29 07:40:48 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(python): standardize LOG_LEVEL environment variable

Add LOG_LEVEL as the primary log level env var for Python components:
- Agent: LOG_LEVEL takes precedence over AGENT_LOG_LEVEL (backward compatible)
- MCPServer: LOG_LEVEL takes precedence over MCP_LOG_LEVEL (backward compatible)
- Add get_log_level() helper function for consistent env var resolution

This enables centralized log level control from the operator while
maintaining backward compatibility with existing configurations.

```

---

### 4cd4338fb9209ff90681c6e6f5aa170eac56b838
**Date:** 2026-01-29 07:43:08 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(operator): propagate LOG_LEVEL to all data plane components

- Add util.GetDefaultLogLevel() and util.BuildLogLevelEnvVar() helpers
- Agent controller: pass LOG_LEVEL env var to all agents
- MCPServer controller: pass LOG_LEVEL env var to MCP servers
- ModelAPI controller: map LOG_LEVEL to LITELLM_LOG for LiteLLM proxy
- ModelAPI controller: map LOG_LEVEL to OLLAMA_DEBUG for Ollama hosted
- TRACE/DEBUG/INFO/WARNING/ERROR levels properly mapped to each runtime

```

---

### d7853649cc8d58bbdd17f2fb90cdad07a4e97c68
**Date:** 2026-01-29 07:46:24 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(python): add comprehensive debug logging

- Add OTEL configuration to startup logs (endpoint, service name, attributes)
- Add log level to agent and MCPServer startup config
- Add debug logging for model calls (message count, response length)
- Add debug logging for tool execution (tool name, args, success/failure)
- Add debug logging for sub-agent delegation (target, task length, result)
- Debug statements include exception details for troubleshooting

```

---

### 3dde9b7166d1436a0603938478e6ac2a15544be2
**Date:** 2026-01-29 07:48:02 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add trace context to memory events

- Add get_current_trace_context() to get trace_id and span_id
- Update LocalMemory.create_event() to include trace context in metadata
- Memory events now automatically correlate with active traces when OTel is enabled
- Enables querying logs/events by trace_id for debugging agent workflows

```

---

### 0da23c8d7aeb06ffcb745593ed6176122bc7c818
**Date:** 2026-01-29 07:50:09 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
docs: add REPORT.md with log level implementation evaluation

- Document log level configuration for control and data plane
- Add log level mapping table (KAOS → Python/LiteLLM/Ollama)
- Document trace context correlation in memory events
- Update files changed summary with all new modifications
- Remove REPORT.md from .gitignore to allow tracking

```

---

### 3cd5dabe3f408068e521c56eec54bc3345ee8a46
**Date:** 2026-01-29 08:19:10 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
chore(docs): Revert "docs: add REPORT.md with log level implementation evaluation"

This reverts commit 0da23c8d7aeb06ffcb745593ed6176122bc7c818.

```

---

### 54ec842cf312e62f9d007ca3c7bf0f5c8928ad64
**Date:** 2026-01-29 08:27:43 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(telemetry): use logger.error for failures to enable OTEL log correlation

- Change model call failures from logger.debug to logger.error
- Change tool execution failures from logger.debug to logger.error
- Change delegation failures from logger.debug to logger.error
- Add logger.error to RemoteAgent request failures
- Keep success paths at DEBUG to avoid log noise at INFO level
- Create REPORT.md with analysis of OTEL logging strategy

This ensures failed spans have corresponding log entries at default
LOG_LEVEL=INFO, enabling proper log-trace correlation in SigNoz/Jaeger.

```

---

### 60e7efb5c6b750cc47bc1ccdfe003358e0c07927
**Date:** 2026-01-29 08:42:51 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(telemetry): correct log/span ordering for proper trace correlation

- Move logger.error BEFORE span_failure() so logs have trace context
- Move logger.debug AFTER span_begin() so logs include span context
- span_failure() detaches context, logs after it lose correlation
- Update REPORT.md with correct log/span ordering pattern

Fixes: Logs were being emitted after span context was detached,
resulting in log entries without trace_id/span_id correlation.

```

---

### 1e046405177c26c7b99593a3c334abe84b81e44b
**Date:** 2026-01-29 11:01:10 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
chore: reverted adding files and added to gitignore

Signed-off-by: Alejandro Saucedo <alejandro.saucedo@zalando.de>

```

---

### f9fccbee37706aabba41b79157ca224c62e8c787
**Date:** 2026-01-29 15:25:10 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(telemetry): respect LOG_LEVEL for OTEL log export

- Add _get_log_level() helper to convert LOG_LEVEL env var to logging constant
- Update LoggingHandler to use configured log level instead of hardcoded INFO
- DEBUG logs now exported to OTEL collector when LOG_LEVEL=DEBUG

```

---

### b9fb19b6af692ea7b87507255570fc2f06add531
**Date:** 2026-01-29 15:27:21 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): include logger name in OTEL log exports

- Add KaosLoggingHandler that extends LoggingHandler
- Adds logger_name attribute to log records for visibility in SigNoz
- Standard LoggingHandler excludes name from attributes (uses InstrumentationScope)
- Custom handler ensures logger name is searchable as log attribute

```

---

### 07167f4fa8b20f0b76cbdb173cf996db07a5895c
**Date:** 2026-01-29 15:28:35 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): make HTTPX client tracing opt-in

- Add OTEL_INCLUDE_HTTP_CLIENT env var (default: false)
- HTTPX instrumentation disabled by default to reduce MCP SSE noise
- httpx/httpcore/mcp.client.streamable_http loggers set to WARNING by default
- Enable HTTP client tracing with OTEL_INCLUDE_HTTP_CLIENT=true

```

---

### 97271cc7088e4b861adbdc062e7086aa62ae7110
**Date:** 2026-01-29 15:29:42 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): move uvicorn access log control to Python code

- Remove --no-access-log from Dockerfile CMD
- Add OTEL_INCLUDE_HTTP_SERVER env var (default: false)
- Uvicorn access logs disabled by default via logger level
- Enable with OTEL_INCLUDE_HTTP_SERVER=true for request logging

```

---

### d6741f757ec6c40dae90d8d0d69dc95e1c795447
**Date:** 2026-01-29 15:30:16 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
docs: add OTEL logging configuration documentation

- Document LOG_LEVEL, OTEL_INCLUDE_HTTP_CLIENT, OTEL_INCLUDE_HTTP_SERVER
- Explain log record attributes exported to OTEL
- Provide configuration examples for different scenarios

```

---

### 31260be202aef0c457ea6afdfa8010d469486ac1
**Date:** 2026-01-29 16:01:26 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
refactor(telemetry): consolidate log level functions and make FastAPI instrumentation opt-in

- Moved get_log_level() to telemetry/manager.py as single source of truth
- Added get_log_level_int() for numeric level conversion
- Both support AGENT_LOG_LEVEL fallback for backwards compatibility
- FastAPIInstrumentor now opt-in via OTEL_INCLUDE_HTTP_SERVER (default: false)
- Updated REPORT.md documentation to reflect FastAPI instrumentation behavior

```

---

### c2e88f9e7bc5980ab9655851281675e902393a5a
**Date:** 2026-01-29 18:14:48 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add manual trace context propagation and getenv_bool helper

- Add getenv_bool() helper to reduce boolean env var parsing boilerplate
- Add attach_context() and detach_context() methods to KaosOtelManager
- Inject trace context headers in RemoteAgent.process_message()
- Extract and attach trace context in AgentServer chat endpoint
- Refactor existing env var checks to use getenv_bool
- Update REPORT.md with distributed tracing documentation

This enables connected traces across agent delegations without noisy
HTTP spans from HTTPX/FastAPI instrumentors.

```

---

### 8ff1a09a9859ecd1be558d67121d48773cbbffbd
**Date:** 2026-01-29 19:27:54 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
refactor(telemetry): add extract_and_attach_context helper for simpler trace propagation

- Added extract_and_attach_context() that combines extract + attach in one call
- Handles dict conversion for Starlette Headers automatically
- Simplified server.py chat endpoint to use new helper

```

---

### 3826f826bacb5b790ff557bac33b370c08c68bee
**Date:** 2026-01-29 19:30:30 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
refactor(telemetry): make KaosOtelManager a module-level singleton

- Add _get_service_name() to read from OTEL_SERVICE_NAME/AGENT_NAME
- Add get_instance() class method for singleton access
- Create module-level 'otel' variable for easy import
- Update Agent class to use global 'otel' instead of self._otel
- Service name is now read from env vars, not passed to constructor

Usage: from telemetry.manager import otel

```

---

### 1d52e8620cdc1faefd3a688fcf34e03a988fed4d
**Date:** 2026-01-29 19:32:31 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add OTEL instrumentation to MCPServer and MCPClient

- MCPServer: use shared get_log_level and getenv_bool from telemetry.manager
- MCPServer: add OTEL_INCLUDE_HTTP_CLIENT/SERVER env var support
- MCPClient: add span tracing for tool calls with mcp.tool.{name} spans
- MCPClient: add DEBUG logging for tool calls and results
- Both now respect the same telemetry configuration as AgentServer

```

---

### e2be3af5a9453b240ff5dd21816a11c2ee0526f5
**Date:** 2026-01-29 19:34:35 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add comprehensive DEBUG logging for prompts and memory events

- Log system prompt preview and length on process_message entry
- Log user messages and delegation tasks with content preview
- Log memory event creation for traceability
- Log model input (last user message) and full response preview
- Log tool arguments and result content
- Log delegation task and result preview
- Log OTEL_INCLUDE_HTTP_CLIENT/SERVER in startup config

```

---

### 10b0f6fd32344b89de182133c08130309694a689
**Date:** 2026-01-29 19:36:06 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
docs(telemetry): move REPORT.md content to docs and add gitignore

- Remove REPORT.md from git tracking
- Add logging configuration section to docs/operator/telemetry.md
- Add distributed tracing section explaining manual context propagation
- Add PLAN-OTEL-RESULTS.md to .gitignore for session notes

```

---

### 65da1bcd939d98118d76881b0c8a68540ad0de29
**Date:** 2026-01-30 08:33:06 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
refactor(telemetry): make KaosOtelManager true singleton with __new__

- Use __new__ pattern to ensure only one instance exists
- Log warning if service_name override attempted after init
- Remove get_instance() classmethod (no longer needed)
- Add _reset_for_testing() classmethod for test isolation
- Update tests with setup/teardown to reset singleton state

```

---

### a99d0405b1684efe2293825ff94301680bde6d5c
**Date:** 2026-01-30 19:38:23 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(mcptools): suppress noisy docket.worker and fakeredis DEBUG logs

- Add WARNING level for docket.worker and fakeredis loggers
- These are internal FastMCP dependencies that produce noise at DEBUG

```

---

### c5631c277ac61e4dee02de96e43de022048e7257
**Date:** 2026-01-30 20:58:03 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
chore: updated sample to deepseek

Signed-off-by: Alejandro Saucedo <alejandro.saucedo@zalando.de>

```

---

### 96bc72cc8cfb23212d492e752928c955a3e0e88d
**Date:** 2026-01-31 08:08:44 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
chore: add tmp/ to .gitignore for local work files


```

---

## Pre-Merge Commits (Before fix-helm-images-and-version merge)

These commits represent the initial OTEL implementation work.

### 0879a9a29b5c4a0975efd4a5cfa74dec3175e938
**Date:** 2026-01-24 18:48:53 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add OpenTelemetry core module with tracing and metrics

- Add opentelemetry dependencies to pyproject.toml
- Create agent/telemetry module with config, tracing, and metrics
- Support OTLP export, console export, and log correlation config
- Add semantic conventions for agent, model, tool, delegation spans
- Add counters and histograms for request/model/tool/delegation metrics

```

---
### 2d698f1de12ab3cc21b4c2aadca6351164754a0d
**Date:** 2026-01-24 18:50:36 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add FastAPI instrumentation and OTel config to AgentServer

- Add OTel environment variables to AgentServerSettings
- Initialize telemetry in create_agent_server
- Add FastAPI auto-instrumentation
- Shutdown telemetry on server shutdown

```

---
### e1e442cc199ad9e48b0eb7c35727fe065c8c24ef
**Date:** 2026-01-24 18:53:29 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add tracing to Agent agentic loop, model, tools, delegations

- Add OpenTelemetry tracing spans for agent.process_message
- Add spans for each agentic loop step with step type
- Add model.inference spans for LLM calls with timing metrics
- Add tool.{name} spans for MCP tool executions
- Add delegate.{name} spans for A2A delegations
- Record request, model, tool, delegation metrics

```

---
### d966d7c7e4ae2bce520c348841be04123e350b23
**Date:** 2026-01-24 18:54:40 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add OpenTelemetry log correlation with trace_id/span_id

- Update configure_logging to support OTel correlation format
- Add LoggingInstrumentor when otel_log_correlation is enabled
- Include trace_id and span_id in log messages for correlation

```

---
### 79e78deb280abf7bb58265ed7b1d9ecb9459d695
**Date:** 2026-01-24 18:55:50 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(telemetry): add OpenTelemetry support to MCP server

- Add OTel environment variables to MCPServerSettings
- Initialize telemetry in MCPServer when enabled
- Add log correlation support for MCP server

```

---
### 91fc10ad41c32bfb2519fe7e5cf50278d6f812b2
**Date:** 2026-01-24 18:57:13 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
test(telemetry): add unit tests for OpenTelemetry instrumentation

- Test TelemetryConfig env var loading
- Test tracing utilities (get_tracer, context injection/extraction)
- Test metrics recording functions
- Test AgentServerSettings OTel env vars
- Test MCPServerSettings OTel env vars

```

---
### 89aa37ccea1aa8c7c5d53f1d75caa6ef0719be83
**Date:** 2026-01-24 19:01:50 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
docs(telemetry): add OpenTelemetry documentation page and update instructions

- Create docs/operator/telemetry.md with full OTel configuration guide
- Add telemetry page to VitePress sidebar navigation
- Update python.instructions.md with telemetry module reference
- Include examples for SigNoz, Uptrace, and OTel Collector setup

```

---
### f46e82c3532fe35080d04747aebeab8868dbe78e
**Date:** 2026-01-25 14:19:15 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
refactor(telemetry): simplify Python OTel with KaosOtelManager

- Create manager.py with simplified KaosOtelManager class
- Remove redundant otel_* settings from AgentServerSettings
- Remove redundant otel_* settings from MCPServerSettings
- Use standard OTEL_* env vars for configuration
- Add Starlette instrumentation to MCPServer
- Remove duplicate server span (FastAPI handles it via auto-instrumentation)
- Rewrite telemetry tests for new simplified API
- Add opentelemetry-instrumentation-starlette dependency

The new architecture:
- init_otel(service_name): Process-global initialization, idempotent
- is_otel_enabled(): Check OTEL_ENABLED env var
- KaosOtelManager: Lightweight helper with span(), model_span(), tool_span(), etc.
- timed(): Context manager for duration tracking

All 66 Python tests pass.

```

---
### 8932ef92270de5404755ebdec95ae8ffe309a014
**Date:** 2026-01-25 14:20:39 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
docs(telemetry): update for simplified CRD configuration

- Document simplified CRD with just enabled + endpoint fields
- Add MCPServer telemetry example
- Document advanced config via standard OTEL_* env vars
- Update span hierarchy to reflect FastAPI auto-instrumentation
- Remove obsolete fields (insecure, serviceName, tracesEnabled, etc.)

```

---
### 39c69af7c38b5f8bdc0137a3a1f6c6efa04cd7f0
**Date:** 2026-01-25 14:23:56 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(deps): add opentelemetry-instrumentation-starlette to pyproject.toml


```

---
### 21ac2e862d7ed20b64c9787c1a4c2470c126ee79
**Date:** 2026-01-25 17:11:30 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
refactor(telemetry): move to library scope, use OTEL_SDK_DISABLED, BaseSettings

- Move telemetry module from python/agent/telemetry/ to python/telemetry/
- Use OTEL_SDK_DISABLED (standard OTel env var) instead of OTEL_ENABLED
- Convert OtelConfig to pydantic BaseSettings with required fields
- Add _OtelSingleton for process-global SDK state management
- Fix BuildTelemetryEnvVars to append (not overwrite) OTEL_RESOURCE_ATTRIBUTES
- Track request success/failure properly in record_request metrics
- Remove duplicate LoggingInstrumentor call in agent/server.py
- Update tests and documentation

```

---
### 9fd6c7d73aa080d54936ec18037f0256c3006ff2
**Date:** 2026-01-25 17:41:59 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(docker): add telemetry module to Dockerfile COPY


```

---
### f90e18f9c26e8b1879a1d2df3643cebbf577f2aa
**Date:** 2026-01-25 19:47:02 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
refactor(telemetry): implement inline span API with contextvars

- Remove _OtelSingleton, use simple module-level _initialized flag
- Implement span_begin/span_success/span_failure (no context managers)
- Use contextvars for async-safe span state stack (supports nesting)
- Remove timed() helper (timing handled internally)
- is_otel_enabled() now checks _initialized instead of env var
- Metrics recorded automatically in span_success/span_failure

Agent client refactoring:
- Extract _call_model() helper with inline span pattern
- Extract _execute_tool() helper with inline span pattern
- Extract _execute_delegation() helper with inline span pattern
- Refactor process_message() and _agentic_loop() to use inline spans

Tests updated for new API - all 59 tests passing

```

---
### 1865e893c36cdbad8b57ca82e6458f4f09407dce
**Date:** 2026-01-26 09:28:32 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
feat(helm): add global telemetry configuration

- Add telemetry.enabled and telemetry.endpoint to values.yaml
- Add DEFAULT_TELEMETRY_* env vars to operator ConfigMap
- Add GetDefaultTelemetryConfig() and MergeTelemetryConfig() utilities
- Update agent and mcpserver controllers to use merged config
- Component-level config overrides global defaults
- Add unit tests for telemetry utilities
- Update documentation with global config section

```

---
### ccc6ea23f4662db2bf85736683141d47420865bb
**Date:** 2026-01-26 20:45:32 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(telemetry): critical bug fixes and ModelAPI extension

- Fix ContextVar mutable default bug (allocate per-context list)
- Fix span ending pattern (use else instead of finally after except)
- Add should_enable_otel() for pre-init logging correlation check
- Remove os.Getenv from BuildTelemetryEnvVars (was reading operator env)
- Let OTLPSpanExporter use env vars for TLS/endpoint config
- Remove CRD endpoint default (now required when enabled)
- Clarify MCPServer _otel_enabled test semantics
- Extend ModelAPI CRD with telemetry field
- Add LiteLLM OTel callbacks (success_callback/failure_callback)
- Emit warning for Ollama when telemetry enabled (not supported)
- Update documentation for ModelAPI telemetry

```

---
### f6ca856c4d041964a116d7aafacb3fec69c39bdd
**Date:** 2026-01-26 21:05:07 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
docs(telemetry): clarify OTEL_RESOURCE_ATTRIBUTES behavior

- Clarify that operator sets attributes directly (not appends)
- Note that user values in spec.config.env take precedence

```

---
### 775ad4940a0e40ba299df30401d63ac91f446f21
**Date:** 2026-01-27 06:53:56 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(telemetry): prevent span leakage with try/except/finally pattern

Replace try/except/else with try/except/finally using failed flag to ensure
spans always close, even on early return, yield, or continue statements.

Affected functions:
- process_message(): generator may not reach else on early exit
- _agentic_loop(): contains return/continue that bypass else block
- _call_model, _execute_tool, _execute_delegation: updated for consistency

The pattern ensures span_success() runs in finally unless span_failure()
already handled the span, preventing span leakage in streaming scenarios.

```

---
### f4922f748fed279035c7e6be98dca40b4c09fb36
**Date:** 2026-01-27 06:55:14 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(telemetry): MCPServer uses should_enable_otel() for proper detection

MCPServer was treating 'not disabled' as 'enabled', which enabled OTel
instrumentation/log correlation without required env vars, resulting in
a misconfigured state.

Now uses should_enable_otel() which requires:
- OTEL_SDK_DISABLED != 'true'
- OTEL_SERVICE_NAME is set
- OTEL_EXPORTER_OTLP_ENDPOINT is set

This aligns MCPServer behavior with AgentServer and ensures OTel is only
enabled when properly configured.

```

---
### 210dc695d159f5f48c71e4c756fd08ed28c12173
**Date:** 2026-01-27 06:56:27 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(operator): use standard OTEL_EXPORTER_OTLP_ENDPOINT for LiteLLM

Changed OTEL_ENDPOINT to OTEL_EXPORTER_OTLP_ENDPOINT in ModelAPI controller
to align with OpenTelemetry standard environment variables.

Also documented that custom configYaml requires manual OTel callback setup.

```

---
### 2eacb36cbcbe912c5c37aa21ccf24abe34ddbc06
**Date:** 2026-01-27 07:00:55 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(operator): implement field-wise telemetry config merge with validation

MergeTelemetryConfig now performs field-wise merge instead of all-or-nothing:
- Component can set enabled=true and inherit global endpoint
- Endpoint field: component wins if set, otherwise inherits global

Added IsTelemetryConfigValid() to check for enabled=true with empty endpoint.
Controllers now emit warnings when telemetry config is invalid.

```

---
### 410283081043f24cff18c3b4be70f86278fa8f36
**Date:** 2026-01-27 07:02:07 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(operator): remove localhost:4317 default from telemetry endpoint CRD

Regenerated Helm chart CRDs to remove incorrect default for endpoint.
Endpoint is now truly required when telemetry is enabled, matching docs.

```

---
### 25f25b816ad2e1b5f01687ecba1518cfe8b6cc8d
**Date:** 2026-01-27 07:03:05 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
docs(telemetry): fix metric names and labels to match implementation

Updated docs to reflect actual metric names and labels:
- Metric names: kaos.requests (not kaos.agent.requests)
- Duration unit: milliseconds (not seconds)
- Labels: success=true/false (not status=success/error)
- Labels: agent.name, tool, target (not agent_name, tool_name, etc.)
- Removed mcp_server label (not implemented)

```

---
### 353052ff606b0ab1d2c3e1ff669b092df9627e78
**Date:** 2026-01-27 07:06:18 +0100  
**Author:** Alejandro Saucedo <alejandro.saucedo@zalando.de>

```
fix(telemetry): add proper type annotation for SpanState.token

Changed token field from 'object' to 'Token[Context]' to satisfy type checker.
Added import for Token from contextvars and Context from opentelemetry.context.

```

---
