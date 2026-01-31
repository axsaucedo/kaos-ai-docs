# OpenTelemetry Integration - Complete Commit History with Diffs

This document contains all commits with their full diffs related to OpenTelemetry integration in KAOS.

**Branch:** `39-opentelemetry-extension`  
**Purpose:** Detailed technical reference for blog post development

---

## Table of Contents

1. [Core OTEL Module](#core-otel-module)
2. [FastAPI Integration](#fastapi-integration)
3. [Agent Tracing](#agent-tracing)
4. [Log Correlation](#log-correlation)
5. [MCP Server Instrumentation](#mcp-server-instrumentation)
6. [Operator CRD Extensions](#operator-crd-extensions)
7. [Helm Configuration](#helm-configuration)
8. [Bug Fixes & Refinements](#bug-fixes--refinements)

---

## Post-Merge Commits


### 1f02aa265c9e6c68ca14e1cdd472cd50365dc000
**Subject:** feat(telemetry): exclude health check endpoints from traces
**Date:** 2026-01-28 19:23:10 +0100
**Author:** Alejandro Saucedo

- Add OTEL_PYTHON_FASTAPI_EXCLUDED_URLS env var to Agent/MCPServer deployments
- Excludes /health and /ready endpoints from FastAPI instrumentation
- Reduces noise from Kubernetes liveness/readiness probes
- LiteLLM already excludes health endpoints (callback-based OTEL)
- Update telemetry documentation with health check exclusion details
- Add unit test for new env var



### 47eee1930a17d43c073960889a306e770b110135
**Subject:** feat(operator): add OTEL health check exclusions for ModelAPI/LiteLLM
**Date:** 2026-01-28 19:47:04 +0100
**Author:** Alejandro Saucedo

- Add OTEL_PYTHON_EXCLUDED_URLS='^/health.*' env var to LiteLLM deployments
- Uses generic exclusion (not FastAPI-specific) for broader coverage
- Excludes /health/liveliness, /health/liveness, /health/readiness
- Update telemetry docs with separate env var tables for each component type



### cba50bceb153746d4d0120b0536268a3feaa01c1
**Subject:** fix(operator): simplify OTEL excluded URL patterns for health checks
**Date:** 2026-01-28 20:00:28 +0100
**Author:** Alejandro Saucedo

- Remove regex anchors (^, $) - patterns use search() not match()
- Agent/MCPServer: /health,/ready (matches anywhere in URL)
- ModelAPI: /health (matches all health endpoints)
- Update tests and documentation to match new patterns

The OpenTelemetry Python util uses re.search() which finds patterns
anywhere in the string, so anchors are not needed for simple path
matching.



### 22b0fd27ee69a1924bc479a86563cf7626758d73
**Subject:** feat(telemetry): add OTLP log export and fix LiteLLM trace export
**Date:** 2026-01-28 20:48:27 +0100
**Author:** Alejandro Saucedo

- Add OTLPLogExporter and LoggingHandler to Python telemetry manager
- Fix LiteLLM OTEL_EXPORTER: 'otlp' -> 'otlp_grpc' (LiteLLM requires explicit exporter type)
- Add OpenTelemetry packages to Dockerfile.litellm (api, sdk, exporter-otlp)
- Update docs to reflect logs export and correct env var values



### fb9dcef69c48932e664ecc129e9ceaa488234b2f
**Subject:** feat(helm): add logLevel configuration for control and data plane
**Date:** 2026-01-29 07:38:46 +0100
**Author:** Alejandro Saucedo

Add global logLevel setting to Helm chart values.yaml with support for:
- TRACE, DEBUG, INFO (default), WARNING, ERROR levels
- DEFAULT_LOG_LEVEL env var passed to operator via ConfigMap
- Documentation for log level mapping across components

The operator will propagate this to all data plane components
(Agent, MCPServer, ModelAPI) in subsequent commits.



### b0fea19c487459c6b7224fd366ba5668472ae7d7
**Subject:** feat(python): standardize LOG_LEVEL environment variable
**Date:** 2026-01-29 07:40:48 +0100
**Author:** Alejandro Saucedo

Add LOG_LEVEL as the primary log level env var for Python components:
- Agent: LOG_LEVEL takes precedence over AGENT_LOG_LEVEL (backward compatible)
- MCPServer: LOG_LEVEL takes precedence over MCP_LOG_LEVEL (backward compatible)
- Add get_log_level() helper function for consistent env var resolution

This enables centralized log level control from the operator while
maintaining backward compatibility with existing configurations.



### 4cd4338fb9209ff90681c6e6f5aa170eac56b838
**Subject:** feat(operator): propagate LOG_LEVEL to all data plane components
**Date:** 2026-01-29 07:43:08 +0100
**Author:** Alejandro Saucedo

- Add util.GetDefaultLogLevel() and util.BuildLogLevelEnvVar() helpers
- Agent controller: pass LOG_LEVEL env var to all agents
- MCPServer controller: pass LOG_LEVEL env var to MCP servers
- ModelAPI controller: map LOG_LEVEL to LITELLM_LOG for LiteLLM proxy
- ModelAPI controller: map LOG_LEVEL to OLLAMA_DEBUG for Ollama hosted
- TRACE/DEBUG/INFO/WARNING/ERROR levels properly mapped to each runtime



### d7853649cc8d58bbdd17f2fb90cdad07a4e97c68
**Subject:** feat(python): add comprehensive debug logging
**Date:** 2026-01-29 07:46:24 +0100
**Author:** Alejandro Saucedo

- Add OTEL configuration to startup logs (endpoint, service name, attributes)
- Add log level to agent and MCPServer startup config
- Add debug logging for model calls (message count, response length)
- Add debug logging for tool execution (tool name, args, success/failure)
- Add debug logging for sub-agent delegation (target, task length, result)
- Debug statements include exception details for troubleshooting



### 3dde9b7166d1436a0603938478e6ac2a15544be2
**Subject:** feat(telemetry): add trace context to memory events
**Date:** 2026-01-29 07:48:02 +0100
**Author:** Alejandro Saucedo

- Add get_current_trace_context() to get trace_id and span_id
- Update LocalMemory.create_event() to include trace context in metadata
- Memory events now automatically correlate with active traces when OTel is enabled
- Enables querying logs/events by trace_id for debugging agent workflows



### 0da23c8d7aeb06ffcb745593ed6176122bc7c818
**Subject:** docs: add REPORT.md with log level implementation evaluation
**Date:** 2026-01-29 07:50:09 +0100
**Author:** Alejandro Saucedo

- Document log level configuration for control and data plane
- Add log level mapping table (KAOS → Python/LiteLLM/Ollama)
- Document trace context correlation in memory events
- Update files changed summary with all new modifications
- Remove REPORT.md from .gitignore to allow tracking



### 3cd5dabe3f408068e521c56eec54bc3345ee8a46
**Subject:** chore(docs): Revert "docs: add REPORT.md with log level implementation evaluation"
**Date:** 2026-01-29 08:19:10 +0100
**Author:** Alejandro Saucedo

This reverts commit 0da23c8d7aeb06ffcb745593ed6176122bc7c818.



### 54ec842cf312e62f9d007ca3c7bf0f5c8928ad64
**Subject:** fix(telemetry): use logger.error for failures to enable OTEL log correlation
**Date:** 2026-01-29 08:27:43 +0100
**Author:** Alejandro Saucedo

- Change model call failures from logger.debug to logger.error
- Change tool execution failures from logger.debug to logger.error
- Change delegation failures from logger.debug to logger.error
- Add logger.error to RemoteAgent request failures
- Keep success paths at DEBUG to avoid log noise at INFO level
- Create REPORT.md with analysis of OTEL logging strategy

This ensures failed spans have corresponding log entries at default
LOG_LEVEL=INFO, enabling proper log-trace correlation in SigNoz/Jaeger.



### 60e7efb5c6b750cc47bc1ccdfe003358e0c07927
**Subject:** fix(telemetry): correct log/span ordering for proper trace correlation
**Date:** 2026-01-29 08:42:51 +0100
**Author:** Alejandro Saucedo

- Move logger.error BEFORE span_failure() so logs have trace context
- Move logger.debug AFTER span_begin() so logs include span context
- span_failure() detaches context, logs after it lose correlation
- Update REPORT.md with correct log/span ordering pattern

Fixes: Logs were being emitted after span context was detached,
resulting in log entries without trace_id/span_id correlation.



### 1e046405177c26c7b99593a3c334abe84b81e44b
**Subject:** chore: reverted adding files and added to gitignore
**Date:** 2026-01-29 11:01:10 +0100
**Author:** Alejandro Saucedo

Signed-off-by: Alejandro Saucedo <alejandro.saucedo@zalando.de>



### f9fccbee37706aabba41b79157ca224c62e8c787
**Subject:** fix(telemetry): respect LOG_LEVEL for OTEL log export
**Date:** 2026-01-29 15:25:10 +0100
**Author:** Alejandro Saucedo

- Add _get_log_level() helper to convert LOG_LEVEL env var to logging constant
- Update LoggingHandler to use configured log level instead of hardcoded INFO
- DEBUG logs now exported to OTEL collector when LOG_LEVEL=DEBUG



### b9fb19b6af692ea7b87507255570fc2f06add531
**Subject:** feat(telemetry): include logger name in OTEL log exports
**Date:** 2026-01-29 15:27:21 +0100
**Author:** Alejandro Saucedo

- Add KaosLoggingHandler that extends LoggingHandler
- Adds logger_name attribute to log records for visibility in SigNoz
- Standard LoggingHandler excludes name from attributes (uses InstrumentationScope)
- Custom handler ensures logger name is searchable as log attribute



### 07167f4fa8b20f0b76cbdb173cf996db07a5895c
**Subject:** feat(telemetry): make HTTPX client tracing opt-in
**Date:** 2026-01-29 15:28:35 +0100
**Author:** Alejandro Saucedo

- Add OTEL_INCLUDE_HTTP_CLIENT env var (default: false)
- HTTPX instrumentation disabled by default to reduce MCP SSE noise
- httpx/httpcore/mcp.client.streamable_http loggers set to WARNING by default
- Enable HTTP client tracing with OTEL_INCLUDE_HTTP_CLIENT=true



### 97271cc7088e4b861adbdc062e7086aa62ae7110
**Subject:** feat(telemetry): move uvicorn access log control to Python code
**Date:** 2026-01-29 15:29:42 +0100
**Author:** Alejandro Saucedo

- Remove --no-access-log from Dockerfile CMD
- Add OTEL_INCLUDE_HTTP_SERVER env var (default: false)
- Uvicorn access logs disabled by default via logger level
- Enable with OTEL_INCLUDE_HTTP_SERVER=true for request logging



### d6741f757ec6c40dae90d8d0d69dc95e1c795447
**Subject:** docs: add OTEL logging configuration documentation
**Date:** 2026-01-29 15:30:16 +0100
**Author:** Alejandro Saucedo

- Document LOG_LEVEL, OTEL_INCLUDE_HTTP_CLIENT, OTEL_INCLUDE_HTTP_SERVER
- Explain log record attributes exported to OTEL
- Provide configuration examples for different scenarios



### 31260be202aef0c457ea6afdfa8010d469486ac1
**Subject:** refactor(telemetry): consolidate log level functions and make FastAPI instrumentation opt-in
**Date:** 2026-01-29 16:01:26 +0100
**Author:** Alejandro Saucedo

- Moved get_log_level() to telemetry/manager.py as single source of truth
- Added get_log_level_int() for numeric level conversion
- Both support AGENT_LOG_LEVEL fallback for backwards compatibility
- FastAPIInstrumentor now opt-in via OTEL_INCLUDE_HTTP_SERVER (default: false)
- Updated REPORT.md documentation to reflect FastAPI instrumentation behavior



### c2e88f9e7bc5980ab9655851281675e902393a5a
**Subject:** feat(telemetry): add manual trace context propagation and getenv_bool helper
**Date:** 2026-01-29 18:14:48 +0100
**Author:** Alejandro Saucedo

- Add getenv_bool() helper to reduce boolean env var parsing boilerplate
- Add attach_context() and detach_context() methods to KaosOtelManager
- Inject trace context headers in RemoteAgent.process_message()
- Extract and attach trace context in AgentServer chat endpoint
- Refactor existing env var checks to use getenv_bool
- Update REPORT.md with distributed tracing documentation

This enables connected traces across agent delegations without noisy
HTTP spans from HTTPX/FastAPI instrumentors.



### 8ff1a09a9859ecd1be558d67121d48773cbbffbd
**Subject:** refactor(telemetry): add extract_and_attach_context helper for simpler trace propagation
**Date:** 2026-01-29 19:27:54 +0100
**Author:** Alejandro Saucedo

- Added extract_and_attach_context() that combines extract + attach in one call
- Handles dict conversion for Starlette Headers automatically
- Simplified server.py chat endpoint to use new helper



### 3826f826bacb5b790ff557bac33b370c08c68bee
**Subject:** refactor(telemetry): make KaosOtelManager a module-level singleton
**Date:** 2026-01-29 19:30:30 +0100
**Author:** Alejandro Saucedo

- Add _get_service_name() to read from OTEL_SERVICE_NAME/AGENT_NAME
- Add get_instance() class method for singleton access
- Create module-level 'otel' variable for easy import
- Update Agent class to use global 'otel' instead of self._otel
- Service name is now read from env vars, not passed to constructor

Usage: from telemetry.manager import otel



### 1d52e8620cdc1faefd3a688fcf34e03a988fed4d
**Subject:** feat(telemetry): add OTEL instrumentation to MCPServer and MCPClient
**Date:** 2026-01-29 19:32:31 +0100
**Author:** Alejandro Saucedo

- MCPServer: use shared get_log_level and getenv_bool from telemetry.manager
- MCPServer: add OTEL_INCLUDE_HTTP_CLIENT/SERVER env var support
- MCPClient: add span tracing for tool calls with mcp.tool.{name} spans
- MCPClient: add DEBUG logging for tool calls and results
- Both now respect the same telemetry configuration as AgentServer



### e2be3af5a9453b240ff5dd21816a11c2ee0526f5
**Subject:** feat(telemetry): add comprehensive DEBUG logging for prompts and memory events
**Date:** 2026-01-29 19:34:35 +0100
**Author:** Alejandro Saucedo

- Log system prompt preview and length on process_message entry
- Log user messages and delegation tasks with content preview
- Log memory event creation for traceability
- Log model input (last user message) and full response preview
- Log tool arguments and result content
- Log delegation task and result preview
- Log OTEL_INCLUDE_HTTP_CLIENT/SERVER in startup config



### 10b0f6fd32344b89de182133c08130309694a689
**Subject:** docs(telemetry): move REPORT.md content to docs and add gitignore
**Date:** 2026-01-29 19:36:06 +0100
**Author:** Alejandro Saucedo

- Remove REPORT.md from git tracking
- Add logging configuration section to docs/operator/telemetry.md
- Add distributed tracing section explaining manual context propagation
- Add PLAN-OTEL-RESULTS.md to .gitignore for session notes



### 65da1bcd939d98118d76881b0c8a68540ad0de29
**Subject:** refactor(telemetry): make KaosOtelManager true singleton with __new__
**Date:** 2026-01-30 08:33:06 +0100
**Author:** Alejandro Saucedo

- Use __new__ pattern to ensure only one instance exists
- Log warning if service_name override attempted after init
- Remove get_instance() classmethod (no longer needed)
- Add _reset_for_testing() classmethod for test isolation
- Update tests with setup/teardown to reset singleton state



### a99d0405b1684efe2293825ff94301680bde6d5c
**Subject:** fix(mcptools): suppress noisy docket.worker and fakeredis DEBUG logs
**Date:** 2026-01-30 19:38:23 +0100
**Author:** Alejandro Saucedo

- Add WARNING level for docket.worker and fakeredis loggers
- These are internal FastMCP dependencies that produce noise at DEBUG



### c5631c277ac61e4dee02de96e43de022048e7257
**Subject:** chore: updated sample to deepseek
**Date:** 2026-01-30 20:58:03 +0100
**Author:** Alejandro Saucedo

Signed-off-by: Alejandro Saucedo <alejandro.saucedo@zalando.de>



### 96bc72cc8cfb23212d492e752928c955a3e0e88d
**Subject:** chore: add tmp/ to .gitignore for local work files
**Date:** 2026-01-31 08:08:44 +0100
**Author:** Alejandro Saucedo




### Commit: 1f02aa2
**Subject:** feat(telemetry): exclude health check endpoints from traces
**Date:** 2026-01-28 19:23:10 +0100
**Author:** Alejandro Saucedo

- Add OTEL_PYTHON_FASTAPI_EXCLUDED_URLS env var to Agent/MCPServer deployments
- Excludes /health and /ready endpoints from FastAPI instrumentation
- Reduces noise from Kubernetes liveness/readiness probes
- LiteLLM already excludes health endpoints (callback-based OTEL)
- Update telemetry documentation with health check exclusion details
- Add unit test for new env var


```diff
 docs/operator/telemetry.md          | 35 +++++++++++++++++++++++++++++++++++
 operator/pkg/util/telemetry.go      |  8 ++++++++
 operator/pkg/util/telemetry_test.go | 21 ++++++++++++++-------
 3 files changed, 57 insertions(+), 7 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index 37d7b24..5d0f6e7 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -407,9 +407,44 @@ The operator automatically sets these environment variables when telemetry is en
 | `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL from `telemetry.endpoint` |
 | `OTEL_SERVICE_NAME` | Defaults to CR name (agent or MCP server name) |
 | `OTEL_RESOURCE_ATTRIBUTES` | Sets `service.namespace` and `kaos.resource.name`; if user sets same var in spec.config.env, their value takes precedence |
+| `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` | Excludes `/health` and `/ready` endpoints from tracing (reduces noise from Kubernetes probes) |
 
 For additional configuration, use standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html) via `spec.config.env`.
 
+## Health Check Exclusions
+
+By default, Kubernetes liveness and readiness probe endpoints are excluded from telemetry traces. This prevents excessive trace data from health checks that typically run every 10-30 seconds.
+
+### Excluded Endpoints
+
+**Agent and MCPServer (Python):**
+- `/health` - Kubernetes liveness probe
+- `/ready` - Kubernetes readiness probe
+
+These are excluded via the `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` environment variable, which is automatically set when telemetry is enabled.
+
+**ModelAPI (LiteLLM):**
+- `/health/liveliness`, `/health/liveness`, `/health/readiness`
+
+These endpoints are not traced because LiteLLM uses callback-based OpenTelemetry integration that only traces LLM completion requests, not HTTP endpoints.
+
+### Customizing Exclusions
+
+To exclude additional URLs from tracing, override the `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` environment variable:
+
+```yaml
+spec:
+  config:
+    telemetry:
+      enabled: true
+      endpoint: "http://otel-collector:4317"
+    env:
+    - name: OTEL_PYTHON_FASTAPI_EXCLUDED_URLS
+      value: "^/health$,^/ready$,^/metrics$,^/internal/.*"
+```
+
+The value is a comma-separated list of regex patterns. Note that when you override this variable, you must include the default patterns (`^/health$,^/ready$`) if you still want to exclude health checks.
+
 ## Troubleshooting
 
 ### No traces appearing
diff --git a/operator/pkg/util/telemetry.go b/operator/pkg/util/telemetry.go
index 0b55109..d682b7f 100644
--- a/operator/pkg/util/telemetry.go
+++ b/operator/pkg/util/telemetry.go
@@ -98,5 +98,13 @@ func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, names
 		Value: kaosAttrs,
 	})
 
+	// Exclude health check endpoints from FastAPI instrumentation traces
+	// Reduces noise from Kubernetes liveness/readiness probes
+	// Uses regex patterns: "^/health$" matches exactly /health, "^/ready$" matches exactly /ready
+	envVars = append(envVars, corev1.EnvVar{
+		Name:  "OTEL_PYTHON_FASTAPI_EXCLUDED_URLS",
+		Value: "^/health$,^/ready$",
+	})
+
 	return envVars
 }
diff --git a/operator/pkg/util/telemetry_test.go b/operator/pkg/util/telemetry_test.go
index f60d9e2..c4644d3 100644
--- a/operator/pkg/util/telemetry_test.go
+++ b/operator/pkg/util/telemetry_test.go
@@ -184,12 +184,12 @@ func TestBuildTelemetryEnvVars(t *testing.T) {
 	os.Unsetenv("OTEL_RESOURCE_ATTRIBUTES")
 
 	tests := []struct {
-		name         string
-		tel          *kaosv1alpha1.TelemetryConfig
-		serviceName  string
-		namespace    string
-		expectCount  int
-		expectOTEL   bool
+		name        string
+		tel         *kaosv1alpha1.TelemetryConfig
+		serviceName string
+		namespace   string
+		expectCount int
+		expectOTEL  bool
 	}{
 		{
 			name:        "nil config returns empty",
@@ -209,7 +209,7 @@ func TestBuildTelemetryEnvVars(t *testing.T) {
 			},
 			serviceName: "test-agent",
 			namespace:   "default",
-			expectCount: 4, // OTEL_SDK_DISABLED, OTEL_SERVICE_NAME, OTEL_EXPORTER_OTLP_ENDPOINT, OTEL_RESOURCE_ATTRIBUTES
+			expectCount: 5, // OTEL_SDK_DISABLED, OTEL_SERVICE_NAME, OTEL_EXPORTER_OTLP_ENDPOINT, OTEL_RESOURCE_ATTRIBUTES, OTEL_PYTHON_FASTAPI_EXCLUDED_URLS
 			expectOTEL:  true,
 		},
 	}
@@ -225,6 +225,7 @@ func TestBuildTelemetryEnvVars(t *testing.T) {
 			if tt.expectOTEL {
 				hasSDKDisabled := false
 				hasServiceName := false
+				hasExcludedURLs := false
 				for _, env := range result {
 					if env.Name == "OTEL_SDK_DISABLED" && env.Value == "false" {
 						hasSDKDisabled = true
@@ -232,6 +233,9 @@ func TestBuildTelemetryEnvVars(t *testing.T) {
 					if env.Name == "OTEL_SERVICE_NAME" && env.Value == tt.serviceName {
 						hasServiceName = true
 					}
+					if env.Name == "OTEL_PYTHON_FASTAPI_EXCLUDED_URLS" && env.Value == "^/health$,^/ready$" {
+						hasExcludedURLs = true
+					}
 				}
 				if !hasSDKDisabled {
 					t.Error("expected OTEL_SDK_DISABLED=false")
@@ -239,6 +243,9 @@ func TestBuildTelemetryEnvVars(t *testing.T) {
 				if !hasServiceName {
 					t.Errorf("expected OTEL_SERVICE_NAME=%s", tt.serviceName)
 				}
+				if !hasExcludedURLs {
+					t.Error("expected OTEL_PYTHON_FASTAPI_EXCLUDED_URLS=^/health$,^/ready$")
+				}
 			}
 		})
 	}
```

---

### Commit: 47eee19
**Subject:** feat(operator): add OTEL health check exclusions for ModelAPI/LiteLLM
**Date:** 2026-01-28 19:47:04 +0100
**Author:** Alejandro Saucedo

- Add OTEL_PYTHON_EXCLUDED_URLS='^/health.*' env var to LiteLLM deployments
- Uses generic exclusion (not FastAPI-specific) for broader coverage
- Excludes /health/liveliness, /health/liveness, /health/readiness
- Update telemetry docs with separate env var tables for each component type


```diff
 docs/operator/telemetry.md                  | 13 ++++++++++++-
 operator/controllers/modelapi_controller.go |  7 +++++++
 2 files changed, 19 insertions(+), 1 deletion(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index 5d0f6e7..4db1469 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -401,6 +401,8 @@ config:
 
 The operator automatically sets these environment variables when telemetry is enabled:
 
+**Agent and MCPServer:**
+
 | Variable | Description |
 |----------|-------------|
 | `OTEL_SDK_DISABLED` | "false" when telemetry is enabled (standard OTel env var) |
@@ -409,6 +411,15 @@ The operator automatically sets these environment variables when telemetry is en
 | `OTEL_RESOURCE_ATTRIBUTES` | Sets `service.namespace` and `kaos.resource.name`; if user sets same var in spec.config.env, their value takes precedence |
 | `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` | Excludes `/health` and `/ready` endpoints from tracing (reduces noise from Kubernetes probes) |
 
+**ModelAPI (LiteLLM):**
+
+| Variable | Description |
+|----------|-------------|
+| `OTEL_EXPORTER` | "otlp" to enable OTLP exporter |
+| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL from `telemetry.endpoint` |
+| `OTEL_SERVICE_NAME` | Defaults to ModelAPI CR name |
+| `OTEL_PYTHON_EXCLUDED_URLS` | Excludes `/health.*` endpoints from tracing (generic exclusion for all instrumentations) |
+
 For additional configuration, use standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html) via `spec.config.env`.
 
 ## Health Check Exclusions
@@ -426,7 +437,7 @@ These are excluded via the `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` environment varia
 **ModelAPI (LiteLLM):**
 - `/health/liveliness`, `/health/liveness`, `/health/readiness`
 
-These endpoints are not traced because LiteLLM uses callback-based OpenTelemetry integration that only traces LLM completion requests, not HTTP endpoints.
+These endpoints are excluded via the `OTEL_PYTHON_EXCLUDED_URLS` environment variable (generic version that covers all instrumentations). LiteLLM uses callback-based OpenTelemetry for LLM traces, but may also have FastAPI auto-instrumentation enabled.
 
 ### Customizing Exclusions
 
diff --git a/operator/controllers/modelapi_controller.go b/operator/controllers/modelapi_controller.go
index 7c664f2..eff1150 100644
--- a/operator/controllers/modelapi_controller.go
+++ b/operator/controllers/modelapi_controller.go
@@ -511,6 +511,13 @@ func (r *ModelAPIReconciler) constructContainer(modelapi *kaosv1alpha1.ModelAPI)
 				Name:  "OTEL_SERVICE_NAME",
 				Value: modelapi.Name,
 			})
+			// Exclude health check endpoints from OTEL traces (reduces noise from K8s probes)
+			// Uses OTEL_PYTHON_EXCLUDED_URLS (generic) since LiteLLM may use various instrumentations
+			// LiteLLM health endpoints: /health/liveliness, /health/liveness, /health/readiness
+			env = append(env, corev1.EnvVar{
+				Name:  "OTEL_PYTHON_EXCLUDED_URLS",
+				Value: "^/health.*",
+			})
 		}
 
 	} else {
```

---

### Commit: cba50bc
**Subject:** fix(operator): simplify OTEL excluded URL patterns for health checks
**Date:** 2026-01-28 20:00:28 +0100
**Author:** Alejandro Saucedo

- Remove regex anchors (^, $) - patterns use search() not match()
- Agent/MCPServer: /health,/ready (matches anywhere in URL)
- ModelAPI: /health (matches all health endpoints)
- Update tests and documentation to match new patterns

The OpenTelemetry Python util uses re.search() which finds patterns
anywhere in the string, so anchors are not needed for simple path
matching.


```diff
 docs/operator/telemetry.md                  | 10 +++++-----
 operator/controllers/modelapi_controller.go |  2 +-
 operator/pkg/util/telemetry.go              |  4 ++--
 operator/pkg/util/telemetry_test.go         |  4 ++--
 4 files changed, 10 insertions(+), 10 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index 4db1469..7760aab 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -418,7 +418,7 @@ The operator automatically sets these environment variables when telemetry is en
 | `OTEL_EXPORTER` | "otlp" to enable OTLP exporter |
 | `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL from `telemetry.endpoint` |
 | `OTEL_SERVICE_NAME` | Defaults to ModelAPI CR name |
-| `OTEL_PYTHON_EXCLUDED_URLS` | Excludes `/health.*` endpoints from tracing (generic exclusion for all instrumentations) |
+| `OTEL_PYTHON_EXCLUDED_URLS` | Excludes `/health` endpoints from tracing (generic exclusion for all instrumentations) |
 
 For additional configuration, use standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html) via `spec.config.env`.
 
@@ -432,12 +432,12 @@ By default, Kubernetes liveness and readiness probe endpoints are excluded from
 - `/health` - Kubernetes liveness probe
 - `/ready` - Kubernetes readiness probe
 
-These are excluded via the `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` environment variable, which is automatically set when telemetry is enabled.
+These are excluded via the `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` environment variable set to `/health,/ready`. The patterns use regex `search()` (not `match()`), so they match anywhere in the URL path.
 
 **ModelAPI (LiteLLM):**
 - `/health/liveliness`, `/health/liveness`, `/health/readiness`
 
-These endpoints are excluded via the `OTEL_PYTHON_EXCLUDED_URLS` environment variable (generic version that covers all instrumentations). LiteLLM uses callback-based OpenTelemetry for LLM traces, but may also have FastAPI auto-instrumentation enabled.
+These endpoints are excluded via the `OTEL_PYTHON_EXCLUDED_URLS` environment variable set to `/health`. This matches all health-related endpoints.
 
 ### Customizing Exclusions
 
@@ -451,10 +451,10 @@ spec:
       endpoint: "http://otel-collector:4317"
     env:
     - name: OTEL_PYTHON_FASTAPI_EXCLUDED_URLS
-      value: "^/health$,^/ready$,^/metrics$,^/internal/.*"
+      value: "/health,/ready,/metrics,/internal"
 ```
 
-The value is a comma-separated list of regex patterns. Note that when you override this variable, you must include the default patterns (`^/health$,^/ready$`) if you still want to exclude health checks.
+The value is a comma-separated list of patterns. Patterns are matched using regex `search()`, so `/health` will match any URL containing `/health`. Note that when you override this variable, you must include the default patterns (`/health,/ready`) if you still want to exclude health checks.
 
 ## Troubleshooting
 
diff --git a/operator/controllers/modelapi_controller.go b/operator/controllers/modelapi_controller.go
index eff1150..f7e17d8 100644
--- a/operator/controllers/modelapi_controller.go
+++ b/operator/controllers/modelapi_controller.go
@@ -516,7 +516,7 @@ func (r *ModelAPIReconciler) constructContainer(modelapi *kaosv1alpha1.ModelAPI)
 			// LiteLLM health endpoints: /health/liveliness, /health/liveness, /health/readiness
 			env = append(env, corev1.EnvVar{
 				Name:  "OTEL_PYTHON_EXCLUDED_URLS",
-				Value: "^/health.*",
+				Value: "/health",
 			})
 		}
 
diff --git a/operator/pkg/util/telemetry.go b/operator/pkg/util/telemetry.go
index d682b7f..5986e42 100644
--- a/operator/pkg/util/telemetry.go
+++ b/operator/pkg/util/telemetry.go
@@ -100,10 +100,10 @@ func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, names
 
 	// Exclude health check endpoints from FastAPI instrumentation traces
 	// Reduces noise from Kubernetes liveness/readiness probes
-	// Uses regex patterns: "^/health$" matches exactly /health, "^/ready$" matches exactly /ready
+	// Uses simple patterns that match anywhere in URL path (search, not match)
 	envVars = append(envVars, corev1.EnvVar{
 		Name:  "OTEL_PYTHON_FASTAPI_EXCLUDED_URLS",
-		Value: "^/health$,^/ready$",
+		Value: "/health,/ready",
 	})
 
 	return envVars
diff --git a/operator/pkg/util/telemetry_test.go b/operator/pkg/util/telemetry_test.go
index c4644d3..a5fb4d4 100644
--- a/operator/pkg/util/telemetry_test.go
+++ b/operator/pkg/util/telemetry_test.go
@@ -233,7 +233,7 @@ func TestBuildTelemetryEnvVars(t *testing.T) {
 					if env.Name == "OTEL_SERVICE_NAME" && env.Value == tt.serviceName {
 						hasServiceName = true
 					}
-					if env.Name == "OTEL_PYTHON_FASTAPI_EXCLUDED_URLS" && env.Value == "^/health$,^/ready$" {
+					if env.Name == "OTEL_PYTHON_FASTAPI_EXCLUDED_URLS" && env.Value == "/health,/ready" {
 						hasExcludedURLs = true
 					}
 				}
@@ -244,7 +244,7 @@ func TestBuildTelemetryEnvVars(t *testing.T) {
 					t.Errorf("expected OTEL_SERVICE_NAME=%s", tt.serviceName)
 				}
 				if !hasExcludedURLs {
-					t.Error("expected OTEL_PYTHON_FASTAPI_EXCLUDED_URLS=^/health$,^/ready$")
+					t.Error("expected OTEL_PYTHON_FASTAPI_EXCLUDED_URLS=/health,/ready")
 				}
 			}
 		})
```

---

### Commit: 22b0fd2
**Subject:** feat(telemetry): add OTLP log export and fix LiteLLM trace export
**Date:** 2026-01-28 20:48:27 +0100
**Author:** Alejandro Saucedo

- Add OTLPLogExporter and LoggingHandler to Python telemetry manager
- Fix LiteLLM OTEL_EXPORTER: 'otlp' -> 'otlp_grpc' (LiteLLM requires explicit exporter type)
- Add OpenTelemetry packages to Dockerfile.litellm (api, sdk, exporter-otlp)
- Update docs to reflect logs export and correct env var values


```diff
 docs/operator/telemetry.md                  |  4 ++--
 operator/controllers/modelapi_controller.go |  5 +++--
 operator/hack/Dockerfile.litellm            |  6 +++++-
 python/telemetry/manager.py                 | 13 +++++++++++++
 4 files changed, 23 insertions(+), 5 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index 7760aab..7131c1b 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -8,7 +8,7 @@ When enabled, OpenTelemetry instrumentation provides:
 
 - **Tracing**: Distributed traces across agent requests, model calls, tool executions, and delegations
 - **Metrics**: Counters and histograms for requests, latency, and error rates
-- **Log Correlation**: Automatic injection of trace_id and span_id into log entries
+- **Logs Export**: Automatic export of Python logs via OTLP (in addition to trace_id/span_id correlation)
 
 ## Global Telemetry Configuration
 
@@ -415,7 +415,7 @@ The operator automatically sets these environment variables when telemetry is en
 
 | Variable | Description |
 |----------|-------------|
-| `OTEL_EXPORTER` | "otlp" to enable OTLP exporter |
+| `OTEL_EXPORTER` | "otlp_grpc" for gRPC OTLP exporter (port 4317); use "otlp_http" for HTTP (port 4318) |
 | `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL from `telemetry.endpoint` |
 | `OTEL_SERVICE_NAME` | Defaults to ModelAPI CR name |
 | `OTEL_PYTHON_EXCLUDED_URLS` | Excludes `/health` endpoints from tracing (generic exclusion for all instrumentations) |
diff --git a/operator/controllers/modelapi_controller.go b/operator/controllers/modelapi_controller.go
index f7e17d8..89ab1d1 100644
--- a/operator/controllers/modelapi_controller.go
+++ b/operator/controllers/modelapi_controller.go
@@ -494,10 +494,11 @@ func (r *ModelAPIReconciler) constructContainer(modelapi *kaosv1alpha1.ModelAPI)
 		// Add OTel env vars for LiteLLM when telemetry is enabled
 		telemetry := util.MergeTelemetryConfig(modelapi.Spec.Telemetry)
 		if telemetry != nil && telemetry.Enabled {
-			// LiteLLM uses OTEL_EXPORTER="otlp" to enable OTel exporter
+			// LiteLLM uses OTEL_EXPORTER to select exporter type
+			// Use "otlp_grpc" for gRPC collector (port 4317) or "otlp_http" for HTTP (port 4318)
 			env = append(env, corev1.EnvVar{
 				Name:  "OTEL_EXPORTER",
-				Value: "otlp",
+				Value: "otlp_grpc",
 			})
 			if telemetry.Endpoint != "" {
 				// Use standard OTEL_EXPORTER_OTLP_ENDPOINT env var
diff --git a/operator/hack/Dockerfile.litellm b/operator/hack/Dockerfile.litellm
index d7333df..473ae3d 100644
--- a/operator/hack/Dockerfile.litellm
+++ b/operator/hack/Dockerfile.litellm
@@ -9,7 +9,11 @@ WORKDIR /app
 
 # Install LiteLLM proxy deps with pinned version
 # 1.80.0 adds support for nebius provider and other improvements
-RUN pip install --no-cache-dir "litellm[proxy]==1.80.0"
+# Include OpenTelemetry packages for OTEL tracing support
+RUN pip install --no-cache-dir "litellm[proxy]==1.80.0" \
+    opentelemetry-api \
+    opentelemetry-sdk \
+    opentelemetry-exporter-otlp
 
 EXPOSE 8000
 
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 543c75c..2d9609b 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -21,14 +21,18 @@ from typing import Any, Dict, List, Optional
 
 from pydantic_settings import BaseSettings, SettingsConfigDict
 from opentelemetry import trace, metrics, context as otel_context
+from opentelemetry import _logs as otel_logs
 from opentelemetry.context import Context
 from opentelemetry.sdk.trace import TracerProvider
 from opentelemetry.sdk.trace.export import BatchSpanProcessor
 from opentelemetry.sdk.metrics import MeterProvider
 from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
+from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
+from opentelemetry.sdk._logs.export import BatchLogRecordProcessor
 from opentelemetry.sdk.resources import Resource, SERVICE_NAME
 from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
 from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
+from opentelemetry.exporter.otlp.proto.grpc._log_exporter import OTLPLogExporter
 from opentelemetry.propagate import set_global_textmap, inject, extract
 from opentelemetry.propagators.composite import CompositePropagator
 from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
@@ -180,6 +184,15 @@ def init_otel(service_name: Optional[str] = None) -> bool:
     meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
     metrics.set_meter_provider(meter_provider)
 
+    # Initialize logs export - exports Python logs to OTLP collector
+    otlp_log_exporter = OTLPLogExporter()  # Uses OTEL_EXPORTER_OTLP_* env vars
+    logger_provider = LoggerProvider(resource=resource)
+    logger_provider.add_log_record_processor(BatchLogRecordProcessor(otlp_log_exporter))
+    otel_logs.set_logger_provider(logger_provider)
+    # Attach OTEL handler to root logger to export all logs
+    otel_handler = LoggingHandler(level=logging.INFO, logger_provider=logger_provider)
+    logging.getLogger().addHandler(otel_handler)
+
     logger.info(
         f"OpenTelemetry initialized: {config.otel_exporter_otlp_endpoint} "
         f"(service: {config.otel_service_name})"
```

---

### Commit: fb9dcef
**Subject:** feat(helm): add logLevel configuration for control and data plane
**Date:** 2026-01-29 07:38:46 +0100
**Author:** Alejandro Saucedo

Add global logLevel setting to Helm chart values.yaml with support for:
- TRACE, DEBUG, INFO (default), WARNING, ERROR levels
- DEFAULT_LOG_LEVEL env var passed to operator via ConfigMap
- Documentation for log level mapping across components

The operator will propagate this to all data plane components
(Agent, MCPServer, ModelAPI) in subsequent commits.


```diff
 operator/chart/templates/operator-configmap.yaml | 2 ++
 operator/chart/values.yaml                       | 8 ++++++++
 operator/hack/Dockerfile.litellm                 | 6 +-----
 3 files changed, 11 insertions(+), 5 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/operator/chart/templates/operator-configmap.yaml b/operator/chart/templates/operator-configmap.yaml
index 26d0305..8b21d6e 100644
--- a/operator/chart/templates/operator-configmap.yaml
+++ b/operator/chart/templates/operator-configmap.yaml
@@ -30,3 +30,5 @@ data:
   {{- else }}
   DEFAULT_TELEMETRY_ENABLED: "false"
   {{- end }}
+  # Global log level for all components
+  DEFAULT_LOG_LEVEL: {{ .Values.logLevel | default "INFO" | upper | quote }}
diff --git a/operator/chart/values.yaml b/operator/chart/values.yaml
index 6d7a222..898d81e 100644
--- a/operator/chart/values.yaml
+++ b/operator/chart/values.yaml
@@ -66,3 +66,11 @@ telemetry:
   # endpoint is the OTLP endpoint URL (required when enabled)
   # Example: "http://otel-collector.observability:4317"
   endpoint: ""
+
+# Global log level for all components (control plane and data plane)
+# Supported values: TRACE, DEBUG, INFO, WARNING, ERROR
+# - Control plane (operator): Uses Go slog levels
+# - Data plane (Agent, MCPServer): Uses Python logging levels
+# - ModelAPI (LiteLLM): Maps to LITELLM_LOG
+# - ModelAPI (Ollama): Maps to OLLAMA_DEBUG
+logLevel: INFO
diff --git a/operator/hack/Dockerfile.litellm b/operator/hack/Dockerfile.litellm
index 473ae3d..89484ab 100644
--- a/operator/hack/Dockerfile.litellm
+++ b/operator/hack/Dockerfile.litellm
@@ -10,11 +10,7 @@ WORKDIR /app
 # Install LiteLLM proxy deps with pinned version
 # 1.80.0 adds support for nebius provider and other improvements
 # Include OpenTelemetry packages for OTEL tracing support
-RUN pip install --no-cache-dir "litellm[proxy]==1.80.0" \
-    opentelemetry-api \
-    opentelemetry-sdk \
-    opentelemetry-exporter-otlp
-
+RUN pip install --no-cache-dir "litellm[proxy]==1.81.4"
 EXPOSE 8000
 
 ENTRYPOINT ["litellm"]
```

---

### Commit: b0fea19
**Subject:** feat(python): standardize LOG_LEVEL environment variable
**Date:** 2026-01-29 07:40:48 +0100
**Author:** Alejandro Saucedo

Add LOG_LEVEL as the primary log level env var for Python components:
- Agent: LOG_LEVEL takes precedence over AGENT_LOG_LEVEL (backward compatible)
- MCPServer: LOG_LEVEL takes precedence over MCP_LOG_LEVEL (backward compatible)
- Add get_log_level() helper function for consistent env var resolution

This enables centralized log level control from the operator while
maintaining backward compatibility with existing configurations.


```diff
 python/agent/server.py    | 12 ++++++++++--
 python/mcptools/server.py |  9 ++++++++-
 2 files changed, 18 insertions(+), 3 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/server.py b/python/agent/server.py
index e3964af..0ce70ce 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -6,6 +6,7 @@ Supports both streaming and non-streaming responses.
 Includes OpenTelemetry instrumentation for tracing, metrics, and log correlation.
 """
 
+import os
 import time
 import uuid
 import logging
@@ -15,7 +16,7 @@ from contextlib import asynccontextmanager
 
 from fastapi import FastAPI, HTTPException, Request
 from fastapi.responses import JSONResponse, StreamingResponse
-from pydantic import BaseModel
+from pydantic import BaseModel, model_validator
 from pydantic_settings import BaseSettings
 import uvicorn
 
@@ -26,6 +27,11 @@ from mcptools.client import MCPClient
 from telemetry.manager import init_otel, is_otel_enabled, should_enable_otel
 
 
+def get_log_level() -> str:
+    """Get log level from environment, preferring LOG_LEVEL over AGENT_LOG_LEVEL."""
+    return os.getenv("LOG_LEVEL", os.getenv("AGENT_LOG_LEVEL", "INFO")).upper()
+
+
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
     """Configure logging for the application.
 
@@ -477,7 +483,9 @@ def create_agent_server(
     otel_should_enable = should_enable_otel()
 
     # Configure logging with optional OTel correlation
-    configure_logging(settings.agent_log_level, otel_correlation=otel_should_enable)
+    # Use LOG_LEVEL env var (preferred) or fallback to AGENT_LOG_LEVEL
+    log_level = get_log_level()
+    configure_logging(log_level, otel_correlation=otel_should_enable)
 
     model_api = ModelAPI(model=settings.model_name, api_base=settings.model_api_url)
 
diff --git a/python/mcptools/server.py b/python/mcptools/server.py
index e447af7..4239313 100644
--- a/python/mcptools/server.py
+++ b/python/mcptools/server.py
@@ -12,6 +12,11 @@ from starlette.routing import Route
 from starlette.responses import JSONResponse
 
 
+def get_log_level() -> str:
+    """Get log level from environment, preferring LOG_LEVEL over MCP_LOG_LEVEL."""
+    return os.getenv("LOG_LEVEL", os.getenv("MCP_LOG_LEVEL", "INFO")).upper()
+
+
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
     """Configure logging for the application.
 
@@ -91,7 +96,9 @@ class MCPServer:
         otel_enabled = should_enable_otel()
 
         # Configure logging with optional OTel correlation
-        configure_logging(settings.mcp_log_level, otel_correlation=otel_enabled)
+        # Use LOG_LEVEL env var (preferred) or fallback to MCP_LOG_LEVEL
+        log_level = get_log_level()
+        configure_logging(log_level, otel_correlation=otel_enabled)
 
         self._host = settings.mcp_host
         self._port = settings.mcp_port
```

---

### Commit: 4cd4338
**Subject:** feat(operator): propagate LOG_LEVEL to all data plane components
**Date:** 2026-01-29 07:43:08 +0100
**Author:** Alejandro Saucedo

- Add util.GetDefaultLogLevel() and util.BuildLogLevelEnvVar() helpers
- Agent controller: pass LOG_LEVEL env var to all agents
- MCPServer controller: pass LOG_LEVEL env var to MCP servers
- ModelAPI controller: map LOG_LEVEL to LITELLM_LOG for LiteLLM proxy
- ModelAPI controller: map LOG_LEVEL to OLLAMA_DEBUG for Ollama hosted
- TRACE/DEBUG/INFO/WARNING/ERROR levels properly mapped to each runtime


```diff
 operator/controllers/agent_controller.go     |  5 ++++
 operator/controllers/mcpserver_controller.go |  5 ++++
 operator/controllers/modelapi_controller.go  | 35 +++++++++++++++++++++++++++-
 operator/pkg/util/telemetry.go               | 27 +++++++++++++++++++++
 4 files changed, 71 insertions(+), 1 deletion(-)
```

#### Detailed Changes

```diff
diff --git a/operator/controllers/agent_controller.go b/operator/controllers/agent_controller.go
index bdbdf12..5579400 100644
--- a/operator/controllers/agent_controller.go
+++ b/operator/controllers/agent_controller.go
@@ -549,6 +549,11 @@ func (r *AgentReconciler) constructEnvVars(agent *kaosv1alpha1.Agent, modelapi *
 		env = append(env, otelEnv...)
 	}
 
+	// Add LOG_LEVEL env var (if not already set by user in spec.config.env)
+	if logLevelEnv := util.BuildLogLevelEnvVar(env); logLevelEnv != nil {
+		env = append(env, logLevelEnv...)
+	}
+
 	return env
 }
 
diff --git a/operator/controllers/mcpserver_controller.go b/operator/controllers/mcpserver_controller.go
index 0ad7bcd..a0a1577 100644
--- a/operator/controllers/mcpserver_controller.go
+++ b/operator/controllers/mcpserver_controller.go
@@ -337,6 +337,11 @@ func (r *MCPServerReconciler) constructPythonContainer(mcpserver *kaosv1alpha1.M
 		env = append(env, otelEnv...)
 	}
 
+	// Add LOG_LEVEL env var (if not already set by user in spec.config.env)
+	if logLevelEnv := util.BuildLogLevelEnvVar(env); logLevelEnv != nil {
+		env = append(env, logLevelEnv...)
+	}
+
 	container := corev1.Container{
 		Name:            "mcp-server",
 		Image:           image,
diff --git a/operator/controllers/modelapi_controller.go b/operator/controllers/modelapi_controller.go
index 89ab1d1..0e83d8b 100644
--- a/operator/controllers/modelapi_controller.go
+++ b/operator/controllers/modelapi_controller.go
@@ -485,9 +485,15 @@ func (r *ModelAPIReconciler) constructContainer(modelapi *kaosv1alpha1.ModelAPI)
 			}
 		}
 		if !hasLiteLLMLog {
+			// Map LOG_LEVEL to LITELLM_LOG (LiteLLM supports DEBUG, INFO, WARNING, ERROR)
+			litellmLogLevel := util.GetDefaultLogLevel()
+			// TRACE -> DEBUG for LiteLLM (no TRACE level)
+			if litellmLogLevel == "TRACE" {
+				litellmLogLevel = "DEBUG"
+			}
 			env = append(env, corev1.EnvVar{
 				Name:  "LITELLM_LOG",
-				Value: "INFO",
+				Value: litellmLogLevel,
 			})
 		}
 
@@ -535,6 +541,33 @@ func (r *ModelAPIReconciler) constructContainer(modelapi *kaosv1alpha1.ModelAPI)
 		if modelapi.Spec.HostedConfig != nil {
 			env = append(env, modelapi.Spec.HostedConfig.Env...)
 		}
+
+		// Map LOG_LEVEL to OLLAMA_DEBUG (Ollama uses 0=INFO, 1=DEBUG, 2=TRACE)
+		hasOllamaDebug := false
+		for _, e := range env {
+			if e.Name == "OLLAMA_DEBUG" {
+				hasOllamaDebug = true
+				break
+			}
+		}
+		if !hasOllamaDebug {
+			logLevel := util.GetDefaultLogLevel()
+			var ollamaDebugLevel string
+			switch logLevel {
+			case "TRACE":
+				ollamaDebugLevel = "2"
+			case "DEBUG":
+				ollamaDebugLevel = "1"
+			default:
+				ollamaDebugLevel = "0" // INFO, WARNING, ERROR -> no debug
+			}
+			if ollamaDebugLevel != "0" { // Only set if enabling debug
+				env = append(env, corev1.EnvVar{
+					Name:  "OLLAMA_DEBUG",
+					Value: ollamaDebugLevel,
+				})
+			}
+		}
 	}
 
 	// Build volume mounts - add litellm-config for Proxy mode (always uses config file)
diff --git a/operator/pkg/util/telemetry.go b/operator/pkg/util/telemetry.go
index 5986e42..66060d1 100644
--- a/operator/pkg/util/telemetry.go
+++ b/operator/pkg/util/telemetry.go
@@ -108,3 +108,30 @@ func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, names
 
 	return envVars
 }
+
+// GetDefaultLogLevel returns the default log level from the DEFAULT_LOG_LEVEL env var.
+// Falls back to "INFO" if not set.
+func GetDefaultLogLevel() string {
+	level := os.Getenv("DEFAULT_LOG_LEVEL")
+	if level == "" {
+		return "INFO"
+	}
+	return level
+}
+
+// BuildLogLevelEnvVar creates the LOG_LEVEL env var if not already in the provided list.
+// Returns a slice with the LOG_LEVEL env var, or empty if already present.
+func BuildLogLevelEnvVar(existingEnv []corev1.EnvVar) []corev1.EnvVar {
+	// Check if LOG_LEVEL is already set
+	for _, e := range existingEnv {
+		if e.Name == "LOG_LEVEL" {
+			return nil // Already set by user
+		}
+	}
+	return []corev1.EnvVar{
+		{
+			Name:  "LOG_LEVEL",
+			Value: GetDefaultLogLevel(),
+		},
+	}
+}
```

---

### Commit: d785364
**Subject:** feat(python): add comprehensive debug logging
**Date:** 2026-01-29 07:46:24 +0100
**Author:** Alejandro Saucedo

- Add OTEL configuration to startup logs (endpoint, service name, attributes)
- Add log level to agent and MCPServer startup config
- Add debug logging for model calls (message count, response length)
- Add debug logging for tool execution (tool name, args, success/failure)
- Add debug logging for sub-agent delegation (target, task length, result)
- Debug statements include exception details for troubleshooting


```diff
 python/agent/client.py    |  9 +++++++++
 python/agent/server.py    | 13 +++++++++++++
 python/mcptools/server.py | 15 +++++++++++++++
 3 files changed, 37 insertions(+)
```

#### Detailed Changes

```diff
diff --git a/python/agent/client.py b/python/agent/client.py
index 21f3b16..40e68c0 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -518,11 +518,14 @@ class Agent:
         )
         failed = False
         try:
+            logger.debug(f"Model call: {model_name}, messages count: {len(messages)}")
             content = cast(str, await self.model_api.process_message(messages, stream=False))
+            logger.debug(f"Model response length: {len(content)}")
             return content
         except Exception as e:
             failed = True
             self._otel.span_failure(e)
+            logger.debug(f"Model call failed: {type(e).__name__}: {e}")
             raise
         finally:
             if not failed:
@@ -530,6 +533,7 @@ class Agent:
 
     async def _execute_tool(self, tool_name: str, tool_args: Dict[str, Any]) -> Any:
         """Execute a tool with tracing."""
+        logger.debug(f"Executing tool: {tool_name}, args: {list(tool_args.keys())}")
         self._otel.span_begin(
             f"tool.{tool_name}",
             kind=SpanKind.CLIENT,
@@ -547,10 +551,12 @@ class Agent:
 
             if tool_result is None:
                 raise ValueError(f"Tool '{tool_name}' not found")
+            logger.debug(f"Tool {tool_name} completed successfully")
             return tool_result
         except Exception as e:
             failed = True
             self._otel.span_failure(e)
+            logger.debug(f"Tool {tool_name} failed: {type(e).__name__}: {e}")
             raise
         finally:
             if not failed:
@@ -564,6 +570,7 @@ class Agent:
         session_id: str,
     ) -> str:
         """Execute delegation to a sub-agent with tracing."""
+        logger.debug(f"Delegating to sub-agent: {agent_name}, task length: {len(task)}")
         self._otel.span_begin(
             f"delegate.{agent_name}",
             kind=SpanKind.CLIENT,
@@ -576,10 +583,12 @@ class Agent:
             result = await self.delegate_to_sub_agent(
                 agent_name, task, context_messages, session_id
             )
+            logger.debug(f"Delegation to {agent_name} completed, result length: {len(result)}")
             return result
         except Exception as e:
             failed = True
             self._otel.span_failure(e)
+            logger.debug(f"Delegation to {agent_name} failed: {type(e).__name__}: {e}")
             raise
         finally:
             if not failed:
diff --git a/python/agent/server.py b/python/agent/server.py
index 0ce70ce..e3a993a 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -211,6 +211,7 @@ class AgentServer:
         logger.info(f"Max Steps: {self.agent.max_steps}")
         logger.info(f"Memory Context Limit: {self.agent.memory_context_limit}")
         logger.info(f"Memory Enabled: {self.agent.memory_enabled}")
+        logger.info(f"Log Level: {get_log_level()}")
 
         # Log model API info
         if self.agent.model_api:
@@ -233,6 +234,18 @@ class AgentServer:
         else:
             logger.info("Sub-agents: None")
 
+        # Log OpenTelemetry configuration
+        otel_enabled = is_otel_enabled()
+        logger.info(f"OpenTelemetry Enabled: {otel_enabled}")
+        if otel_enabled:
+            logger.info(f"  OTEL_SERVICE_NAME: {os.getenv('OTEL_SERVICE_NAME', 'N/A')}")
+            logger.info(
+                f"  OTEL_EXPORTER_OTLP_ENDPOINT: {os.getenv('OTEL_EXPORTER_OTLP_ENDPOINT', 'N/A')}"
+            )
+            logger.debug(
+                f"  OTEL_RESOURCE_ATTRIBUTES: {os.getenv('OTEL_RESOURCE_ATTRIBUTES', 'N/A')}"
+            )
+
         logger.info(f"Access Log: {self.access_log}")
         logger.info("=" * 60)
 
diff --git a/python/mcptools/server.py b/python/mcptools/server.py
index 4239313..fe06cd6 100644
--- a/python/mcptools/server.py
+++ b/python/mcptools/server.py
@@ -129,6 +129,8 @@ class MCPServer:
 
     def _log_startup_config(self):
         """Log server configuration on startup for debugging."""
+        from telemetry.manager import is_otel_enabled
+
         logger.info("=" * 60)
         logger.info("MCPServer Starting (Streamable HTTP)")
         logger.info("=" * 60)
@@ -142,6 +144,19 @@ class MCPServer:
             func = self.tools_registry[tool_name]
             doc = func.__doc__.split("\n")[0] if func.__doc__ else "No description"
             logger.info(f"  - {tool_name}: {doc}")
+
+        # Log OpenTelemetry configuration
+        otel_enabled = is_otel_enabled()
+        logger.info(f"OpenTelemetry Enabled: {otel_enabled}")
+        if otel_enabled:
+            logger.info(f"  OTEL_SERVICE_NAME: {os.getenv('OTEL_SERVICE_NAME', 'N/A')}")
+            logger.info(
+                f"  OTEL_EXPORTER_OTLP_ENDPOINT: {os.getenv('OTEL_EXPORTER_OTLP_ENDPOINT', 'N/A')}"
+            )
+            logger.debug(
+                f"  OTEL_RESOURCE_ATTRIBUTES: {os.getenv('OTEL_RESOURCE_ATTRIBUTES', 'N/A')}"
+            )
+
         logger.info("=" * 60)
 
     def register_tools(self, tools: Dict[str, Callable]):
```

---

### Commit: 3dde9b7
**Subject:** feat(telemetry): add trace context to memory events
**Date:** 2026-01-29 07:48:02 +0100
**Author:** Alejandro Saucedo

- Add get_current_trace_context() to get trace_id and span_id
- Update LocalMemory.create_event() to include trace context in metadata
- Memory events now automatically correlate with active traces when OTel is enabled
- Enables querying logs/events by trace_id for debugging agent workflows


```diff
 python/agent/memory.py      | 15 ++++++++++++++-
 python/telemetry/manager.py | 23 +++++++++++++++++++++++
 2 files changed, 37 insertions(+), 1 deletion(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/memory.py b/python/agent/memory.py
index 714dbc5..eddf005 100644
--- a/python/agent/memory.py
+++ b/python/agent/memory.py
@@ -237,13 +237,26 @@ class LocalMemory:
 
         Returns:
             MemoryEvent instance
+
+        If OpenTelemetry is enabled, automatically includes trace_id and span_id
+        in the metadata for log correlation.
         """
+        from telemetry.manager import is_otel_enabled, get_current_trace_context
+
+        event_metadata = metadata.copy() if metadata else {}
+
+        # Add trace context if OTel is enabled
+        if is_otel_enabled():
+            trace_ctx = get_current_trace_context()
+            if trace_ctx:
+                event_metadata.update(trace_ctx)
+
         return MemoryEvent(
             event_id=f"event_{uuid.uuid4().hex[:8]}",
             timestamp=datetime.now(timezone.utc),
             event_type=event_type,
             content=content,
-            metadata=metadata or {},
+            metadata=event_metadata,
         )
 
     async def list_sessions(self, user_id: Optional[str] = None) -> List[str]:
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 2d9609b..c241ebe 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -103,6 +103,29 @@ def is_otel_enabled() -> bool:
     return _initialized
 
 
+def get_current_trace_context() -> Optional[Dict[str, str]]:
+    """Get current trace context (trace_id, span_id) if available.
+
+    Returns:
+        Dictionary with trace_id and span_id, or None if no active span.
+    """
+    if not _initialized:
+        return None
+
+    current_span = trace.get_current_span()
+    if current_span is None:
+        return None
+
+    span_context = current_span.get_span_context()
+    if not span_context.is_valid:
+        return None
+
+    return {
+        "trace_id": format(span_context.trace_id, "032x"),
+        "span_id": format(span_context.span_id, "016x"),
+    }
+
+
 def should_enable_otel() -> bool:
     """Check if OTel should be enabled based on environment variables.
 
```

---

### Commit: 54ec842
**Subject:** fix(telemetry): use logger.error for failures to enable OTEL log correlation
**Date:** 2026-01-29 08:27:43 +0100
**Author:** Alejandro Saucedo

- Change model call failures from logger.debug to logger.error
- Change tool execution failures from logger.debug to logger.error
- Change delegation failures from logger.debug to logger.error
- Add logger.error to RemoteAgent request failures
- Keep success paths at DEBUG to avoid log noise at INFO level
- Create REPORT.md with analysis of OTEL logging strategy

This ensures failed spans have corresponding log entries at default
LOG_LEVEL=INFO, enabling proper log-trace correlation in SigNoz/Jaeger.


```diff
 PLAN_OTEL.md           | 396 +++++++++++++++++++++++++++++++++++++++++++++++++
 python/agent/client.py |   7 +-
 2 files changed, 400 insertions(+), 3 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/PLAN_OTEL.md b/PLAN_OTEL.md
new file mode 100644
index 0000000..586afaf
--- /dev/null
+++ b/PLAN_OTEL.md
@@ -0,0 +1,396 @@
+# OpenTelemetry Implementation Report
+
+## Summary
+
+This report documents the comprehensive fixes and enhancements to the KAOS OpenTelemetry implementation across both the Python data plane and Go control plane.
+
+## Bug Fixes Completed
+
+### 1. ContextVar Mutable Default Bug (CRITICAL)
+
+**Problem**: The `_span_stack` ContextVar was initialized with `default=[]`, which creates a shared mutable object across all async contexts. This could lead to span stack corruption when multiple requests run concurrently.
+
+**Fix**: Changed default to `None` and allocate per-context:
+```python
+_span_stack: ContextVar[Optional[List[SpanState]]] = ContextVar("kaos_span_stack", default=None)
+
+def _get_stack(self) -> List[SpanState]:
+    stack = _span_stack.get()
+    if stack is None:
+        stack = []
+        _span_stack.set(stack)
+    return stack
+```
+
+**Files Modified**: `python/telemetry/manager.py`
+
+### 2. Span Ending Pattern (except/finally → except/else)
+
+**Problem**: Using `span_success()` in `finally` block runs after exception is re-raised, which is semantically wrong even though no-op guards prevent double-ending.
+
+**Fix**: Changed pattern to use `else` block for success:
+```python
+self._otel.span_begin(...)
+try:
+    result = ...
+except Exception as e:
+    self._otel.span_failure(e)
+    raise
+else:
+    self._otel.span_success()
+    return result
+```
+
+**Files Modified**: `python/agent/client.py`
+
+### 3. otel_enabled Computed Before init_otel()
+
+**Problem**: `is_otel_enabled()` returns `_initialized` which is `False` before `init_otel()` is called, so logging correlation wasn't being enabled at startup.
+
+**Fix**: Added `should_enable_otel()` function that checks environment variables directly:
+```python
+def should_enable_otel() -> bool:
+    disabled = os.getenv("OTEL_SDK_DISABLED", "false").lower() in ("true", "1", "yes")
+    has_service_name = bool(os.getenv("OTEL_SERVICE_NAME"))
+    has_endpoint = bool(os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT"))
+    return not disabled and has_service_name and has_endpoint
+```
+
+**Files Modified**: `python/telemetry/manager.py`, `python/agent/server.py`
+
+### 4. OTEL_RESOURCE_ATTRIBUTES Reading Operator's Env
+
+**Problem**: `BuildTelemetryEnvVars` used `os.Getenv("OTEL_RESOURCE_ATTRIBUTES")` which reads the operator pod's environment, not the target workload's CR env.
+
+**Fix**: Removed `os.Getenv()` and set KAOS attributes directly. User values in `spec.config.env` take precedence when they appear later in the env list.
+
+**Files Modified**: `operator/pkg/util/telemetry.go`
+
+### 5. Exporter TLS/Insecure Behavior
+
+**Problem**: Explicitly passing `endpoint=...` to `OTLPSpanExporter` could bypass environment-based configuration for TLS settings.
+
+**Fix**: Let `OTLPSpanExporter()` use environment variables for endpoint configuration so that `OTEL_EXPORTER_OTLP_INSECURE` and other advanced settings work correctly.
+
+**Files Modified**: `python/telemetry/manager.py`
+
+### 6. Docs/CRD Endpoint Default Mismatch
+
+**Problem**: CRD had a default endpoint value (`http://localhost:4317`) but docs said endpoint is required when enabled.
+
+**Fix**: Removed default from CRD so endpoint is truly required. Updated `make generate manifests`.
+
+**Files Modified**: `operator/api/v1alpha1/agent_types.go`
+
+### 7. MCPServer _otel_enabled Test Semantics
+
+**Problem**: Tests claimed MCPServer is "enabled by default" but `_otel_enabled=True` means "not disabled", not "fully configured and active".
+
+**Fix**: Clarified test names and docstrings to reflect actual semantics.
+
+**Files Modified**: `python/tests/test_telemetry.py`
+
+### 8. ModelAPI CRD Telemetry Extension
+
+**Problem**: ModelAPI CRD didn't support telemetry configuration.
+
+**Fix**: 
+- Added `Telemetry *TelemetryConfig` field to `ModelAPISpec`
+- For LiteLLM Proxy mode: 
+  - Generate config with `success_callback: ["otel"]` and `failure_callback: ["otel"]`
+  - Set `OTEL_EXPORTER=otlp`, `OTEL_ENDPOINT`, `OTEL_SERVICE_NAME` env vars
+- For Ollama Hosted mode:
+  - Emit warning: "OpenTelemetry telemetry is not supported for Ollama (Hosted mode)"
+
+**Files Modified**: `operator/api/v1alpha1/modelapi_types.go`, `operator/controllers/modelapi_controller.go`, `docs/operator/telemetry.md`
+
+## Manual Validation Results
+
+### Test Setup
+- Deployed OpenTelemetry Collector in `monitoring` namespace with debug exporter
+- Created test resources in `otel-validation` namespace:
+  - ModelAPI (Proxy mode) with telemetry enabled
+  - MCPServer with telemetry enabled
+  - Agent with telemetry enabled
+
+### Verification Results
+
+#### ✅ Environment Variables
+All resources correctly received OTel environment variables:
+```
+OTEL_SDK_DISABLED=false
+OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.monitoring.svc.cluster.local:4317
+OTEL_SERVICE_NAME=<resource-name>
+OTEL_RESOURCE_ATTRIBUTES=service.namespace=otel-validation,kaos.resource.name=<resource-name>
+```
+
+#### ✅ Initialization Logs
+Agent logs show successful OTel initialization:
+```
+telemetry.manager - INFO - OpenTelemetry initialized: http://otel-collector.monitoring.svc.cluster.local:4317 (service: test-agent)
+agent.server - INFO - OpenTelemetry instrumentation enabled (FastAPI + HTTPX)
+```
+
+#### ✅ Trace Context Propagation
+Same trace ID propagates from Agent to MCPServer:
+```
+# Agent logs
+mcp.client.streamable_http - INFO - [trace_id=2c7e1777b629adf3162a5d7d281b1afc span_id=ad3d87815fcf8613] - Received session ID...
+
+# MCPServer logs  
+mcp.server.streamable_http_manager - INFO - [trace_id=2c7e1777b629adf3162a5d7d281b1afc span_id=0607d56ff716aee5] - Created new transport...
+```
+
+#### ✅ Log Correlation
+All log entries include trace context when within a traced request:
+```
+2026-01-26 20:01:24 - modelapi.client - ERROR - [trace_id=2c7e1777b629adf3162a5d7d281b1afc span_id=793a53477df4024c] - HTTP error...
+```
+
+#### ✅ Spans Exported
+OTel Collector received spans with correct attributes:
+```
+Resource attributes:
+  -> telemetry.sdk.language: Str(python)
+  -> telemetry.sdk.name: Str(opentelemetry)
+  -> service.namespace: Str(otel-validation)
+  -> kaos.resource.name: Str(test-agent)
+  -> service.name: Str(test-agent)
+
+Span:
+  Trace ID: 2c7e1777b629adf3162a5d7d281b1afc
+  Name: GET /mcp
+  Kind: Server
+```
+
+#### ✅ LiteLLM Configuration
+ModelAPI Proxy correctly generates LiteLLM config with OTel callbacks:
+```yaml
+litellm_settings:
+  success_callback: ["otel"]
+  failure_callback: ["otel"]
+```
+
+#### ✅ Ollama Warning
+Operator correctly emits warning for Hosted mode:
+```
+WARNING: OpenTelemetry telemetry is not supported for Ollama (Hosted mode). Traces and metrics will not be collected.
+```
+
+## Architecture Overview
+
+### CRD Configuration
+Minimal CRD fields (advanced settings via standard `OTEL_*` env vars):
+- `telemetry.enabled`: Enable/disable OpenTelemetry
+- `telemetry.endpoint`: OTLP gRPC endpoint (required when enabled)
+
+### Operator-Set Environment Variables
+| Variable | Description |
+|----------|-------------|
+| `OTEL_SDK_DISABLED` | "false" when enabled (standard OTel env var) |
+| `OTEL_EXPORTER_OTLP_ENDPOINT` | From `telemetry.endpoint` |
+| `OTEL_SERVICE_NAME` | CR name |
+| `OTEL_RESOURCE_ATTRIBUTES` | `service.namespace`, `kaos.resource.name` |
+
+### Python Telemetry Manager
+- Inline span API: `span_begin()`, `span_success()`, `span_failure()`
+- ContextVar-based span stack for async safety
+- Auto-instrumentation: FastAPI, HTTPX, logging correlation
+
+### Span Hierarchy
+```
+HTTP POST /v1/chat/completions (SERVER, auto-instrumented)
+└── agent.agentic_loop (INTERNAL)
+    ├── agent.step.1 (INTERNAL)
+    │   └── model.inference (CLIENT)
+    ├── agent.step.2 (INTERNAL)
+    │   ├── model.inference (CLIENT)
+    │   └── tool.calculator (CLIENT)
+    └── agent.step.3 (INTERNAL)
+        └── delegate.researcher (CLIENT)
+```
+
+## Bug Fixes Round 2 (Additional Fixes)
+
+### 1. Python Span Leakage in Success Paths
+
+**Problem**: Using `try/except/else` with `return`/`yield` inside `try` block bypasses the `else` block, leaving spans open and context attached.
+
+**Fix**: Changed to `try/except/finally` with `failed` flag:
+```python
+self._otel.span_begin(...)
+failed = False
+try:
+    ...
+except Exception as e:
+    failed = True
+    self._otel.span_failure(e)
+    raise
+finally:
+    if not failed:
+        self._otel.span_success()
+```
+
+**Files Modified**: `python/agent/client.py`
+
+### 2. MCPServer Treating "Not Disabled" as "Enabled"
+
+**Problem**: MCPServer checked only `OTEL_SDK_DISABLED`, not whether required env vars exist. This caused instrumentation to run without a valid backend.
+
+**Fix**: Changed to use `should_enable_otel()` from telemetry.manager which checks all required env vars.
+
+**Files Modified**: `python/mcptools/server.py`, `python/tests/test_telemetry.py`
+
+### 3. LiteLLM OTel Env Var Names
+
+**Problem**: Operator set `OTEL_ENDPOINT` but standard is `OTEL_EXPORTER_OTLP_ENDPOINT`.
+
+**Fix**: Changed to use standard env var name. Documented that custom `proxyConfig.configYaml.fromString` requires manual callback setup.
+
+**Files Modified**: `operator/controllers/modelapi_controller.go`, `docs/operator/telemetry.md`
+
+### 4. TelemetryConfig Merge (Field-Wise)
+
+**Problem**: `MergeTelemetryConfig()` returned component OR global config, not merged fields. Users couldn't set enabled at component level while inheriting global endpoint.
+
+**Fix**: Implemented field-wise merge:
+- If component sets enabled but no endpoint, inherit global endpoint
+- Added `IsTelemetryConfigValid()` validation function
+- Controllers emit warning when enabled=true but endpoint empty
+
+**Files Modified**: `operator/pkg/util/telemetry.go`, `operator/controllers/agent_controller.go`, `operator/controllers/mcpserver_controller.go`, `operator/controllers/modelapi_controller.go`
+
+### 5. CRD Endpoint Defaults Inconsistency
+
+**Problem**: Helm CRDs had `endpoint: default: http://localhost:4317` which silently blackholes telemetry in-cluster.
+
+**Fix**: Removed default. Endpoint is now required when enabled (validated by controllers).
+
+**Files Modified**: `operator/chart/crds/*.yaml`
+
+### 6. Metric Names/Labels in Docs
+
+**Problem**: Docs said `kaos.agent.requests`, code emits `kaos.requests`. Docs said seconds, code uses milliseconds.
+
+**Fix**: Updated docs to match actual implementation:
+- Metrics: `kaos.requests`, `kaos.request.duration`, `kaos.tool.duration`, etc.
+- Labels: `success: "true"/"false"` (not `status`)
+- Duration: milliseconds
+
+**Files Modified**: `docs/operator/telemetry.md`
+
+### 7. SpanState Token Type Annotation
+
+**Problem**: Type checker error - `SpanState.token` typed as `object` but needs `Token[Context]`.
+
+**Fix**: Added proper type annotation with necessary imports.
+
+**Files Modified**: `python/telemetry/manager.py`
+
+### 8. Helm values.yaml Regeneration
+
+**Problem**: `make helm` overwrites values.yaml, removing custom values for defaultImages, gatewayAPI, gateway.defaultTimeouts, and telemetry.
+
+**Fix**: Restored all required values after CRD regeneration.
+
+**Files Modified**: `operator/chart/values.yaml`
+
+## Log Level Configuration (Round 3)
+
+### Overview
+Added centralized log level configuration to KAOS that applies to both control plane (operator) and data plane (Agent, MCPServer, ModelAPI) components.
+
+### Changes Made
+
+#### 1. Helm Chart Configuration
+Added `logLevel` parameter to `values.yaml`:
+```yaml
+logLevel: INFO  # TRACE, DEBUG, INFO, WARNING, ERROR
+```
+
+The operator reads this as `DEFAULT_LOG_LEVEL` environment variable.
+
+**Files Modified**: `operator/chart/values.yaml`, `operator/chart/templates/operator-configmap.yaml`
+
+#### 2. Python LOG_LEVEL Standardization
+Standardized on `LOG_LEVEL` environment variable across all Python components:
+- Agent: `LOG_LEVEL` (fallback: `AGENT_LOG_LEVEL`)
+- MCPServer: `LOG_LEVEL` (fallback: `MCP_LOG_LEVEL`)
+
+**Files Modified**: `python/agent/server.py`, `python/mcptools/server.py`
+
+#### 3. Operator LOG_LEVEL Propagation
+Updated all controllers to pass `LOG_LEVEL` to data plane pods:
+- `util.GetDefaultLogLevel()` - Gets default from `DEFAULT_LOG_LEVEL` env var
+- `util.BuildLogLevelEnvVar()` - Builds LOG_LEVEL env var if not set by user
+
+**Files Modified**: `operator/pkg/util/telemetry.go`, `operator/controllers/agent_controller.go`, `operator/controllers/mcpserver_controller.go`
+
+#### 4. LiteLLM and Ollama Log Level Mapping
+Added log level mapping for external components:
+- **LiteLLM**: Maps `LOG_LEVEL` to `LITELLM_LOG` (TRACE→DEBUG, DEBUG→DEBUG, INFO→INFO, etc.)
+- **Ollama**: Maps `LOG_LEVEL` to `OLLAMA_DEBUG` (TRACE→2, DEBUG→1, INFO/WARNING/ERROR→0)
+
+**Files Modified**: `operator/controllers/modelapi_controller.go`
+
+#### 5. Comprehensive Debug Logging
+Added debug logging throughout the Python codebase:
+- **Startup config**: OTEL endpoint, service name, resource attributes, log level
+- **Model calls**: Message count, response length, failures
+- **Tool execution**: Tool name, argument keys, success/failure
+- **Delegation**: Target agent, task length, result length
+
+**Files Modified**: `python/agent/server.py`, `python/mcptools/server.py`, `python/agent/client.py`
+
+#### 6. Trace Context in Memory Events
+Memory events now automatically include trace context when OTel is enabled:
+```python
+# Added to LocalMemory.create_event()
+if is_otel_enabled():
+    trace_ctx = get_current_trace_context()  # Returns trace_id, span_id
+    event_metadata.update(trace_ctx)
+```
+
+Enables querying events by trace_id for debugging agent workflows.
+
+**Files Modified**: `python/agent/memory.py`, `python/telemetry/manager.py`
+
+### Log Level Mapping Summary
+
+| KAOS Level | Python | LiteLLM | Ollama |
+|------------|--------|---------|--------|
+| TRACE | DEBUG | DEBUG | OLLAMA_DEBUG=2 |
+| DEBUG | DEBUG | DEBUG | OLLAMA_DEBUG=1 |
+| INFO | INFO | INFO | (default) |
+| WARNING | WARNING | WARNING | (default) |
+| ERROR | ERROR | ERROR | (default) |
+
+## CI Status
+- All tests passing (Python, Go unit tests, E2E)
+- Final commit: 3dde9b7
+
+## Files Changed
+
+### Python
+- `python/telemetry/manager.py` - Core OTel implementation with trace context getter
+- `python/agent/client.py` - Agent with debug logging for model/tool/delegation
+- `python/agent/server.py` - Server with LOG_LEVEL support and OTEL startup logging
+- `python/agent/memory.py` - Memory events with trace context correlation
+- `python/mcptools/server.py` - MCPServer with LOG_LEVEL support and OTEL logging
+- `python/tests/test_telemetry.py` - Updated tests
+
+### Go Operator
+- `operator/api/v1alpha1/agent_types.go` - Removed endpoint default
+- `operator/api/v1alpha1/modelapi_types.go` - Added Telemetry field
+- `operator/controllers/agent_controller.go` - LOG_LEVEL propagation
+- `operator/controllers/mcpserver_controller.go` - LOG_LEVEL propagation
+- `operator/controllers/modelapi_controller.go` - LiteLLM/Ollama log level mapping
+- `operator/pkg/util/telemetry.go` - Log level utilities
+
+### Helm Chart
+- `operator/chart/values.yaml` - Added logLevel parameter
+- `operator/chart/templates/operator-configmap.yaml` - DEFAULT_LOG_LEVEL env var
+
+### Documentation
+- `docs/operator/telemetry.md` - Updated with ModelAPI telemetry, corrected env var descriptions
+
diff --git a/python/agent/client.py b/python/agent/client.py
index 40e68c0..899fb7a 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -153,6 +153,7 @@ class RemoteAgent:
             return data["choices"][0]["message"]["content"]
         except Exception as e:
             self._active = False
+            logger.error(f"RemoteAgent {self.name} request failed: {type(e).__name__}: {e}")
             raise RuntimeError(f"Agent {self.name}: {type(e).__name__}: {e}")
 
     async def close(self):
@@ -525,7 +526,7 @@ class Agent:
         except Exception as e:
             failed = True
             self._otel.span_failure(e)
-            logger.debug(f"Model call failed: {type(e).__name__}: {e}")
+            logger.error(f"Model call failed: {type(e).__name__}: {e}")
             raise
         finally:
             if not failed:
@@ -556,7 +557,7 @@ class Agent:
         except Exception as e:
             failed = True
             self._otel.span_failure(e)
-            logger.debug(f"Tool {tool_name} failed: {type(e).__name__}: {e}")
+            logger.error(f"Tool {tool_name} failed: {type(e).__name__}: {e}")
             raise
         finally:
             if not failed:
@@ -588,7 +589,7 @@ class Agent:
         except Exception as e:
             failed = True
             self._otel.span_failure(e)
-            logger.debug(f"Delegation to {agent_name} failed: {type(e).__name__}: {e}")
+            logger.error(f"Delegation to {agent_name} failed: {type(e).__name__}: {e}")
             raise
         finally:
             if not failed:
```

---

### Commit: 60e7efb
**Subject:** fix(telemetry): correct log/span ordering for proper trace correlation
**Date:** 2026-01-29 08:42:51 +0100
**Author:** Alejandro Saucedo

- Move logger.error BEFORE span_failure() so logs have trace context
- Move logger.debug AFTER span_begin() so logs include span context
- span_failure() detaches context, logs after it lose correlation
- Update REPORT.md with correct log/span ordering pattern

Fixes: Logs were being emitted after span context was detached,
resulting in log entries without trace_id/span_id correlation.


```diff
 .gitignore             |   1 -
 REPORT.md              | 138 +++++++++++++++++++++++++++++++++++++++++++++++++
 python/agent/client.py |  12 ++---
 3 files changed, 144 insertions(+), 7 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/.gitignore b/.gitignore
index 62e7c4d..ef00d0a 100644
--- a/.gitignore
+++ b/.gitignore
@@ -72,7 +72,6 @@ hack/kind-e2e-values.yaml
 
 # Analysis reports (local only, not committed)
 MEMORY_REPORT.md
-REPORT.md
 
 # Planning documents (local only, not committed)
 PLAN.md
diff --git a/REPORT.md b/REPORT.md
new file mode 100644
index 0000000..6e3caa7
--- /dev/null
+++ b/REPORT.md
@@ -0,0 +1,138 @@
+# OTEL Logging Correlation Analysis
+
+## Executive Summary
+
+This report analyzes the current implementation of logging for OpenTelemetry (OTEL) trace correlation in KAOS, specifically examining:
+1. Whether `logger.debug` statements are appropriate for OTEL correlation
+2. Whether memory events should replace or supplement debug logs
+3. Whether exceptions are properly logged with `logger.error` for correlation
+4. Whether log statements are correctly ordered relative to span lifecycle
+
+**Key Findings:**
+- ~~Current `logger.debug` for failures is **problematic**~~ → Fixed: now uses `logger.error`
+- Memory events and logs serve **different purposes** - both should be retained
+- ~~Several exception paths **lack `logger.error`** calls~~ → Fixed: added `logger.error`
+- ~~Log statements were placed **after** span ends~~ → Fixed: logs now precede `span_failure`
+
+---
+
+## Issue 1: Debug Logging in Exception Paths (FIXED)
+
+**Problem:** The implementation used `logger.debug` for failures:
+
+```python
+# BEFORE (problematic)
+except Exception as e:
+    failed = True
+    self._otel.span_failure(e)
+    logger.debug(f"Model call failed: ...")  # ❌ DEBUG level, invisible at INFO
+    raise
+```
+
+**Solution:** Changed to `logger.error`:
+
+```python
+# AFTER (fixed)
+except Exception as e:
+    failed = True
+    logger.error(f"Model call failed: ...")  # ✅ ERROR level, always visible
+    self._otel.span_failure(e)
+    raise
+```
+
+---
+
+## Issue 2: Log/Span Ordering (FIXED)
+
+**Problem:** Logs were emitted AFTER `span_failure()` which detaches the context:
+
+```python
+# BEFORE (problematic)
+except Exception as e:
+    self._otel.span_failure(e)  # ← Context detached here
+    logger.error(...)           # ← Log has NO trace correlation!
+```
+
+The `span_failure()` method calls `otel_context.detach(state.token)`, removing the span from the current context. Any logs after this point lose their trace_id/span_id correlation.
+
+**Solution:** Reordered to log BEFORE span ends:
+
+```python
+# AFTER (fixed)
+except Exception as e:
+    logger.error(...)           # ✅ Log while span is active
+    self._otel.span_failure(e)  # ← Context detached after log
+```
+
+Similarly for span_begin - logs should come AFTER span starts:
+
+```python
+# BEFORE (problematic)
+logger.debug(f"Executing tool: {tool_name}")  # ← No span context yet
+self._otel.span_begin(f"tool.{tool_name}")
+
+# AFTER (fixed)
+self._otel.span_begin(f"tool.{tool_name}")    # ← Span starts
+logger.debug(f"Executing tool: {tool_name}")  # ✅ Log with span context
+```
+
+**Affected Locations (all fixed):**
+| Method | Issue | Fix |
+|--------|-------|-----|
+| `_call_model` | logger.error after span_failure | Reordered |
+| `_execute_tool` | logger.debug before span_begin, logger.error after span_failure | Reordered both |
+| `_execute_delegation` | logger.debug before span_begin, logger.error after span_failure | Reordered both |
+| `process_message` | logger.error after span_failure | Reordered |
+
+---
+
+## Issue 3: Memory Events vs Debug Logs
+
+**Analysis:** These serve different purposes and should both be retained.
+
+| Aspect | Memory Events | Debug Logs |
+|--------|--------------|------------|
+| **Purpose** | Agent working memory for context | Observability output |
+| **Visibility** | Internal (agent use) | External (SigNoz, logs) |
+| **Query-ability** | Limited (in-memory) | Searchable in log systems |
+| **Trace Correlation** | Via metadata field | Via log format with trace_id |
+
+**Recommendation:** Keep both - no changes needed.
+
+---
+
+## Correct Log/Span Ordering Pattern
+
+```python
+# Correct pattern for traced operations:
+
+self._otel.span_begin("operation.name")       # 1. Start span (attaches context)
+logger.debug("Starting operation...")          # 2. Log with span context
+
+failed = False
+try:
+    result = do_work()
+    logger.debug("Operation succeeded")        # 3. Success log (still in span)
+    return result
+except Exception as e:
+    failed = True
+    logger.error(f"Operation failed: {e}")     # 4. Error log (still in span)
+    self._otel.span_failure(e)                 # 5. End span with error
+    raise
+finally:
+    if not failed:
+        self._otel.span_success()              # 6. End span with success
+```
+
+**Key Rule:** All logs must occur BETWEEN `span_begin` and `span_success/span_failure`.
+
+---
+
+## Validation Checklist
+
+- [x] `logger.error` used for all failure paths
+- [x] Logs occur BEFORE `span_failure()` (while context active)
+- [x] Logs occur AFTER `span_begin()` (after context attached)
+- [x] Memory events still include trace context
+- [x] All unit tests pass
+- [x] Linting passes
diff --git a/python/agent/client.py b/python/agent/client.py
index 899fb7a..22f6156 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -388,9 +388,9 @@ class Agent:
 
         except Exception as e:
             span_failed = True
-            self._otel.span_failure(e)
             error_msg = f"Error processing message: {str(e)}"
             logger.error(error_msg)
+            self._otel.span_failure(e)
             error_event = self.memory.create_event("error", error_msg)
             await self.memory.add_event(session_id, error_event)
             yield f"Sorry, I encountered an error: {str(e)}"
@@ -525,8 +525,8 @@ class Agent:
             return content
         except Exception as e:
             failed = True
-            self._otel.span_failure(e)
             logger.error(f"Model call failed: {type(e).__name__}: {e}")
+            self._otel.span_failure(e)
             raise
         finally:
             if not failed:
@@ -534,7 +534,6 @@ class Agent:
 
     async def _execute_tool(self, tool_name: str, tool_args: Dict[str, Any]) -> Any:
         """Execute a tool with tracing."""
-        logger.debug(f"Executing tool: {tool_name}, args: {list(tool_args.keys())}")
         self._otel.span_begin(
             f"tool.{tool_name}",
             kind=SpanKind.CLIENT,
@@ -542,6 +541,7 @@ class Agent:
             metric_kind="tool",
             metric_attrs={"tool": tool_name},
         )
+        logger.debug(f"Executing tool: {tool_name}, args: {list(tool_args.keys())}")
         failed = False
         try:
             tool_result = None
@@ -556,8 +556,8 @@ class Agent:
             return tool_result
         except Exception as e:
             failed = True
-            self._otel.span_failure(e)
             logger.error(f"Tool {tool_name} failed: {type(e).__name__}: {e}")
+            self._otel.span_failure(e)
             raise
         finally:
             if not failed:
@@ -571,7 +571,6 @@ class Agent:
         session_id: str,
     ) -> str:
         """Execute delegation to a sub-agent with tracing."""
-        logger.debug(f"Delegating to sub-agent: {agent_name}, task length: {len(task)}")
         self._otel.span_begin(
             f"delegate.{agent_name}",
             kind=SpanKind.CLIENT,
@@ -579,6 +578,7 @@ class Agent:
             metric_kind="delegation",
             metric_attrs={"target": agent_name},
         )
+        logger.debug(f"Delegating to sub-agent: {agent_name}, task length: {len(task)}")
         failed = False
         try:
             result = await self.delegate_to_sub_agent(
@@ -588,8 +588,8 @@ class Agent:
             return result
         except Exception as e:
             failed = True
-            self._otel.span_failure(e)
             logger.error(f"Delegation to {agent_name} failed: {type(e).__name__}: {e}")
+            self._otel.span_failure(e)
             raise
         finally:
             if not failed:
```

---

### Commit: f9fccbe
**Subject:** fix(telemetry): respect LOG_LEVEL for OTEL log export
**Date:** 2026-01-29 15:25:10 +0100
**Author:** Alejandro Saucedo

- Add _get_log_level() helper to convert LOG_LEVEL env var to logging constant
- Update LoggingHandler to use configured log level instead of hardcoded INFO
- DEBUG logs now exported to OTEL collector when LOG_LEVEL=DEBUG


```diff
 python/telemetry/manager.py | 24 ++++++++++++++++++++++--
 1 file changed, 22 insertions(+), 2 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index c241ebe..2913f17 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -42,6 +42,25 @@ from opentelemetry.context import Context
 
 logger = logging.getLogger(__name__)
 
+
+def _get_log_level() -> int:
+    """Get the configured log level as a logging constant.
+
+    Reads from LOG_LEVEL env var and converts to logging.DEBUG/INFO/etc.
+    Defaults to INFO if not set or invalid.
+    """
+    level_str = os.getenv("LOG_LEVEL", "INFO").upper()
+    level_map = {
+        "TRACE": logging.DEBUG,  # Python doesn't have TRACE
+        "DEBUG": logging.DEBUG,
+        "INFO": logging.INFO,
+        "WARNING": logging.WARNING,
+        "WARN": logging.WARNING,
+        "ERROR": logging.ERROR,
+    }
+    return level_map.get(level_str, logging.INFO)
+
+
 # Semantic conventions for KAOS spans
 ATTR_AGENT_NAME = "agent.name"
 ATTR_SESSION_ID = "session.id"
@@ -212,8 +231,9 @@ def init_otel(service_name: Optional[str] = None) -> bool:
     logger_provider = LoggerProvider(resource=resource)
     logger_provider.add_log_record_processor(BatchLogRecordProcessor(otlp_log_exporter))
     otel_logs.set_logger_provider(logger_provider)
-    # Attach OTEL handler to root logger to export all logs
-    otel_handler = LoggingHandler(level=logging.INFO, logger_provider=logger_provider)
+    # Attach OTEL handler to root logger to export all logs at configured level
+    log_level = _get_log_level()
+    otel_handler = LoggingHandler(level=log_level, logger_provider=logger_provider)
     logging.getLogger().addHandler(otel_handler)
 
     logger.info(
```

---

### Commit: b9fb19b
**Subject:** feat(telemetry): include logger name in OTEL log exports
**Date:** 2026-01-29 15:27:21 +0100
**Author:** Alejandro Saucedo

- Add KaosLoggingHandler that extends LoggingHandler
- Adds logger_name attribute to log records for visibility in SigNoz
- Standard LoggingHandler excludes name from attributes (uses InstrumentationScope)
- Custom handler ensures logger name is searchable as log attribute


```diff
 python/telemetry/manager.py | 22 ++++++++++++++++++++--
 1 file changed, 20 insertions(+), 2 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 2913f17..3bf01a7 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -61,6 +61,23 @@ def _get_log_level() -> int:
     return level_map.get(level_str, logging.INFO)
 
 
+class KaosLoggingHandler(LoggingHandler):
+    """Custom LoggingHandler that adds logger name as an explicit attribute.
+
+    The standard LoggingHandler uses logger name for InstrumentationScope but
+    excludes it from log record attributes. This subclass adds it back as
+    'logger.name' for better visibility in log viewers like SigNoz.
+    """
+
+    def emit(self, record: logging.LogRecord) -> None:
+        """Emit a log record with logger name as attribute."""
+        # Add logger name as attribute before translation
+        # This is safe because we're adding to the record, not modifying reserved attrs
+        if not hasattr(record, "logger_name"):
+            record.logger_name = record.name
+        super().emit(record)
+
+
 # Semantic conventions for KAOS spans
 ATTR_AGENT_NAME = "agent.name"
 ATTR_SESSION_ID = "session.id"
@@ -231,9 +248,10 @@ def init_otel(service_name: Optional[str] = None) -> bool:
     logger_provider = LoggerProvider(resource=resource)
     logger_provider.add_log_record_processor(BatchLogRecordProcessor(otlp_log_exporter))
     otel_logs.set_logger_provider(logger_provider)
-    # Attach OTEL handler to root logger to export all logs at configured level
+    # Attach custom handler to root logger to export all logs at configured level
+    # Uses KaosLoggingHandler which adds logger.name as explicit attribute
     log_level = _get_log_level()
-    otel_handler = LoggingHandler(level=log_level, logger_provider=logger_provider)
+    otel_handler = KaosLoggingHandler(level=log_level, logger_provider=logger_provider)
     logging.getLogger().addHandler(otel_handler)
 
     logger.info(
```

---

### Commit: 07167f4
**Subject:** feat(telemetry): make HTTPX client tracing opt-in
**Date:** 2026-01-29 15:28:35 +0100
**Author:** Alejandro Saucedo

- Add OTEL_INCLUDE_HTTP_CLIENT env var (default: false)
- HTTPX instrumentation disabled by default to reduce MCP SSE noise
- httpx/httpcore/mcp.client.streamable_http loggers set to WARNING by default
- Enable HTTP client tracing with OTEL_INCLUDE_HTTP_CLIENT=true


```diff
 python/agent/server.py | 35 +++++++++++++++++++++++++++++------
 1 file changed, 29 insertions(+), 6 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/server.py b/python/agent/server.py
index e3a993a..574246a 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -85,8 +85,16 @@ def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> No
         logging.getLogger(logger_name).setLevel(log_level)
 
     # Reduce noise from third-party libraries
-    logging.getLogger("httpx").setLevel(logging.WARNING)
-    logging.getLogger("httpcore").setLevel(logging.WARNING)
+    # HTTPX/HTTPCORE: set to WARNING by default, or log_level if OTEL_INCLUDE_HTTP_CLIENT=true
+    include_http_client = os.getenv("OTEL_INCLUDE_HTTP_CLIENT", "false").lower() in (
+        "true",
+        "1",
+        "yes",
+    )
+    http_log_level = log_level if include_http_client else logging.WARNING
+    logging.getLogger("httpx").setLevel(http_log_level)
+    logging.getLogger("httpcore").setLevel(http_log_level)
+    logging.getLogger("mcp.client.streamable_http").setLevel(http_log_level)
     logging.getLogger("uvicorn.error").setLevel(log_level)
 
 
@@ -180,15 +188,30 @@ class AgentServer:
         logger.info(f"AgentServer initialized for {agent.name} on port {port}")
 
     def _setup_telemetry(self):
-        """Setup OpenTelemetry instrumentation for FastAPI."""
+        """Setup OpenTelemetry instrumentation for FastAPI.
+
+        HTTPX client tracing is disabled by default to reduce noise from MCP SSE
+        connections. Enable with OTEL_INCLUDE_HTTP_CLIENT=true.
+        """
         if is_otel_enabled():
             try:
                 from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
-                from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
 
                 FastAPIInstrumentor.instrument_app(self.app)
-                HTTPXClientInstrumentor().instrument()
-                logger.info("OpenTelemetry instrumentation enabled (FastAPI + HTTPX)")
+
+                # HTTPX instrumentation is opt-in (noisy with MCP SSE)
+                include_http_client = os.getenv("OTEL_INCLUDE_HTTP_CLIENT", "false").lower() in (
+                    "true",
+                    "1",
+                    "yes",
+                )
+                if include_http_client:
+                    from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
+
+                    HTTPXClientInstrumentor().instrument()
+                    logger.info("OpenTelemetry instrumentation enabled (FastAPI + HTTPX)")
+                else:
+                    logger.info("OpenTelemetry instrumentation enabled (FastAPI only)")
             except Exception as e:
                 logger.warning(f"Failed to enable OpenTelemetry instrumentation: {e}")
 
```

---

### Commit: 97271cc
**Subject:** feat(telemetry): move uvicorn access log control to Python code
**Date:** 2026-01-29 15:29:42 +0100
**Author:** Alejandro Saucedo

- Remove --no-access-log from Dockerfile CMD
- Add OTEL_INCLUDE_HTTP_SERVER env var (default: false)
- Uvicorn access logs disabled by default via logger level
- Enable with OTEL_INCLUDE_HTTP_SERVER=true for request logging


```diff
 python/Dockerfile      |  4 ++--
 python/agent/server.py | 10 ++++++++++
 2 files changed, 12 insertions(+), 2 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/Dockerfile b/python/Dockerfile
index eabc73a..dd1e308 100644
--- a/python/Dockerfile
+++ b/python/Dockerfile
@@ -33,5 +33,5 @@ HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
 EXPOSE 8000
 
 # Run the agent server using factory pattern
-# --no-access-log disables uvicorn access logs (less noise from /health, /ready)
-CMD ["python", "-m", "uvicorn", "agent.server:get_app", "--factory", "--host", "0.0.0.0", "--port", "8000", "--no-access-log"]
+# Access logs are controlled by OTEL_INCLUDE_HTTP_SERVER env var in Python code
+CMD ["python", "-m", "uvicorn", "agent.server:get_app", "--factory", "--host", "0.0.0.0", "--port", "8000"]
diff --git a/python/agent/server.py b/python/agent/server.py
index 574246a..28beed8 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -95,7 +95,17 @@ def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> No
     logging.getLogger("httpx").setLevel(http_log_level)
     logging.getLogger("httpcore").setLevel(http_log_level)
     logging.getLogger("mcp.client.streamable_http").setLevel(http_log_level)
+
+    # Uvicorn access logs: disabled by default, enable with OTEL_INCLUDE_HTTP_SERVER=true
+    include_http_server = os.getenv("OTEL_INCLUDE_HTTP_SERVER", "false").lower() in (
+        "true",
+        "1",
+        "yes",
+    )
     logging.getLogger("uvicorn.error").setLevel(log_level)
+    # Access logger at CRITICAL effectively disables it; at log_level enables it
+    uvicorn_access_level = log_level if include_http_server else logging.CRITICAL
+    logging.getLogger("uvicorn.access").setLevel(uvicorn_access_level)
 
 
 logger = logging.getLogger(__name__)
```

---

### Commit: d6741f7
**Subject:** docs: add OTEL logging configuration documentation
**Date:** 2026-01-29 15:30:16 +0100
**Author:** Alejandro Saucedo

- Document LOG_LEVEL, OTEL_INCLUDE_HTTP_CLIENT, OTEL_INCLUDE_HTTP_SERVER
- Explain log record attributes exported to OTEL
- Provide configuration examples for different scenarios


```diff
 REPORT.md | 96 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 96 insertions(+)
```

#### Detailed Changes

```diff
diff --git a/REPORT.md b/REPORT.md
new file mode 100644
index 0000000..33958fa
--- /dev/null
+++ b/REPORT.md
@@ -0,0 +1,96 @@
+# OTEL Logging Configuration
+
+## Overview
+
+This document describes the OpenTelemetry logging configuration options for KAOS data plane components (Agent, MCPServer).
+
+## Environment Variables
+
+### Log Level
+
+| Variable | Default | Description |
+|----------|---------|-------------|
+| `LOG_LEVEL` | `INFO` | Log level for all components (TRACE, DEBUG, INFO, WARNING, ERROR) |
+
+The log level controls:
+- Python logging output to stdout
+- OTEL log export level (DEBUG logs only exported when LOG_LEVEL=DEBUG)
+
+### HTTP Tracing Options
+
+| Variable | Default | Description |
+|----------|---------|-------------|
+| `OTEL_INCLUDE_HTTP_CLIENT` | `false` | Enable HTTPX client tracing and logging |
+| `OTEL_INCLUDE_HTTP_SERVER` | `false` | Enable uvicorn access logs |
+
+#### OTEL_INCLUDE_HTTP_CLIENT
+
+When **disabled** (default):
+- HTTPX client calls are not traced (no spans for outgoing HTTP requests)
+- `httpx`, `httpcore`, `mcp.client.streamable_http` loggers set to WARNING
+- Reduces noise from MCP SSE connections which create many HTTP calls
+
+When **enabled** (`true`):
+- All HTTPX client calls create trace spans
+- HTTP client loggers use configured LOG_LEVEL
+- Useful for debugging network issues
+
+#### OTEL_INCLUDE_HTTP_SERVER
+
+When **disabled** (default):
+- Uvicorn access logs are suppressed (no "GET /health 200" messages)
+- Reduces noise from Kubernetes health/readiness probes
+- FastAPI request handling is still traced
+
+When **enabled** (`true`):
+- Uvicorn access logs are emitted at configured LOG_LEVEL
+- Useful for debugging incoming request issues
+
+## Log Record Attributes
+
+OTEL log exports include these attributes:
+
+| Attribute | Description |
+|-----------|-------------|
+| `logger_name` | Python logger name (e.g., `agent.client`, `mcptools.server`) |
+| `code.filepath` | Source file path |
+| `code.function` | Function name |
+| `code.lineno` | Line number |
+
+## Log/Span Correlation
+
+When OTEL is enabled, all log entries within a traced request include:
+- `trace_id` - correlates with the request trace
+- `span_id` - correlates with the active span
+
+**Important:** Logs must be emitted BEFORE `span_failure()` or AFTER `span_begin()` to include trace correlation.
+
+## Configuration Examples
+
+### Minimal OTEL (default)
+
+```bash
+# Only core agent spans, no HTTP noise
+LOG_LEVEL=INFO
+OTEL_SDK_DISABLED=false
+OTEL_SERVICE_NAME=my-agent
+OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
+```
+
+### Debug with full HTTP tracing
+
+```bash
+# Full visibility for debugging
+LOG_LEVEL=DEBUG
+OTEL_INCLUDE_HTTP_CLIENT=true
+OTEL_INCLUDE_HTTP_SERVER=true
+```
+
+### Production recommended
+
+```bash
+# Errors always visible, minimal noise
+LOG_LEVEL=INFO
+OTEL_INCLUDE_HTTP_CLIENT=false
+OTEL_INCLUDE_HTTP_SERVER=false
+```
```

---

### Commit: 31260be
**Subject:** refactor(telemetry): consolidate log level functions and make FastAPI instrumentation opt-in
**Date:** 2026-01-29 16:01:26 +0100
**Author:** Alejandro Saucedo

- Moved get_log_level() to telemetry/manager.py as single source of truth
- Added get_log_level_int() for numeric level conversion
- Both support AGENT_LOG_LEVEL fallback for backwards compatibility
- FastAPIInstrumentor now opt-in via OTEL_INCLUDE_HTTP_SERVER (default: false)
- Updated REPORT.md documentation to reflect FastAPI instrumentation behavior


```diff
 REPORT.md                   |  3 ++-
 python/agent/server.py      | 37 ++++++++++++++++++++++++-------------
 python/telemetry/manager.py | 18 ++++++++++++++----
 3 files changed, 40 insertions(+), 18 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/REPORT.md b/REPORT.md
index 33958fa..7be3739 100644
--- a/REPORT.md
+++ b/REPORT.md
@@ -39,10 +39,11 @@ When **enabled** (`true`):
 
 When **disabled** (default):
 - Uvicorn access logs are suppressed (no "GET /health 200" messages)
+- FastAPI request handling is NOT traced (no spans for incoming HTTP requests)
 - Reduces noise from Kubernetes health/readiness probes
-- FastAPI request handling is still traced
 
 When **enabled** (`true`):
+- FastAPI incoming requests create trace spans
 - Uvicorn access logs are emitted at configured LOG_LEVEL
 - Useful for debugging incoming request issues
 
diff --git a/python/agent/server.py b/python/agent/server.py
index 28beed8..547c3de 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -24,12 +24,7 @@ from modelapi.client import ModelAPI
 from agent.client import Agent, RemoteAgent
 from agent.memory import LocalMemory
 from mcptools.client import MCPClient
-from telemetry.manager import init_otel, is_otel_enabled, should_enable_otel
-
-
-def get_log_level() -> str:
-    """Get log level from environment, preferring LOG_LEVEL over AGENT_LOG_LEVEL."""
-    return os.getenv("LOG_LEVEL", os.getenv("AGENT_LOG_LEVEL", "INFO")).upper()
+from telemetry.manager import init_otel, is_otel_enabled, should_enable_otel, get_log_level
 
 
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
@@ -200,14 +195,17 @@ class AgentServer:
     def _setup_telemetry(self):
         """Setup OpenTelemetry instrumentation for FastAPI.
 
-        HTTPX client tracing is disabled by default to reduce noise from MCP SSE
-        connections. Enable with OTEL_INCLUDE_HTTP_CLIENT=true.
+        HTTP server/client tracing is disabled by default to reduce noise.
+        Enable with OTEL_INCLUDE_HTTP_SERVER=true (FastAPI) or OTEL_INCLUDE_HTTP_CLIENT=true (HTTPX).
         """
         if is_otel_enabled():
             try:
-                from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
-
-                FastAPIInstrumentor.instrument_app(self.app)
+                # FastAPI instrumentation is opt-in (noisy with health probes)
+                include_http_server = os.getenv("OTEL_INCLUDE_HTTP_SERVER", "false").lower() in (
+                    "true",
+                    "1",
+                    "yes",
+                )
 
                 # HTTPX instrumentation is opt-in (noisy with MCP SSE)
                 include_http_client = os.getenv("OTEL_INCLUDE_HTTP_CLIENT", "false").lower() in (
@@ -215,13 +213,26 @@ class AgentServer:
                     "1",
                     "yes",
                 )
+
+                instrumentations = []
+                if include_http_server:
+                    from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
+
+                    FastAPIInstrumentor.instrument_app(self.app)
+                    instrumentations.append("FastAPI")
+
                 if include_http_client:
                     from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
 
                     HTTPXClientInstrumentor().instrument()
-                    logger.info("OpenTelemetry instrumentation enabled (FastAPI + HTTPX)")
+                    instrumentations.append("HTTPX")
+
+                if instrumentations:
+                    logger.info(
+                        f"OpenTelemetry HTTP instrumentation enabled: {', '.join(instrumentations)}"
+                    )
                 else:
-                    logger.info("OpenTelemetry instrumentation enabled (FastAPI only)")
+                    logger.info("OpenTelemetry enabled (HTTP instrumentation disabled by default)")
             except Exception as e:
                 logger.warning(f"Failed to enable OpenTelemetry instrumentation: {e}")
 
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 3bf01a7..672fbf3 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -43,13 +43,23 @@ from opentelemetry.context import Context
 logger = logging.getLogger(__name__)
 
 
-def _get_log_level() -> int:
+def get_log_level() -> str:
+    """Get the configured log level as a string.
+
+    Reads from LOG_LEVEL env var (with AGENT_LOG_LEVEL as fallback for backwards
+    compatibility) and returns the normalized level string.
+    Defaults to INFO if not set.
+    """
+    return os.getenv("LOG_LEVEL", os.getenv("AGENT_LOG_LEVEL", "INFO")).upper()
+
+
+def get_log_level_int() -> int:
     """Get the configured log level as a logging constant.
 
-    Reads from LOG_LEVEL env var and converts to logging.DEBUG/INFO/etc.
+    Converts the LOG_LEVEL string to logging.DEBUG/INFO/etc.
     Defaults to INFO if not set or invalid.
     """
-    level_str = os.getenv("LOG_LEVEL", "INFO").upper()
+    level_str = get_log_level()
     level_map = {
         "TRACE": logging.DEBUG,  # Python doesn't have TRACE
         "DEBUG": logging.DEBUG,
@@ -250,7 +260,7 @@ def init_otel(service_name: Optional[str] = None) -> bool:
     otel_logs.set_logger_provider(logger_provider)
     # Attach custom handler to root logger to export all logs at configured level
     # Uses KaosLoggingHandler which adds logger.name as explicit attribute
-    log_level = _get_log_level()
+    log_level = get_log_level_int()
     otel_handler = KaosLoggingHandler(level=log_level, logger_provider=logger_provider)
     logging.getLogger().addHandler(otel_handler)
 
```

---

### Commit: c2e88f9
**Subject:** feat(telemetry): add manual trace context propagation and getenv_bool helper
**Date:** 2026-01-29 18:14:48 +0100
**Author:** Alejandro Saucedo

- Add getenv_bool() helper to reduce boolean env var parsing boilerplate
- Add attach_context() and detach_context() methods to KaosOtelManager
- Inject trace context headers in RemoteAgent.process_message()
- Extract and attach trace context in AgentServer chat endpoint
- Refactor existing env var checks to use getenv_bool
- Update REPORT.md with distributed tracing documentation

This enables connected traces across agent delegations without noisy
HTTP spans from HTTPX/FastAPI instrumentors.


```diff
 REPORT.md                   | 38 +++++++++++++++++++++++++++++---------
 python/agent/client.py      |  7 +++++++
 python/agent/server.py      | 42 +++++++++++++++++++++---------------------
 python/telemetry/manager.py | 36 ++++++++++++++++++++++++++++++++++++
 4 files changed, 93 insertions(+), 30 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/REPORT.md b/REPORT.md
index 7be3739..90b178d 100644
--- a/REPORT.md
+++ b/REPORT.md
@@ -4,6 +4,15 @@
 
 This document describes the OpenTelemetry logging configuration options for KAOS data plane components (Agent, MCPServer).
 
+## Distributed Tracing
+
+KAOS implements **manual trace context propagation** across agent delegations:
+
+- **Outgoing requests** (RemoteAgent): Automatically injects `traceparent`/`tracestate` headers
+- **Incoming requests** (AgentServer): Automatically extracts trace context from headers
+
+This ensures traces are connected across agent-to-agent delegations **without** requiring HTTPX/FastAPI auto-instrumentation, avoiding noisy HTTP spans.
+
 ## Environment Variables
 
 ### Log Level
@@ -11,6 +20,7 @@ This document describes the OpenTelemetry logging configuration options for KAOS
 | Variable | Default | Description |
 |----------|---------|-------------|
 | `LOG_LEVEL` | `INFO` | Log level for all components (TRACE, DEBUG, INFO, WARNING, ERROR) |
+| `AGENT_LOG_LEVEL` | - | Fallback for LOG_LEVEL (backwards compatibility) |
 
 The log level controls:
 - Python logging output to stdout
@@ -20,30 +30,32 @@ The log level controls:
 
 | Variable | Default | Description |
 |----------|---------|-------------|
-| `OTEL_INCLUDE_HTTP_CLIENT` | `false` | Enable HTTPX client tracing and logging |
-| `OTEL_INCLUDE_HTTP_SERVER` | `false` | Enable uvicorn access logs |
+| `OTEL_INCLUDE_HTTP_CLIENT` | `false` | Enable HTTPX client span tracing and logging |
+| `OTEL_INCLUDE_HTTP_SERVER` | `false` | Enable FastAPI span tracing and uvicorn access logs |
 
 #### OTEL_INCLUDE_HTTP_CLIENT
 
 When **disabled** (default):
-- HTTPX client calls are not traced (no spans for outgoing HTTP requests)
+- HTTPX client calls do NOT create trace spans
+- Trace context is still propagated via headers (manual injection)
 - `httpx`, `httpcore`, `mcp.client.streamable_http` loggers set to WARNING
 - Reduces noise from MCP SSE connections which create many HTTP calls
 
 When **enabled** (`true`):
-- All HTTPX client calls create trace spans
+- All HTTPX client calls create trace spans (noisy)
 - HTTP client loggers use configured LOG_LEVEL
 - Useful for debugging network issues
 
 #### OTEL_INCLUDE_HTTP_SERVER
 
 When **disabled** (default):
+- FastAPI incoming requests do NOT create HTTP spans
+- Trace context is still extracted from headers (manual extraction)
 - Uvicorn access logs are suppressed (no "GET /health 200" messages)
-- FastAPI request handling is NOT traced (no spans for incoming HTTP requests)
 - Reduces noise from Kubernetes health/readiness probes
 
 When **enabled** (`true`):
-- FastAPI incoming requests create trace spans
+- FastAPI incoming requests create trace spans (noisy)
 - Uvicorn access logs are emitted at configured LOG_LEVEL
 - Useful for debugging incoming request issues
 
@@ -71,7 +83,7 @@ When OTEL is enabled, all log entries within a traced request include:
 ### Minimal OTEL (default)
 
 ```bash
-# Only core agent spans, no HTTP noise
+# Connected agent traces, no HTTP noise
 LOG_LEVEL=INFO
 OTEL_SDK_DISABLED=false
 OTEL_SERVICE_NAME=my-agent
@@ -81,7 +93,7 @@ OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
 ### Debug with full HTTP tracing
 
 ```bash
-# Full visibility for debugging
+# Full visibility for debugging (noisy)
 LOG_LEVEL=DEBUG
 OTEL_INCLUDE_HTTP_CLIENT=true
 OTEL_INCLUDE_HTTP_SERVER=true
@@ -90,8 +102,16 @@ OTEL_INCLUDE_HTTP_SERVER=true
 ### Production recommended
 
 ```bash
-# Errors always visible, minimal noise
+# Connected traces, errors visible, minimal noise
 LOG_LEVEL=INFO
 OTEL_INCLUDE_HTTP_CLIENT=false
 OTEL_INCLUDE_HTTP_SERVER=false
 ```
+
+## Helper Functions
+
+The `telemetry.manager` module exports:
+
+- `get_log_level()` - Get LOG_LEVEL as string
+- `get_log_level_int()` - Get LOG_LEVEL as logging constant
+- `getenv_bool(name, default)` - Parse boolean env var ("true"/"1"/"yes")
diff --git a/python/agent/client.py b/python/agent/client.py
index 22f6156..d411a0a 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -129,6 +129,8 @@ class RemoteAgent:
     ) -> str:
         """Process messages via remote agent's /v1/chat/completions.
 
+        Injects trace context headers for distributed tracing across agents.
+
         Args:
             messages: List of messages providing context. The last message
                      should have role "task-delegation" with the delegated task.
@@ -144,9 +146,14 @@ class RemoteAgent:
                 raise RuntimeError(f"Agent {self.name} unavailable at {self.card_url}")
 
         try:
+            # Inject trace context for distributed tracing
+            headers: Dict[str, str] = {}
+            KaosOtelManager.inject_context(headers)
+
             response = await self._request_client.post(
                 f"{self.card_url}/v1/chat/completions",
                 json={"model": self.name, "messages": messages, "stream": False},
+                headers=headers,
             )
             response.raise_for_status()
             data = response.json()
diff --git a/python/agent/server.py b/python/agent/server.py
index 547c3de..f83cb26 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -24,7 +24,14 @@ from modelapi.client import ModelAPI
 from agent.client import Agent, RemoteAgent
 from agent.memory import LocalMemory
 from mcptools.client import MCPClient
-from telemetry.manager import init_otel, is_otel_enabled, should_enable_otel, get_log_level
+from telemetry.manager import (
+    init_otel,
+    is_otel_enabled,
+    should_enable_otel,
+    get_log_level,
+    getenv_bool,
+    KaosOtelManager,
+)
 
 
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
@@ -81,22 +88,14 @@ def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> No
 
     # Reduce noise from third-party libraries
     # HTTPX/HTTPCORE: set to WARNING by default, or log_level if OTEL_INCLUDE_HTTP_CLIENT=true
-    include_http_client = os.getenv("OTEL_INCLUDE_HTTP_CLIENT", "false").lower() in (
-        "true",
-        "1",
-        "yes",
-    )
+    include_http_client = getenv_bool("OTEL_INCLUDE_HTTP_CLIENT", False)
     http_log_level = log_level if include_http_client else logging.WARNING
     logging.getLogger("httpx").setLevel(http_log_level)
     logging.getLogger("httpcore").setLevel(http_log_level)
     logging.getLogger("mcp.client.streamable_http").setLevel(http_log_level)
 
     # Uvicorn access logs: disabled by default, enable with OTEL_INCLUDE_HTTP_SERVER=true
-    include_http_server = os.getenv("OTEL_INCLUDE_HTTP_SERVER", "false").lower() in (
-        "true",
-        "1",
-        "yes",
-    )
+    include_http_server = getenv_bool("OTEL_INCLUDE_HTTP_SERVER", False)
     logging.getLogger("uvicorn.error").setLevel(log_level)
     # Access logger at CRITICAL effectively disables it; at log_level enables it
     uvicorn_access_level = log_level if include_http_server else logging.CRITICAL
@@ -201,18 +200,10 @@ class AgentServer:
         if is_otel_enabled():
             try:
                 # FastAPI instrumentation is opt-in (noisy with health probes)
-                include_http_server = os.getenv("OTEL_INCLUDE_HTTP_SERVER", "false").lower() in (
-                    "true",
-                    "1",
-                    "yes",
-                )
+                include_http_server = getenv_bool("OTEL_INCLUDE_HTTP_SERVER", False)
 
                 # HTTPX instrumentation is opt-in (noisy with MCP SSE)
-                include_http_client = os.getenv("OTEL_INCLUDE_HTTP_CLIENT", "false").lower() in (
-                    "true",
-                    "1",
-                    "yes",
-                )
+                include_http_client = getenv_bool("OTEL_INCLUDE_HTTP_CLIENT", False)
 
                 instrumentations = []
                 if include_http_server:
@@ -377,7 +368,13 @@ class AgentServer:
 
             The agent decides when to delegate or call tools based on model response.
             Server only routes requests to the agent for processing.
+            Extracts trace context from incoming headers for distributed tracing.
             """
+            # Extract trace context from incoming headers for distributed tracing
+            headers = dict(request.headers)
+            parent_ctx = KaosOtelManager.extract_context(headers)
+            ctx_token = KaosOtelManager.attach_context(parent_ctx)
+
             try:
                 body = await request.json()
 
@@ -410,6 +407,9 @@ class AgentServer:
             except Exception as e:
                 logger.error(f"Chat completion error: {e}")
                 raise HTTPException(status_code=500, detail=str(e))
+            finally:
+                # Detach the parent context
+                KaosOtelManager.detach_context(ctx_token)
 
     async def _complete_chat_completion(self, messages: list, model_name: str) -> JSONResponse:
         """Handle non-streaming chat completion.
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 672fbf3..8782e60 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -71,6 +71,29 @@ def get_log_level_int() -> int:
     return level_map.get(level_str, logging.INFO)
 
 
+def getenv_bool(name: str, default: bool = False) -> bool:
+    """Get a boolean value from an environment variable.
+
+    Args:
+        name: Environment variable name
+        default: Default value if not set (default: False)
+
+    Returns:
+        True if the env var is set to 'true', '1', or 'yes' (case-insensitive)
+        False if set to 'false', '0', or 'no'
+        default if not set or unrecognized value
+    """
+    value = os.getenv(name)
+    if value is None:
+        return default
+    value = value.lower()
+    if value in ("true", "1", "yes"):
+        return True
+    if value in ("false", "0", "no"):
+        return False
+    return default
+
+
 class KaosLoggingHandler(LoggingHandler):
     """Custom LoggingHandler that adds logger name as an explicit attribute.
 
@@ -520,3 +543,16 @@ class KaosOtelManager:
     def extract_context(carrier: Dict[str, str]) -> Context:
         """Extract trace context from headers."""
         return extract(carrier)
+
+    @staticmethod
+    def attach_context(ctx: Context) -> Token[Context]:
+        """Attach a context to make it current.
+
+        Returns a token that should be passed to detach_context() when done.
+        """
+        return otel_context.attach(ctx)
+
+    @staticmethod
+    def detach_context(token: Token[Context]) -> None:
+        """Detach a previously attached context."""
+        otel_context.detach(token)
```

---

### Commit: 8ff1a09
**Subject:** refactor(telemetry): add extract_and_attach_context helper for simpler trace propagation
**Date:** 2026-01-29 19:27:54 +0100
**Author:** Alejandro Saucedo

- Added extract_and_attach_context() that combines extract + attach in one call
- Handles dict conversion for Starlette Headers automatically
- Simplified server.py chat endpoint to use new helper


```diff
 python/agent/server.py      |  7 ++-----
 python/telemetry/manager.py | 18 ++++++++++++++++++
 2 files changed, 20 insertions(+), 5 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/server.py b/python/agent/server.py
index f83cb26..e3c45ec 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -370,10 +370,8 @@ class AgentServer:
             Server only routes requests to the agent for processing.
             Extracts trace context from incoming headers for distributed tracing.
             """
-            # Extract trace context from incoming headers for distributed tracing
-            headers = dict(request.headers)
-            parent_ctx = KaosOtelManager.extract_context(headers)
-            ctx_token = KaosOtelManager.attach_context(parent_ctx)
+            # Extract and attach trace context for distributed tracing
+            ctx_token = KaosOtelManager.extract_and_attach_context(request.headers)
 
             try:
                 body = await request.json()
@@ -408,7 +406,6 @@ class AgentServer:
                 logger.error(f"Chat completion error: {e}")
                 raise HTTPException(status_code=500, detail=str(e))
             finally:
-                # Detach the parent context
                 KaosOtelManager.detach_context(ctx_token)
 
     async def _complete_chat_completion(self, messages: list, model_name: str) -> JSONResponse:
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 8782e60..4587571 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -556,3 +556,21 @@ class KaosOtelManager:
     def detach_context(token: Token[Context]) -> None:
         """Detach a previously attached context."""
         otel_context.detach(token)
+
+    @staticmethod
+    def extract_and_attach_context(headers: Any) -> Token[Context]:
+        """Extract trace context from headers and attach it as current context.
+
+        Convenience method that combines extract_context + attach_context.
+        Returns a token that must be passed to detach_context() in finally block.
+
+        Args:
+            headers: HTTP headers (dict, starlette Headers, or any mapping)
+
+        Returns:
+            Token for detaching context in finally block
+        """
+        # Convert to dict if needed (handles Starlette Headers, etc.)
+        carrier = dict(headers) if not isinstance(headers, dict) else headers
+        ctx = extract(carrier)
+        return otel_context.attach(ctx)
```

---

### Commit: 3826f82
**Subject:** refactor(telemetry): make KaosOtelManager a module-level singleton
**Date:** 2026-01-29 19:30:30 +0100
**Author:** Alejandro Saucedo

- Add _get_service_name() to read from OTEL_SERVICE_NAME/AGENT_NAME
- Add get_instance() class method for singleton access
- Create module-level 'otel' variable for easy import
- Update Agent class to use global 'otel' instead of self._otel
- Service name is now read from env vars, not passed to constructor

Usage: from telemetry.manager import otel


```diff
 python/agent/client.py      | 34 ++++++++++++++++------------------
 python/telemetry/manager.py | 37 +++++++++++++++++++++++++++++++------
 2 files changed, 47 insertions(+), 24 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/client.py b/python/agent/client.py
index d411a0a..4f17d2c 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -23,6 +23,7 @@ from modelapi.client import ModelAPI
 from agent.memory import LocalMemory, NullMemory
 from mcptools.client import MCPClient
 from telemetry.manager import (
+    otel,
     KaosOtelManager,
     ATTR_SESSION_ID,
     ATTR_MODEL_NAME,
@@ -201,9 +202,6 @@ class Agent:
         self.memory_context_limit = memory_context_limit
         self.memory_enabled = memory_enabled
 
-        # Telemetry manager (lightweight, always created - no-ops if OTel disabled)
-        self._otel = KaosOtelManager(name)
-
         logger.info(f"Agent initialized: {name}")
 
     async def _get_tools_prompt(self) -> Optional[str]:
@@ -345,7 +343,7 @@ class Agent:
             "stream": stream,
             ATTR_SESSION_ID: session_id,
         }
-        self._otel.span_begin(
+        otel.span_begin(
             "agent.agentic_loop",
             attrs=span_attrs,
             metric_kind="request",
@@ -397,13 +395,13 @@ class Agent:
             span_failed = True
             error_msg = f"Error processing message: {str(e)}"
             logger.error(error_msg)
-            self._otel.span_failure(e)
+            otel.span_failure(e)
             error_event = self.memory.create_event("error", error_msg)
             await self.memory.add_event(session_id, error_event)
             yield f"Sorry, I encountered an error: {str(e)}"
         finally:
             if not span_failed:
-                self._otel.span_success()
+                otel.span_success()
 
     async def _agentic_loop(
         self,
@@ -417,7 +415,7 @@ class Agent:
 
             # Start step span
             step_attrs = {"step": step + 1, "max_steps": self.max_steps}
-            self._otel.span_begin(f"agent.step.{step + 1}", attrs=step_attrs)
+            otel.span_begin(f"agent.step.{step + 1}", attrs=step_attrs)
             # Use failed flag pattern to ensure spans close on continue/return/yield
             step_failed = False
             try:
@@ -504,11 +502,11 @@ class Agent:
 
             except Exception as e:
                 step_failed = True
-                self._otel.span_failure(e)
+                otel.span_failure(e)
                 raise
             finally:
                 if not step_failed:
-                    self._otel.span_success()
+                    otel.span_success()
 
         # Max steps reached
         max_steps_msg = f"Reached maximum reasoning steps ({self.max_steps})"
@@ -517,7 +515,7 @@ class Agent:
 
     async def _call_model(self, messages: List[Dict[str, str]], model_name: str) -> str:
         """Call the model API with tracing."""
-        self._otel.span_begin(
+        otel.span_begin(
             "model.inference",
             kind=SpanKind.CLIENT,
             attrs={ATTR_MODEL_NAME: model_name},
@@ -533,15 +531,15 @@ class Agent:
         except Exception as e:
             failed = True
             logger.error(f"Model call failed: {type(e).__name__}: {e}")
-            self._otel.span_failure(e)
+            otel.span_failure(e)
             raise
         finally:
             if not failed:
-                self._otel.span_success()
+                otel.span_success()
 
     async def _execute_tool(self, tool_name: str, tool_args: Dict[str, Any]) -> Any:
         """Execute a tool with tracing."""
-        self._otel.span_begin(
+        otel.span_begin(
             f"tool.{tool_name}",
             kind=SpanKind.CLIENT,
             attrs={ATTR_TOOL_NAME: tool_name},
@@ -564,11 +562,11 @@ class Agent:
         except Exception as e:
             failed = True
             logger.error(f"Tool {tool_name} failed: {type(e).__name__}: {e}")
-            self._otel.span_failure(e)
+            otel.span_failure(e)
             raise
         finally:
             if not failed:
-                self._otel.span_success()
+                otel.span_success()
 
     async def _execute_delegation(
         self,
@@ -578,7 +576,7 @@ class Agent:
         session_id: str,
     ) -> str:
         """Execute delegation to a sub-agent with tracing."""
-        self._otel.span_begin(
+        otel.span_begin(
             f"delegate.{agent_name}",
             kind=SpanKind.CLIENT,
             attrs={ATTR_DELEGATION_TARGET: agent_name},
@@ -596,11 +594,11 @@ class Agent:
         except Exception as e:
             failed = True
             logger.error(f"Delegation to {agent_name} failed: {type(e).__name__}: {e}")
-            self._otel.span_failure(e)
+            otel.span_failure(e)
             raise
         finally:
             if not failed:
-                self._otel.span_success()
+                otel.span_success()
 
     async def delegate_to_sub_agent(
         self,
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 4587571..86e626c 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -295,14 +295,22 @@ def init_otel(service_name: Optional[str] = None) -> bool:
     return True
 
 
+def _get_service_name() -> str:
+    """Get service name from environment variables."""
+    return os.getenv("OTEL_SERVICE_NAME", os.getenv("AGENT_NAME", "kaos-service"))
+
+
 class KaosOtelManager:
     """Lightweight helper for creating spans and recording metrics.
 
     Uses inline span management via span_begin/span_success/span_failure instead
     of context managers. Timing is handled internally via contextvars.
 
+    Can be used as a singleton via the module-level `otel` instance, or
+    instantiated with a specific service name for tests.
+
     Example:
-        otel = KaosOtelManager("my-agent")
+        from telemetry.manager import otel
         otel.span_begin("process_request", attrs={"session.id": "abc123"})
         try:
             # do work
@@ -314,15 +322,18 @@ class KaosOtelManager:
             otel.span_success()
     """
 
-    def __init__(self, service_name: str):
+    # Singleton instance (lazy initialized)
+    _instance: Optional["KaosOtelManager"] = None
+
+    def __init__(self, service_name: Optional[str] = None):
         """Initialize manager with service context.
 
         Args:
-            service_name: Name of the service (e.g., agent name)
+            service_name: Name of the service (reads from OTEL_SERVICE_NAME if not provided)
         """
-        self.service_name = service_name
-        self._tracer = trace.get_tracer(f"kaos.{service_name}")
-        self._meter = metrics.get_meter(f"kaos.{service_name}")
+        self.service_name = service_name or _get_service_name()
+        self._tracer = trace.get_tracer(f"kaos.{self.service_name}")
+        self._meter = metrics.get_meter(f"kaos.{self.service_name}")
 
         # Lazily initialized metrics
         self._request_counter: Optional[metrics.Counter] = None
@@ -574,3 +585,17 @@ class KaosOtelManager:
         carrier = dict(headers) if not isinstance(headers, dict) else headers
         ctx = extract(carrier)
         return otel_context.attach(ctx)
+
+    @classmethod
+    def get_instance(cls) -> "KaosOtelManager":
+        """Get or create the singleton instance.
+
+        Service name is read from OTEL_SERVICE_NAME or AGENT_NAME env var.
+        """
+        if cls._instance is None:
+            cls._instance = cls()
+        return cls._instance
+
+
+# Module-level singleton for easy import: `from telemetry.manager import otel`
+otel = KaosOtelManager.get_instance()
```

---

### Commit: 1d52e86
**Subject:** feat(telemetry): add OTEL instrumentation to MCPServer and MCPClient
**Date:** 2026-01-29 19:32:31 +0100
**Author:** Alejandro Saucedo

- MCPServer: use shared get_log_level and getenv_bool from telemetry.manager
- MCPServer: add OTEL_INCLUDE_HTTP_CLIENT/SERVER env var support
- MCPClient: add span tracing for tool calls with mcp.tool.{name} spans
- MCPClient: add DEBUG logging for tool calls and results
- Both now respect the same telemetry configuration as AgentServer


```diff
 python/mcptools/client.py | 30 +++++++++++++++++++++++++++++-
 python/mcptools/server.py | 17 ++++++++++-------
 2 files changed, 39 insertions(+), 8 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/mcptools/client.py b/python/mcptools/client.py
index 0097a20..1f9815f 100644
--- a/python/mcptools/client.py
+++ b/python/mcptools/client.py
@@ -3,6 +3,7 @@ MCP Client using the official MCP SDK for protocol-compliant communication.
 
 This client uses the MCP SDK's Streamable HTTP client to connect to any
 MCP-compliant server (FastMCP servers, external MCP servers, etc.).
+Instrumented with OpenTelemetry for distributed tracing.
 """
 
 import logging
@@ -13,6 +14,8 @@ from contextlib import asynccontextmanager
 from mcp import ClientSession
 from mcp.client.streamable_http import streamable_http_client
 from mcp import types as mcp_types
+from telemetry.manager import otel, ATTR_TOOL_NAME
+from opentelemetry.trace import SpanKind
 
 logger = logging.getLogger(__name__)
 
@@ -103,7 +106,7 @@ class MCPClient:
             return False
 
     async def call_tool(self, name: str, args: Optional[Dict[str, Any]] = None) -> Any:
-        """Call a tool on the MCP server.
+        """Call a tool on the MCP server with tracing.
 
         Args:
             name: Name of the tool to call
@@ -123,6 +126,21 @@ class MCPClient:
         if name not in self._tools:
             raise ValueError(f"Tool '{name}' not found. Available: {list(self._tools.keys())}")
 
+        # Start span for MCP tool call
+        otel.span_begin(
+            f"mcp.tool.{name}",
+            kind=SpanKind.CLIENT,
+            attrs={
+                ATTR_TOOL_NAME: name,
+                "mcp.server": self.name,
+                "mcp.url": self._mcp_url,
+            },
+            metric_kind="tool",
+            metric_attrs={"tool": name, "server": self.name},
+        )
+        logger.debug(f"MCPClient calling tool: {name} with args: {args}")
+        failed = False
+
         try:
             async with self._connect() as session:
                 result = await session.call_tool(name, args or {})
@@ -130,11 +148,15 @@ class MCPClient:
                 # Extract result from CallToolResult
                 # Prefer structured content if available
                 if result.structuredContent:
+                    logger.debug(f"MCPClient tool {name} returned structured content")
                     return result.structuredContent
                 elif result.content:
                     # Return text content from first content block
                     for content in result.content:
                         if hasattr(content, "text"):
+                            logger.debug(
+                                f"MCPClient tool {name} returned text: {content.text[:100]}..."
+                            )
                             return {"result": content.text}
                     return {"result": str(result.content)}
                 else:
@@ -142,7 +164,13 @@ class MCPClient:
 
         except Exception as e:
             self._active = False
+            failed = True
+            logger.error(f"MCPClient tool {name} failed: {type(e).__name__}: {e}")
+            otel.span_failure(e)
             raise RuntimeError(f"Tool {name}: {type(e).__name__}: {e}")
+        finally:
+            if not failed:
+                otel.span_success()
 
     def get_tools(self) -> List[Tool]:
         """Get list of discovered tools."""
diff --git a/python/mcptools/server.py b/python/mcptools/server.py
index fe06cd6..61604ff 100644
--- a/python/mcptools/server.py
+++ b/python/mcptools/server.py
@@ -11,10 +11,7 @@ from pydantic_settings import BaseSettings
 from starlette.routing import Route
 from starlette.responses import JSONResponse
 
-
-def get_log_level() -> str:
-    """Get log level from environment, preferring LOG_LEVEL over MCP_LOG_LEVEL."""
-    return os.getenv("LOG_LEVEL", os.getenv("MCP_LOG_LEVEL", "INFO")).upper()
+from telemetry.manager import get_log_level, getenv_bool
 
 
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
@@ -60,10 +57,16 @@ def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> No
     for logger_name in ["mcptools", "mcptools.server", "mcptools.client"]:
         logging.getLogger(logger_name).setLevel(log_level)
 
-    # Reduce noise from third-party libraries
-    logging.getLogger("httpx").setLevel(logging.WARNING)
-    logging.getLogger("httpcore").setLevel(logging.WARNING)
+    # Reduce noise from third-party libraries based on OTEL_INCLUDE_HTTP_* settings
+    include_http_client = getenv_bool("OTEL_INCLUDE_HTTP_CLIENT", False)
+    http_log_level = log_level if include_http_client else logging.WARNING
+    logging.getLogger("httpx").setLevel(http_log_level)
+    logging.getLogger("httpcore").setLevel(http_log_level)
+
+    include_http_server = getenv_bool("OTEL_INCLUDE_HTTP_SERVER", False)
     logging.getLogger("uvicorn.error").setLevel(log_level)
+    uvicorn_access_level = log_level if include_http_server else logging.CRITICAL
+    logging.getLogger("uvicorn.access").setLevel(uvicorn_access_level)
 
 
 logger = logging.getLogger(__name__)
```

---

### Commit: e2be3af
**Subject:** feat(telemetry): add comprehensive DEBUG logging for prompts and memory events
**Date:** 2026-01-29 19:34:35 +0100
**Author:** Alejandro Saucedo

- Log system prompt preview and length on process_message entry
- Log user messages and delegation tasks with content preview
- Log memory event creation for traceability
- Log model input (last user message) and full response preview
- Log tool arguments and result content
- Log delegation task and result preview
- Log OTEL_INCLUDE_HTTP_CLIENT/SERVER in startup config


```diff
 python/agent/client.py | 25 ++++++++++++++++++++-----
 python/agent/server.py |  6 ++++++
 2 files changed, 26 insertions(+), 5 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/client.py b/python/agent/client.py
index 4f17d2c..2025974 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -361,12 +361,16 @@ class Agent:
 
             # Build enhanced system prompt with tools/agents info
             system_prompt = await self._build_system_prompt(user_system_prompt)
+            logger.debug(f"System prompt built ({len(system_prompt)} chars)")
+            logger.debug(f"System prompt preview: {system_prompt[:300]}...")
             messages = [{"role": "system", "content": system_prompt}]
 
             # Handle both string and array input formats
             if isinstance(message, str):
+                logger.debug(f"User message: {message[:200]}...")
                 user_event = self.memory.create_event("user_message", message)
                 await self.memory.add_event(session_id, user_event)
+                logger.debug(f"Memory event created: user_message")
                 messages.append({"role": "user", "content": message})
             else:
                 for msg in message:
@@ -376,18 +380,23 @@ class Agent:
                         continue  # Already captured above
 
                     if role == "task-delegation":
+                        logger.debug(f"Received delegation task: {content[:200]}...")
                         delegation_event = self.memory.create_event(
                             "task_delegation_received", content
                         )
                         await self.memory.add_event(session_id, delegation_event)
+                        logger.debug(f"Memory event created: task_delegation_received")
                         messages.append({"role": "user", "content": content})
                     else:
                         messages.append({"role": role, "content": content})
                         if role == "user":
+                            logger.debug(f"User message: {content[:200]}...")
                             user_event = self.memory.create_event("user_message", content)
                             await self.memory.add_event(session_id, user_event)
+                            logger.debug(f"Memory event created: user_message")
 
             # Agentic loop - iterate up to max_steps
+            logger.debug(f"Starting agentic loop with {len(messages)} messages")
             async for chunk in self._agentic_loop(messages, session_id, stream):
                 yield chunk
 
@@ -525,8 +534,13 @@ class Agent:
         failed = False
         try:
             logger.debug(f"Model call: {model_name}, messages count: {len(messages)}")
+            # Log the last user message for debugging
+            for msg in reversed(messages):
+                if msg.get("role") in ("user", "task-delegation"):
+                    logger.debug(f"Model input (last user msg): {msg.get('content', '')[:200]}...")
+                    break
             content = cast(str, await self.model_api.process_message(messages, stream=False))
-            logger.debug(f"Model response length: {len(content)}")
+            logger.debug(f"Model response ({len(content)} chars): {content[:200]}...")
             return content
         except Exception as e:
             failed = True
@@ -546,7 +560,7 @@ class Agent:
             metric_kind="tool",
             metric_attrs={"tool": tool_name},
         )
-        logger.debug(f"Executing tool: {tool_name}, args: {list(tool_args.keys())}")
+        logger.debug(f"Executing tool: {tool_name}, args: {tool_args}")
         failed = False
         try:
             tool_result = None
@@ -557,7 +571,7 @@ class Agent:
 
             if tool_result is None:
                 raise ValueError(f"Tool '{tool_name}' not found")
-            logger.debug(f"Tool {tool_name} completed successfully")
+            logger.debug(f"Tool {tool_name} result: {str(tool_result)[:200]}...")
             return tool_result
         except Exception as e:
             failed = True
@@ -583,13 +597,14 @@ class Agent:
             metric_kind="delegation",
             metric_attrs={"target": agent_name},
         )
-        logger.debug(f"Delegating to sub-agent: {agent_name}, task length: {len(task)}")
+        logger.debug(f"Delegating to sub-agent: {agent_name}")
+        logger.debug(f"Delegation task: {task[:200]}...")
         failed = False
         try:
             result = await self.delegate_to_sub_agent(
                 agent_name, task, context_messages, session_id
             )
-            logger.debug(f"Delegation to {agent_name} completed, result length: {len(result)}")
+            logger.debug(f"Delegation to {agent_name} result: {result[:200]}...")
             return result
         except Exception as e:
             failed = True
diff --git a/python/agent/server.py b/python/agent/server.py
index e3c45ec..5d5bd39 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -277,6 +277,12 @@ class AgentServer:
             logger.info(
                 f"  OTEL_EXPORTER_OTLP_ENDPOINT: {os.getenv('OTEL_EXPORTER_OTLP_ENDPOINT', 'N/A')}"
             )
+            logger.info(
+                f"  OTEL_INCLUDE_HTTP_CLIENT: {getenv_bool('OTEL_INCLUDE_HTTP_CLIENT', False)}"
+            )
+            logger.info(
+                f"  OTEL_INCLUDE_HTTP_SERVER: {getenv_bool('OTEL_INCLUDE_HTTP_SERVER', False)}"
+            )
             logger.debug(
                 f"  OTEL_RESOURCE_ATTRIBUTES: {os.getenv('OTEL_RESOURCE_ATTRIBUTES', 'N/A')}"
             )
```

---

### Commit: 10b0f6f
**Subject:** docs(telemetry): move REPORT.md content to docs and add gitignore
**Date:** 2026-01-29 19:36:06 +0100
**Author:** Alejandro Saucedo

- Remove REPORT.md from git tracking
- Add logging configuration section to docs/operator/telemetry.md
- Add distributed tracing section explaining manual context propagation
- Add PLAN-OTEL-RESULTS.md to .gitignore for session notes


```diff
 .gitignore                 | 113 +++++++++++++++++++------------------------
 REPORT.md                  | 117 ---------------------------------------------
 docs/operator/telemetry.md |  74 ++++++++++++++++++++++++++++
 3 files changed, 123 insertions(+), 181 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/.gitignore b/.gitignore
index d08b9a8..18912c7 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,83 +1,68 @@
-# Go
-bin/
-dist/
-*.o
-*.a
-*.so
-.DS_Store
-vendor/
-go.sum
 
-# Python
 __pycache__/
-*.py[cod]
-*$py.class
-*.so
-.Python
-build/
-develop-eggs/
-dist/
-downloads/
-eggs/
-.eggs/
-lib/
-lib64/
-parts/
-sdist/
-var/
-wheels/
-.pytest_cache/
 .coverage
-htmlcov/
-*.egg-info/
+.DS_Store
+.eggs/
 .env
 .env.local
+.idea/
+.operator-sdk/
+.pytest_cache/
+.Python
 .venv
-venv/
-ENV/
-env/
-
-# IDE
 .vscode/
-.idea/
-*.swp
+*.a
+*.egg-info/
+*.log
+*.o
+*.py[cod]
+*.so
 *.swo
+*.swp
 *~
-.DS_Store
-
-# Kubernetes
-kubeconfig
-kubeconfig.yaml
-
+*$py.class
+# Analysis reports (local only, not committed)
+# Build artifacts
 # Docker
-docker-compose.override.yml
-
-# Test outputs
-*.log
-cover.out
-
+# Generated KIND E2E values (created by hack/update-kind-e2e-values.sh)
+# Go
+# IDE
+# Kubernetes
 # Operator SDK
+# Planning documents (local only, not committed)
+# Python
+# Test outputs
+awesome-ai-apps
 bin/
-dist/
-.operator-sdk/
-
-# Build artifacts
 build/
-dist/
 CLAUDE.md
-TASKS.md
-
-# Generated KIND E2E values (created by hack/update-kind-e2e-values.sh)
+cover.out
+develop-eggs/
+dist/
+docker-compose.override.yml
+downloads/
+eggs/
+env/
+ENV/
+go.sum
 hack/kind-e2e-values.yaml
-
-# Analysis reports (local only, not committed)
+htmlcov/
+kubeconfig
+kubeconfig.yaml
+lib/
+lib64/
 MEMORY_REPORT.md
-
-# Planning documents (local only, not committed)
-PLAN.md
-PLAN-*.md
-awesome-ai-apps
-REPORT.md
+parts/
 PLAN_*
+PLAN-*.md
+PLAN-OTEL-RESULTS.md
+PLAN.md
 REPORT_*.md
 REPORT-*.md
+REPORT.md
+sdist/
+TASKS.md
+var/
+vendor/
+venv/
+wheels/
diff --git a/REPORT.md b/REPORT.md
deleted file mode 100644
index 90b178d..0000000
--- a/REPORT.md
+++ /dev/null
@@ -1,117 +0,0 @@
-# OTEL Logging Configuration
-
-## Overview
-
-This document describes the OpenTelemetry logging configuration options for KAOS data plane components (Agent, MCPServer).
-
-## Distributed Tracing
-
-KAOS implements **manual trace context propagation** across agent delegations:
-
-- **Outgoing requests** (RemoteAgent): Automatically injects `traceparent`/`tracestate` headers
-- **Incoming requests** (AgentServer): Automatically extracts trace context from headers
-
-This ensures traces are connected across agent-to-agent delegations **without** requiring HTTPX/FastAPI auto-instrumentation, avoiding noisy HTTP spans.
-
-## Environment Variables
-
-### Log Level
-
-| Variable | Default | Description |
-|----------|---------|-------------|
-| `LOG_LEVEL` | `INFO` | Log level for all components (TRACE, DEBUG, INFO, WARNING, ERROR) |
-| `AGENT_LOG_LEVEL` | - | Fallback for LOG_LEVEL (backwards compatibility) |
-
-The log level controls:
-- Python logging output to stdout
-- OTEL log export level (DEBUG logs only exported when LOG_LEVEL=DEBUG)
-
-### HTTP Tracing Options
-
-| Variable | Default | Description |
-|----------|---------|-------------|
-| `OTEL_INCLUDE_HTTP_CLIENT` | `false` | Enable HTTPX client span tracing and logging |
-| `OTEL_INCLUDE_HTTP_SERVER` | `false` | Enable FastAPI span tracing and uvicorn access logs |
-
-#### OTEL_INCLUDE_HTTP_CLIENT
-
-When **disabled** (default):
-- HTTPX client calls do NOT create trace spans
-- Trace context is still propagated via headers (manual injection)
-- `httpx`, `httpcore`, `mcp.client.streamable_http` loggers set to WARNING
-- Reduces noise from MCP SSE connections which create many HTTP calls
-
-When **enabled** (`true`):
-- All HTTPX client calls create trace spans (noisy)
-- HTTP client loggers use configured LOG_LEVEL
-- Useful for debugging network issues
-
-#### OTEL_INCLUDE_HTTP_SERVER
-
-When **disabled** (default):
-- FastAPI incoming requests do NOT create HTTP spans
-- Trace context is still extracted from headers (manual extraction)
-- Uvicorn access logs are suppressed (no "GET /health 200" messages)
-- Reduces noise from Kubernetes health/readiness probes
-
-When **enabled** (`true`):
-- FastAPI incoming requests create trace spans (noisy)
-- Uvicorn access logs are emitted at configured LOG_LEVEL
-- Useful for debugging incoming request issues
-
-## Log Record Attributes
-
-OTEL log exports include these attributes:
-
-| Attribute | Description |
-|-----------|-------------|
-| `logger_name` | Python logger name (e.g., `agent.client`, `mcptools.server`) |
-| `code.filepath` | Source file path |
-| `code.function` | Function name |
-| `code.lineno` | Line number |
-
-## Log/Span Correlation
-
-When OTEL is enabled, all log entries within a traced request include:
-- `trace_id` - correlates with the request trace
-- `span_id` - correlates with the active span
-
-**Important:** Logs must be emitted BEFORE `span_failure()` or AFTER `span_begin()` to include trace correlation.
-
-## Configuration Examples
-
-### Minimal OTEL (default)
-
-```bash
-# Connected agent traces, no HTTP noise
-LOG_LEVEL=INFO
-OTEL_SDK_DISABLED=false
-OTEL_SERVICE_NAME=my-agent
-OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
-```
-
-### Debug with full HTTP tracing
-
-```bash
-# Full visibility for debugging (noisy)
-LOG_LEVEL=DEBUG
-OTEL_INCLUDE_HTTP_CLIENT=true
-OTEL_INCLUDE_HTTP_SERVER=true
-```
-
-### Production recommended
-
-```bash
-# Connected traces, errors visible, minimal noise
-LOG_LEVEL=INFO
-OTEL_INCLUDE_HTTP_CLIENT=false
-OTEL_INCLUDE_HTTP_SERVER=false
-```
-
-## Helper Functions
-
-The `telemetry.manager` module exports:
-
-- `get_log_level()` - Get LOG_LEVEL as string
-- `get_log_level_int()` - Get LOG_LEVEL as logging constant
-- `getenv_bool(name, default)` - Parse boolean env var ("true"/"1"/"yes")
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index 7131c1b..6eb3de0 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -422,6 +422,80 @@ The operator automatically sets these environment variables when telemetry is en
 
 For additional configuration, use standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html) via `spec.config.env`.
 
+## Logging Configuration
+
+### Log Level
+
+| Variable | Default | Description |
+|----------|---------|-------------|
+| `LOG_LEVEL` | `INFO` | Log level for all components (TRACE, DEBUG, INFO, WARNING, ERROR) |
+| `AGENT_LOG_LEVEL` | - | Fallback for LOG_LEVEL (backwards compatibility) |
+
+The log level controls:
+- Python logging output to stdout
+- OTEL log export level (DEBUG logs only exported when LOG_LEVEL=DEBUG)
+
+### HTTP Tracing Options
+
+| Variable | Default | Description |
+|----------|---------|-------------|
+| `OTEL_INCLUDE_HTTP_CLIENT` | `false` | Enable HTTPX client span tracing and logging |
+| `OTEL_INCLUDE_HTTP_SERVER` | `false` | Enable FastAPI span tracing and uvicorn access logs |
+
+**OTEL_INCLUDE_HTTP_CLIENT:**
+
+When **disabled** (default):
+- HTTPX client calls do NOT create trace spans
+- Trace context is still propagated via headers (manual injection)
+- `httpx`, `httpcore`, `mcp.client.streamable_http` loggers set to WARNING
+- Reduces noise from MCP SSE connections which create many HTTP calls
+
+When **enabled** (`true`):
+- All HTTPX client calls create trace spans (noisy)
+- HTTP client loggers use configured LOG_LEVEL
+- Useful for debugging network issues
+
+**OTEL_INCLUDE_HTTP_SERVER:**
+
+When **disabled** (default):
+- FastAPI incoming requests do NOT create HTTP spans
+- Trace context is still extracted from headers (manual extraction)
+- Uvicorn access logs are suppressed (no "GET /health 200" messages)
+- Reduces noise from Kubernetes health/readiness probes
+
+When **enabled** (`true`):
+- FastAPI incoming requests create trace spans (noisy)
+- Uvicorn access logs are emitted at configured LOG_LEVEL
+- Useful for debugging incoming request issues
+
+### Log Record Attributes
+
+OTEL log exports include these attributes:
+
+| Attribute | Description |
+|-----------|-------------|
+| `logger_name` | Python logger name (e.g., `agent.client`, `mcptools.server`) |
+| `code.filepath` | Source file path |
+| `code.function` | Function name |
+| `code.lineno` | Line number |
+
+### Log/Span Correlation
+
+When OTEL is enabled, all log entries within a traced request include:
+- `trace_id` - correlates with the request trace
+- `span_id` - correlates with the active span
+
+**Important:** Logs must be emitted BEFORE `span_failure()` or AFTER `span_begin()` to include trace correlation.
+
+## Distributed Tracing
+
+KAOS implements **manual trace context propagation** across agent delegations:
+
+- **Outgoing requests** (RemoteAgent): Automatically injects `traceparent`/`tracestate` headers
+- **Incoming requests** (AgentServer): Automatically extracts trace context from headers
+
+This ensures traces are connected across agent-to-agent delegations **without** requiring HTTPX/FastAPI auto-instrumentation, avoiding noisy HTTP spans.
+
 ## Health Check Exclusions
 
 By default, Kubernetes liveness and readiness probe endpoints are excluded from telemetry traces. This prevents excessive trace data from health checks that typically run every 10-30 seconds.
```

---

### Commit: 65da1bc
**Subject:** refactor(telemetry): make KaosOtelManager true singleton with __new__
**Date:** 2026-01-30 08:33:06 +0100
**Author:** Alejandro Saucedo

- Use __new__ pattern to ensure only one instance exists
- Log warning if service_name override attempted after init
- Remove get_instance() classmethod (no longer needed)
- Add _reset_for_testing() classmethod for test isolation
- Update tests with setup/teardown to reset singleton state


```diff
 operator/config/samples/3-hierarchical-agents.yaml | 14 +++---
 python/telemetry/manager.py                        | 56 +++++++++++++++-------
 python/tests/test_telemetry.py                     | 12 +++++
 3 files changed, 59 insertions(+), 23 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/operator/config/samples/3-hierarchical-agents.yaml b/operator/config/samples/3-hierarchical-agents.yaml
index d3ce74a..ae5061f 100644
--- a/operator/config/samples/3-hierarchical-agents.yaml
+++ b/operator/config/samples/3-hierarchical-agents.yaml
@@ -29,7 +29,7 @@ spec:
   mode: Proxy
   proxyConfig:
     models:
-    - deepseek-ai/DeepSeek-V3-0324
+    - "*"
     provider: nebius
     apiKey:
       valueFrom:
@@ -83,7 +83,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324
+  model: deepseek-ai/DeepSeek-V3-0324-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -105,7 +105,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324
+  model: deepseek-ai/DeepSeek-V3-0324-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -127,7 +127,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324
+  model: deepseek-ai/DeepSeek-V3-0324-fast
   mcpServers:
   - hierarchy-calc-mcp
   config:
@@ -148,7 +148,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324
+  model: deepseek-ai/DeepSeek-V3-0324-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -166,7 +166,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324
+  model: deepseek-ai/DeepSeek-V3-0324-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -184,7 +184,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324
+  model: deepseek-ai/DeepSeek-V3-0324-fast
   mcpServers:
   - hierarchy-calc-mcp
   config:
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 86e626c..fabd472 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -306,11 +306,15 @@ class KaosOtelManager:
     Uses inline span management via span_begin/span_success/span_failure instead
     of context managers. Timing is handled internally via contextvars.
 
-    Can be used as a singleton via the module-level `otel` instance, or
-    instantiated with a specific service name for tests.
+    This is a singleton class - instantiate it in each module as:
+        otel = KaosOtelManager()
+
+    The first instantiation sets the service name (from env var or parameter).
+    Subsequent instantiations return the same instance; if a different service_name
+    is passed, a warning is logged and ignored.
 
     Example:
-        from telemetry.manager import otel
+        otel = KaosOtelManager()
         otel.span_begin("process_request", attrs={"session.id": "abc123"})
         try:
             # do work
@@ -322,15 +326,34 @@ class KaosOtelManager:
             otel.span_success()
     """
 
-    # Singleton instance (lazy initialized)
     _instance: Optional["KaosOtelManager"] = None
+    _initialized: bool = False
+
+    def __new__(cls, service_name: Optional[str] = None) -> "KaosOtelManager":
+        """Ensure only one instance exists (singleton pattern)."""
+        if cls._instance is None:
+            cls._instance = super().__new__(cls)
+        elif service_name is not None:
+            # Warn if trying to re-init with different service name
+            existing_name = getattr(cls._instance, "service_name", None)
+            if existing_name and service_name != existing_name:
+                logger.warning(
+                    f"KaosOtelManager already initialized with service '{existing_name}', "
+                    f"ignoring new service_name '{service_name}'"
+                )
+        return cls._instance
 
     def __init__(self, service_name: Optional[str] = None):
         """Initialize manager with service context.
 
         Args:
-            service_name: Name of the service (reads from OTEL_SERVICE_NAME if not provided)
+            service_name: Name of the service (reads from OTEL_SERVICE_NAME if not provided).
+                          Only used on first initialization; ignored on subsequent calls.
         """
+        if self.__class__._initialized:
+            return
+        self.__class__._initialized = True
+
         self.service_name = service_name or _get_service_name()
         self._tracer = trace.get_tracer(f"kaos.{self.service_name}")
         self._meter = metrics.get_meter(f"kaos.{self.service_name}")
@@ -345,6 +368,16 @@ class KaosOtelManager:
         self._delegation_counter: Optional[metrics.Counter] = None
         self._delegation_duration: Optional[metrics.Histogram] = None
 
+    @classmethod
+    def _reset_for_testing(cls) -> None:
+        """Reset singleton state for testing purposes only.
+
+        This allows tests to create fresh instances with specific service names.
+        Should NOT be used in production code.
+        """
+        cls._instance = None
+        cls._initialized = False
+
     def _ensure_metrics(self) -> None:
         """Lazily initialize metric instruments."""
         if self._request_counter is not None:
@@ -586,16 +619,7 @@ class KaosOtelManager:
         ctx = extract(carrier)
         return otel_context.attach(ctx)
 
-    @classmethod
-    def get_instance(cls) -> "KaosOtelManager":
-        """Get or create the singleton instance.
-
-        Service name is read from OTEL_SERVICE_NAME or AGENT_NAME env var.
-        """
-        if cls._instance is None:
-            cls._instance = cls()
-        return cls._instance
-
 
 # Module-level singleton for easy import: `from telemetry.manager import otel`
-otel = KaosOtelManager.get_instance()
+# Can also be instantiated in any module as `otel = KaosOtelManager()` - returns same instance
+otel = KaosOtelManager()
diff --git a/python/tests/test_telemetry.py b/python/tests/test_telemetry.py
index 3f2882d..cfed936 100644
--- a/python/tests/test_telemetry.py
+++ b/python/tests/test_telemetry.py
@@ -124,6 +124,18 @@ class TestOtelConfig:
 class TestKaosOtelManager:
     """Tests for KaosOtelManager class."""
 
+    def setup_method(self):
+        """Reset singleton before each test."""
+        from telemetry.manager import KaosOtelManager
+
+        KaosOtelManager._reset_for_testing()
+
+    def teardown_method(self):
+        """Reset singleton after each test."""
+        from telemetry.manager import KaosOtelManager
+
+        KaosOtelManager._reset_for_testing()
+
     def test_manager_creation(self):
         """Test creating a KaosOtelManager."""
         from telemetry.manager import KaosOtelManager
```

---

### Commit: a99d040
**Subject:** fix(mcptools): suppress noisy docket.worker and fakeredis DEBUG logs
**Date:** 2026-01-30 19:38:23 +0100
**Author:** Alejandro Saucedo

- Add WARNING level for docket.worker and fakeredis loggers
- These are internal FastMCP dependencies that produce noise at DEBUG


```diff
 operator/config/samples/3-hierarchical-agents.yaml | 12 ++++++------
 python/mcptools/server.py                          |  4 ++++
 2 files changed, 10 insertions(+), 6 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/operator/config/samples/3-hierarchical-agents.yaml b/operator/config/samples/3-hierarchical-agents.yaml
index ae5061f..10fa83c 100644
--- a/operator/config/samples/3-hierarchical-agents.yaml
+++ b/operator/config/samples/3-hierarchical-agents.yaml
@@ -83,7 +83,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324-fast
+  model: meta-llama/Meta-Llama-3.1-8B-Instruct-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -105,7 +105,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324-fast
+  model: meta-llama/Meta-Llama-3.1-8B-Instruct-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -127,7 +127,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324-fast
+  model: meta-llama/Meta-Llama-3.1-8B-Instruct-fast
   mcpServers:
   - hierarchy-calc-mcp
   config:
@@ -148,7 +148,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324-fast
+  model: meta-llama/Meta-Llama-3.1-8B-Instruct-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -166,7 +166,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324-fast
+  model: meta-llama/Meta-Llama-3.1-8B-Instruct-fast
   mcpServers:
   - hierarchy-echo-mcp
   config:
@@ -184,7 +184,7 @@ metadata:
   namespace: kaos-hierarchy
 spec:
   modelAPI: hierarchy-modelapi
-  model: deepseek-ai/DeepSeek-V3-0324-fast
+  model: meta-llama/Meta-Llama-3.1-8B-Instruct-fast
   mcpServers:
   - hierarchy-calc-mcp
   config:
diff --git a/python/mcptools/server.py b/python/mcptools/server.py
index 61604ff..7e2f78d 100644
--- a/python/mcptools/server.py
+++ b/python/mcptools/server.py
@@ -68,6 +68,10 @@ def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> No
     uvicorn_access_level = log_level if include_http_server else logging.CRITICAL
     logging.getLogger("uvicorn.access").setLevel(uvicorn_access_level)
 
+    # Suppress noisy internal libraries (docket/fakeredis used by FastMCP)
+    logging.getLogger("docket.worker").setLevel(logging.WARNING)
+    logging.getLogger("fakeredis").setLevel(logging.WARNING)
+
 
 logger = logging.getLogger(__name__)
 
```

---

---

## Pre-Merge Commits (Initial OTEL Implementation)


### Commit: 0879a9a
**Subject:** feat(telemetry): add OpenTelemetry core module with tracing and metrics
**Date:** 2026-01-24 18:48:53 +0100
**Author:** Alejandro Saucedo

- Add opentelemetry dependencies to pyproject.toml
- Create agent/telemetry module with config, tracing, and metrics
- Support OTLP export, console export, and log correlation config
- Add semantic conventions for agent, model, tool, delegation spans
- Add counters and histograms for request/model/tool/delegation metrics


```diff
 python/agent/telemetry/__init__.py |  43 +++++++
 python/agent/telemetry/config.py   | 173 ++++++++++++++++++++++++++
 python/agent/telemetry/metrics.py  | 248 +++++++++++++++++++++++++++++++++++++
 python/agent/telemetry/tracing.py  | 168 +++++++++++++++++++++++++
 python/pyproject.toml              |   9 ++
 5 files changed, 641 insertions(+)
```

#### Detailed Changes

```diff
diff --git a/python/agent/telemetry/__init__.py b/python/agent/telemetry/__init__.py
new file mode 100644
index 0000000..cb8eb27
--- /dev/null
+++ b/python/agent/telemetry/__init__.py
@@ -0,0 +1,43 @@
+"""
+OpenTelemetry instrumentation for KAOS agents.
+
+Provides tracing, metrics, and log correlation for:
+- Agent processing (agentic loop, tool calls, delegations)
+- Model API calls (LLM inference)
+- MCP tool execution
+- A2A agent communication
+"""
+
+from agent.telemetry.config import TelemetryConfig, init_telemetry
+from agent.telemetry.tracing import (
+    get_tracer,
+    get_current_span,
+    inject_context,
+    extract_context,
+    span_attributes,
+)
+from agent.telemetry.metrics import (
+    get_meter,
+    record_request,
+    record_model_call,
+    record_tool_call,
+    record_delegation,
+)
+
+__all__ = [
+    # Configuration
+    "TelemetryConfig",
+    "init_telemetry",
+    # Tracing
+    "get_tracer",
+    "get_current_span",
+    "inject_context",
+    "extract_context",
+    "span_attributes",
+    # Metrics
+    "get_meter",
+    "record_request",
+    "record_model_call",
+    "record_tool_call",
+    "record_delegation",
+]
diff --git a/python/agent/telemetry/config.py b/python/agent/telemetry/config.py
new file mode 100644
index 0000000..c32c5f6
--- /dev/null
+++ b/python/agent/telemetry/config.py
@@ -0,0 +1,173 @@
+"""
+OpenTelemetry configuration and initialization for KAOS agents.
+
+Configures tracing, metrics, and log correlation based on environment variables.
+Supports OTLP export to any OpenTelemetry-compatible backend.
+"""
+
+import logging
+import os
+from dataclasses import dataclass, field
+from typing import Optional
+
+from opentelemetry import trace, metrics
+from opentelemetry.sdk.trace import TracerProvider
+from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
+from opentelemetry.sdk.metrics import MeterProvider
+from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader, ConsoleMetricExporter
+from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
+from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
+from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
+from opentelemetry.propagate import set_global_textmap
+from opentelemetry.propagators.composite import CompositePropagator
+from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
+from opentelemetry.baggage.propagation import W3CBaggagePropagator
+
+logger = logging.getLogger(__name__)
+
+
+@dataclass
+class TelemetryConfig:
+    """Configuration for OpenTelemetry instrumentation.
+
+    Environment variables:
+        OTEL_ENABLED: Enable/disable telemetry (default: false)
+        OTEL_SERVICE_NAME: Service name for traces (default: agent name)
+        OTEL_SERVICE_VERSION: Service version (default: 0.0.1)
+        OTEL_EXPORTER_OTLP_ENDPOINT: OTLP endpoint (default: http://localhost:4317)
+        OTEL_EXPORTER_OTLP_INSECURE: Use insecure connection (default: true)
+        OTEL_TRACES_ENABLED: Enable tracing (default: true when OTEL_ENABLED)
+        OTEL_METRICS_ENABLED: Enable metrics (default: true when OTEL_ENABLED)
+        OTEL_LOG_CORRELATION: Enable log correlation (default: true when OTEL_ENABLED)
+        OTEL_CONSOLE_EXPORT: Export to console for debugging (default: false)
+    """
+
+    enabled: bool = False
+    service_name: str = "kaos-agent"
+    service_version: str = "0.0.1"
+    otlp_endpoint: str = "http://localhost:4317"
+    otlp_insecure: bool = True
+    traces_enabled: bool = True
+    metrics_enabled: bool = True
+    log_correlation: bool = True
+    console_export: bool = False
+    extra_attributes: dict = field(default_factory=dict)
+
+    @classmethod
+    def from_env(cls, agent_name: Optional[str] = None) -> "TelemetryConfig":
+        """Create configuration from environment variables."""
+        enabled = os.getenv("OTEL_ENABLED", "false").lower() in ("true", "1", "yes")
+
+        return cls(
+            enabled=enabled,
+            service_name=os.getenv("OTEL_SERVICE_NAME", agent_name or "kaos-agent"),
+            service_version=os.getenv("OTEL_SERVICE_VERSION", "0.0.1"),
+            otlp_endpoint=os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4317"),
+            otlp_insecure=os.getenv("OTEL_EXPORTER_OTLP_INSECURE", "true").lower()
+            in ("true", "1", "yes"),
+            traces_enabled=os.getenv("OTEL_TRACES_ENABLED", "true").lower() in ("true", "1", "yes"),
+            metrics_enabled=os.getenv("OTEL_METRICS_ENABLED", "true").lower()
+            in ("true", "1", "yes"),
+            log_correlation=os.getenv("OTEL_LOG_CORRELATION", "true").lower()
+            in ("true", "1", "yes"),
+            console_export=os.getenv("OTEL_CONSOLE_EXPORT", "false").lower()
+            in ("true", "1", "yes"),
+        )
+
+
+# Global state for initialized providers
+_tracer_provider: Optional[TracerProvider] = None
+_meter_provider: Optional[MeterProvider] = None
+_initialized: bool = False
+
+
+def init_telemetry(config: TelemetryConfig) -> bool:
+    """Initialize OpenTelemetry with the given configuration.
+
+    Returns True if telemetry was initialized, False if disabled or already initialized.
+    """
+    global _tracer_provider, _meter_provider, _initialized
+
+    if _initialized:
+        logger.debug("Telemetry already initialized")
+        return False
+
+    if not config.enabled:
+        logger.info("OpenTelemetry disabled (OTEL_ENABLED=false)")
+        _initialized = True
+        return False
+
+    # Create resource with service information
+    resource = Resource.create(
+        {
+            SERVICE_NAME: config.service_name,
+            SERVICE_VERSION: config.service_version,
+            "deployment.environment": os.getenv("DEPLOYMENT_ENVIRONMENT", "development"),
+            **config.extra_attributes,
+        }
+    )
+
+    # Set up context propagation (W3C Trace Context + Baggage)
+    set_global_textmap(
+        CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
+    )
+
+    # Initialize tracing
+    if config.traces_enabled:
+        _tracer_provider = TracerProvider(resource=resource)
+
+        if config.console_export:
+            _tracer_provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
+
+        if config.otlp_endpoint:
+            otlp_exporter = OTLPSpanExporter(
+                endpoint=config.otlp_endpoint,
+                insecure=config.otlp_insecure,
+            )
+            _tracer_provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
+
+        trace.set_tracer_provider(_tracer_provider)
+        logger.info(f"OpenTelemetry tracing initialized: {config.otlp_endpoint}")
+
+    # Initialize metrics
+    if config.metrics_enabled:
+        readers = []
+
+        if config.console_export:
+            readers.append(PeriodicExportingMetricReader(ConsoleMetricExporter()))
+
+        if config.otlp_endpoint:
+            otlp_metric_exporter = OTLPMetricExporter(
+                endpoint=config.otlp_endpoint,
+                insecure=config.otlp_insecure,
+            )
+            readers.append(PeriodicExportingMetricReader(otlp_metric_exporter))
+
+        if readers:
+            _meter_provider = MeterProvider(resource=resource, metric_readers=readers)
+            metrics.set_meter_provider(_meter_provider)
+            logger.info(f"OpenTelemetry metrics initialized: {config.otlp_endpoint}")
+
+    _initialized = True
+    return True
+
+
+def shutdown_telemetry() -> None:
+    """Shutdown OpenTelemetry providers gracefully."""
+    global _tracer_provider, _meter_provider, _initialized
+
+    if _tracer_provider:
+        _tracer_provider.shutdown()
+        _tracer_provider = None
+
+    if _meter_provider:
+        _meter_provider.shutdown()
+        _meter_provider = None
+
+    _initialized = False
+    logger.info("OpenTelemetry shutdown complete")
+
+
+def is_telemetry_enabled() -> bool:
+    """Check if telemetry is enabled and initialized."""
+    return _initialized and (_tracer_provider is not None or _meter_provider is not None)
diff --git a/python/agent/telemetry/metrics.py b/python/agent/telemetry/metrics.py
new file mode 100644
index 0000000..1305ea6
--- /dev/null
+++ b/python/agent/telemetry/metrics.py
@@ -0,0 +1,248 @@
+"""
+Metrics utilities for KAOS agents.
+
+Provides counters, histograms, and gauges for:
+- Request processing
+- Model API calls
+- Tool execution
+- A2A delegation
+- Memory/session management
+"""
+
+import logging
+import time
+from contextlib import contextmanager
+from typing import Any, Dict, Iterator, Optional
+
+from opentelemetry import metrics
+from opentelemetry.metrics import Counter, Histogram, UpDownCounter
+
+logger = logging.getLogger(__name__)
+
+# Meter name for KAOS components
+METER_NAME = "kaos.agent"
+
+# Cached meter and instruments
+_meter: Optional[metrics.Meter] = None
+_request_counter: Optional[Counter] = None
+_request_duration: Optional[Histogram] = None
+_model_call_counter: Optional[Counter] = None
+_model_call_duration: Optional[Histogram] = None
+_tool_call_counter: Optional[Counter] = None
+_tool_call_duration: Optional[Histogram] = None
+_delegation_counter: Optional[Counter] = None
+_delegation_duration: Optional[Histogram] = None
+_active_sessions: Optional[UpDownCounter] = None
+_agentic_steps: Optional[Counter] = None
+
+
+def get_meter(name: str = METER_NAME) -> metrics.Meter:
+    """Get a meter for creating instruments."""
+    return metrics.get_meter(name)
+
+
+def _ensure_instruments() -> None:
+    """Lazily initialize metric instruments."""
+    global _meter, _request_counter, _request_duration
+    global _model_call_counter, _model_call_duration
+    global _tool_call_counter, _tool_call_duration
+    global _delegation_counter, _delegation_duration
+    global _active_sessions, _agentic_steps
+
+    if _meter is not None:
+        return
+
+    _meter = get_meter()
+
+    # Request metrics
+    _request_counter = _meter.create_counter(
+        name="kaos.agent.requests",
+        description="Number of agent requests processed",
+        unit="1",
+    )
+    _request_duration = _meter.create_histogram(
+        name="kaos.agent.request.duration",
+        description="Duration of agent request processing",
+        unit="ms",
+    )
+
+    # Model API metrics
+    _model_call_counter = _meter.create_counter(
+        name="kaos.agent.model.calls",
+        description="Number of model API calls",
+        unit="1",
+    )
+    _model_call_duration = _meter.create_histogram(
+        name="kaos.agent.model.duration",
+        description="Duration of model API calls",
+        unit="ms",
+    )
+
+    # Tool execution metrics
+    _tool_call_counter = _meter.create_counter(
+        name="kaos.agent.tool.calls",
+        description="Number of tool calls",
+        unit="1",
+    )
+    _tool_call_duration = _meter.create_histogram(
+        name="kaos.agent.tool.duration",
+        description="Duration of tool calls",
+        unit="ms",
+    )
+
+    # Delegation metrics
+    _delegation_counter = _meter.create_counter(
+        name="kaos.agent.delegations",
+        description="Number of A2A delegations",
+        unit="1",
+    )
+    _delegation_duration = _meter.create_histogram(
+        name="kaos.agent.delegation.duration",
+        description="Duration of A2A delegations",
+        unit="ms",
+    )
+
+    # Session metrics
+    _active_sessions = _meter.create_up_down_counter(
+        name="kaos.agent.sessions.active",
+        description="Number of active sessions",
+        unit="1",
+    )
+
+    # Agentic loop metrics
+    _agentic_steps = _meter.create_counter(
+        name="kaos.agent.agentic.steps",
+        description="Number of agentic loop steps",
+        unit="1",
+    )
+
+
+def record_request(
+    agent_name: str,
+    duration_ms: float,
+    success: bool = True,
+    stream: bool = False,
+    **attributes: Any,
+) -> None:
+    """Record a request metric."""
+    _ensure_instruments()
+    if _request_counter and _request_duration:
+        labels = {
+            "agent.name": agent_name,
+            "success": str(success).lower(),
+            "stream": str(stream).lower(),
+            **{k: str(v) for k, v in attributes.items() if v is not None},
+        }
+        _request_counter.add(1, labels)
+        _request_duration.record(duration_ms, labels)
+
+
+def record_model_call(
+    agent_name: str,
+    model_name: str,
+    duration_ms: float,
+    success: bool = True,
+    prompt_tokens: Optional[int] = None,
+    completion_tokens: Optional[int] = None,
+    **attributes: Any,
+) -> None:
+    """Record a model API call metric."""
+    _ensure_instruments()
+    if _model_call_counter and _model_call_duration:
+        labels = {
+            "agent.name": agent_name,
+            "model.name": model_name,
+            "success": str(success).lower(),
+            **{k: str(v) for k, v in attributes.items() if v is not None},
+        }
+        _model_call_counter.add(1, labels)
+        _model_call_duration.record(duration_ms, labels)
+
+
+def record_tool_call(
+    agent_name: str,
+    tool_name: str,
+    duration_ms: float,
+    success: bool = True,
+    mcp_server: Optional[str] = None,
+    **attributes: Any,
+) -> None:
+    """Record a tool call metric."""
+    _ensure_instruments()
+    if _tool_call_counter and _tool_call_duration:
+        labels = {
+            "agent.name": agent_name,
+            "tool.name": tool_name,
+            "success": str(success).lower(),
+            **({"mcp.server": mcp_server} if mcp_server else {}),
+            **{k: str(v) for k, v in attributes.items() if v is not None},
+        }
+        _tool_call_counter.add(1, labels)
+        _tool_call_duration.record(duration_ms, labels)
+
+
+def record_delegation(
+    agent_name: str,
+    target_agent: str,
+    duration_ms: float,
+    success: bool = True,
+    **attributes: Any,
+) -> None:
+    """Record an A2A delegation metric."""
+    _ensure_instruments()
+    if _delegation_counter and _delegation_duration:
+        labels = {
+            "agent.name": agent_name,
+            "target.agent": target_agent,
+            "success": str(success).lower(),
+            **{k: str(v) for k, v in attributes.items() if v is not None},
+        }
+        _delegation_counter.add(1, labels)
+        _delegation_duration.record(duration_ms, labels)
+
+
+def record_agentic_step(
+    agent_name: str,
+    step: int,
+    step_type: str,  # "model", "tool", "delegation", "final"
+    **attributes: Any,
+) -> None:
+    """Record an agentic loop step."""
+    _ensure_instruments()
+    if _agentic_steps:
+        labels = {
+            "agent.name": agent_name,
+            "step": str(step),
+            "step.type": step_type,
+            **{k: str(v) for k, v in attributes.items() if v is not None},
+        }
+        _agentic_steps.add(1, labels)
+
+
+def record_session_start(agent_name: str) -> None:
+    """Record a new session starting."""
+    _ensure_instruments()
+    if _active_sessions:
+        _active_sessions.add(1, {"agent.name": agent_name})
+
+
+def record_session_end(agent_name: str) -> None:
+    """Record a session ending."""
+    _ensure_instruments()
+    if _active_sessions:
+        _active_sessions.add(-1, {"agent.name": agent_name})
+
+
+@contextmanager
+def timed_operation(
+    operation_name: str,
+) -> Iterator[Dict[str, Any]]:
+    """Context manager for timing operations.
+
+    Yields a dict that will be populated with duration_ms after the block.
+    """
+    result: Dict[str, Any] = {"start_time": time.perf_counter()}
+    try:
+        yield result
+    finally:
+        result["duration_ms"] = (time.perf_counter() - result["start_time"]) * 1000
diff --git a/python/agent/telemetry/tracing.py b/python/agent/telemetry/tracing.py
new file mode 100644
index 0000000..d850fc6
--- /dev/null
+++ b/python/agent/telemetry/tracing.py
@@ -0,0 +1,168 @@
+"""
+Tracing utilities for KAOS agents.
+
+Provides span creation, context propagation, and semantic conventions
+for agent processing, model calls, tool execution, and A2A delegation.
+"""
+
+import logging
+from contextlib import contextmanager
+from typing import Any, Dict, Iterator, Optional
+
+from opentelemetry import trace
+from opentelemetry.propagate import inject, extract
+from opentelemetry.trace import Span, SpanKind, Status, StatusCode
+from opentelemetry.context import Context
+
+logger = logging.getLogger(__name__)
+
+# Semantic conventions for KAOS spans
+AGENT_NAME = "agent.name"
+AGENT_STEP = "agent.step"
+AGENT_MAX_STEPS = "agent.max_steps"
+SESSION_ID = "session.id"
+MODEL_NAME = "gen_ai.request.model"
+MODEL_PROVIDER = "gen_ai.system"
+TOOL_NAME = "tool.name"
+TOOL_ARGUMENTS = "tool.arguments"
+DELEGATION_TARGET = "agent.delegation.target"
+DELEGATION_TASK = "agent.delegation.task"
+MESSAGE_ROLE = "gen_ai.message.role"
+MESSAGE_CONTENT_LENGTH = "gen_ai.message.content_length"
+PROMPT_TOKENS = "gen_ai.usage.prompt_tokens"
+COMPLETION_TOKENS = "gen_ai.usage.completion_tokens"
+
+# Tracer name for KAOS components
+TRACER_NAME = "kaos.agent"
+
+
+def get_tracer(name: str = TRACER_NAME) -> trace.Tracer:
+    """Get a tracer for creating spans."""
+    return trace.get_tracer(name)
+
+
+def get_current_span() -> Optional[Span]:
+    """Get the current active span."""
+    span = trace.get_current_span()
+    return span if span.is_recording() else None
+
+
+def inject_context(carrier: Dict[str, str]) -> Dict[str, str]:
+    """Inject current trace context into a carrier (e.g., HTTP headers).
+
+    Used for propagating context to downstream services.
+    """
+    inject(carrier)
+    return carrier
+
+
+def extract_context(carrier: Dict[str, str]) -> Context:
+    """Extract trace context from a carrier (e.g., HTTP headers).
+
+    Used for receiving context from upstream services.
+    """
+    return extract(carrier)
+
+
+@contextmanager
+def span_attributes(span: Optional[Span], **attributes: Any) -> Iterator[None]:
+    """Context manager to add attributes to a span if it exists."""
+    if span:
+        for key, value in attributes.items():
+            if value is not None:
+                span.set_attribute(key, value)
+    yield
+
+
+def create_agent_span(
+    name: str,
+    agent_name: str,
+    session_id: Optional[str] = None,
+    kind: SpanKind = SpanKind.INTERNAL,
+    **attributes: Any,
+) -> Span:
+    """Create a span for agent operations."""
+    tracer = get_tracer()
+    span = tracer.start_span(
+        name,
+        kind=kind,
+        attributes={
+            AGENT_NAME: agent_name,
+            **({"session.id": session_id} if session_id else {}),
+            **{k: v for k, v in attributes.items() if v is not None},
+        },
+    )
+    return span
+
+
+def create_model_span(
+    model_name: str,
+    agent_name: str,
+    step: int,
+    **attributes: Any,
+) -> Span:
+    """Create a span for model API calls."""
+    tracer = get_tracer()
+    return tracer.start_span(
+        "model.inference",
+        kind=SpanKind.CLIENT,
+        attributes={
+            MODEL_NAME: model_name,
+            AGENT_NAME: agent_name,
+            AGENT_STEP: step,
+            **{k: v for k, v in attributes.items() if v is not None},
+        },
+    )
+
+
+def create_tool_span(
+    tool_name: str,
+    agent_name: str,
+    arguments: Optional[Dict[str, Any]] = None,
+    **attributes: Any,
+) -> Span:
+    """Create a span for tool execution."""
+    tracer = get_tracer()
+    return tracer.start_span(
+        f"tool.{tool_name}",
+        kind=SpanKind.CLIENT,
+        attributes={
+            TOOL_NAME: tool_name,
+            AGENT_NAME: agent_name,
+            **({"tool.arguments": str(arguments)} if arguments else {}),
+            **{k: v for k, v in attributes.items() if v is not None},
+        },
+    )
+
+
+def create_delegation_span(
+    target_agent: str,
+    task: str,
+    agent_name: str,
+    **attributes: Any,
+) -> Span:
+    """Create a span for A2A delegation."""
+    tracer = get_tracer()
+    return tracer.start_span(
+        f"delegate.{target_agent}",
+        kind=SpanKind.CLIENT,
+        attributes={
+            DELEGATION_TARGET: target_agent,
+            DELEGATION_TASK: task[:500],  # Truncate long tasks
+            AGENT_NAME: agent_name,
+            **{k: v for k, v in attributes.items() if v is not None},
+        },
+    )
+
+
+def end_span_ok(span: Span, message: Optional[str] = None) -> None:
+    """End a span with OK status."""
+    span.set_status(Status(StatusCode.OK, message))
+    span.end()
+
+
+def end_span_error(span: Span, error: Exception) -> None:
+    """End a span with ERROR status and record the exception."""
+    span.set_status(Status(StatusCode.ERROR, str(error)))
+    span.record_exception(error)
+    span.end()
diff --git a/python/pyproject.toml b/python/pyproject.toml
index 423c751..49030e4 100644
--- a/python/pyproject.toml
+++ b/python/pyproject.toml
@@ -12,6 +12,15 @@ dependencies = [
     "fastmcp>=1.0.0",
     "httpx>=0.25.0",
     "sse-starlette>=1.6.0",
+    # OpenTelemetry core
+    "opentelemetry-api>=1.20.0",
+    "opentelemetry-sdk>=1.20.0",
+    # OpenTelemetry instrumentation
+    "opentelemetry-instrumentation-fastapi>=0.41b0",
+    "opentelemetry-instrumentation-httpx>=0.41b0",
+    "opentelemetry-instrumentation-logging>=0.41b0",
+    # OpenTelemetry exporters
+    "opentelemetry-exporter-otlp>=1.20.0",
 ]
 
 [project.optional-dependencies]
```

---

### Commit: 2d698f1
**Subject:** feat(telemetry): add FastAPI instrumentation and OTel config to AgentServer
**Date:** 2026-01-24 18:50:36 +0100
**Author:** Alejandro Saucedo

- Add OTel environment variables to AgentServerSettings
- Initialize telemetry in create_agent_server
- Add FastAPI auto-instrumentation
- Shutdown telemetry on server shutdown


```diff
 python/agent/server.py | 50 ++++++++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 50 insertions(+)
```

#### Detailed Changes

```diff
diff --git a/python/agent/server.py b/python/agent/server.py
index 797ec01..c6755a7 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -3,6 +3,7 @@ AgentServer implementation for OpenAI-compatible API.
 
 FastAPI server with health probes, agent discovery, and chat completions endpoint.
 Supports both streaming and non-streaming responses.
+Includes OpenTelemetry instrumentation for tracing, metrics, and log correlation.
 """
 
 import time
@@ -22,6 +23,7 @@ from modelapi.client import ModelAPI
 from agent.client import Agent, RemoteAgent
 from agent.memory import LocalMemory
 from mcptools.client import MCPClient
+from agent.telemetry.config import TelemetryConfig, init_telemetry, shutdown_telemetry
 
 
 def configure_logging(level: str = "INFO") -> None:
@@ -103,6 +105,17 @@ class AgentServerSettings(BaseSettings):
     # Logging settings
     agent_access_log: bool = False  # Mute uvicorn access logs by default
 
+    # OpenTelemetry configuration
+    otel_enabled: bool = False  # Enable OpenTelemetry instrumentation
+    otel_service_name: Optional[str] = None  # Defaults to agent_name
+    otel_service_version: str = "0.0.1"
+    otel_exporter_otlp_endpoint: str = "http://localhost:4317"
+    otel_exporter_otlp_insecure: bool = True
+    otel_traces_enabled: bool = True
+    otel_metrics_enabled: bool = True
+    otel_log_correlation: bool = True
+    otel_console_export: bool = False  # Export to console for debugging
+
     class Config:
         env_file = ".env"
         case_sensitive = False
@@ -126,6 +139,7 @@ class AgentServer:
         agent: Agent,
         port: int = 8000,
         access_log: bool = False,
+        telemetry_config: Optional[TelemetryConfig] = None,
     ):
         """Initialize AgentServer with an agent.
 
@@ -133,10 +147,13 @@ class AgentServer:
             agent: Agent instance to serve
             port: Port to serve on
             access_log: Whether to enable uvicorn access logs (default: False)
+            telemetry_config: OpenTelemetry configuration (optional)
         """
         self.agent = agent
         self.port = port
         self.access_log = access_log
+        self.telemetry_config = telemetry_config
+        self._telemetry_enabled = False
 
         # Create FastAPI app
         self.app = FastAPI(
@@ -146,14 +163,29 @@ class AgentServer:
         )
 
         self._setup_routes()
+        self._setup_telemetry()
         logger.info(f"AgentServer initialized for {agent.name} on port {port}")
 
+    def _setup_telemetry(self):
+        """Setup OpenTelemetry instrumentation for FastAPI."""
+        if self.telemetry_config and self.telemetry_config.enabled:
+            try:
+                from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
+
+                FastAPIInstrumentor.instrument_app(self.app)
+                self._telemetry_enabled = True
+                logger.info("OpenTelemetry FastAPI instrumentation enabled")
+            except Exception as e:
+                logger.warning(f"Failed to enable OpenTelemetry instrumentation: {e}")
+
     @asynccontextmanager
     async def _lifespan(self, app: FastAPI):
         """Manage agent lifecycle."""
         self._log_startup_config()
         yield
         logger.info("AgentServer shutdown")
+        if self._telemetry_enabled:
+            shutdown_telemetry()
         await self.agent.close()
 
     def _log_startup_config(self):
@@ -505,6 +537,23 @@ def create_agent_server(
     else:
         memory = NullMemory()
 
+    # Initialize OpenTelemetry if enabled
+    telemetry_config = None
+    if settings.otel_enabled:
+        telemetry_config = TelemetryConfig(
+            enabled=settings.otel_enabled,
+            service_name=settings.otel_service_name or settings.agent_name,
+            service_version=settings.otel_service_version,
+            otlp_endpoint=settings.otel_exporter_otlp_endpoint,
+            otlp_insecure=settings.otel_exporter_otlp_insecure,
+            traces_enabled=settings.otel_traces_enabled,
+            metrics_enabled=settings.otel_metrics_enabled,
+            log_correlation=settings.otel_log_correlation,
+            console_export=settings.otel_console_export,
+            extra_attributes={"agent.name": settings.agent_name},
+        )
+        init_telemetry(telemetry_config)
+
     agent = Agent(
         name=settings.agent_name,
         description=settings.agent_description,
@@ -522,6 +571,7 @@ def create_agent_server(
         agent,
         port=settings.agent_port,
         access_log=settings.agent_access_log,
+        telemetry_config=telemetry_config,
     )
 
     return server
```

---

### Commit: e1e442c
**Subject:** feat(telemetry): add tracing to Agent agentic loop, model, tools, delegations
**Date:** 2026-01-24 18:53:29 +0100
**Author:** Alejandro Saucedo

- Add OpenTelemetry tracing spans for agent.process_message
- Add spans for each agentic loop step with step type
- Add model.inference spans for LLM calls with timing metrics
- Add tool.{name} spans for MCP tool executions
- Add delegate.{name} spans for A2A delegations
- Record request, model, tool, delegation metrics


```diff
 python/agent/client.py | 273 ++++++++++++++++++++++++++++++++++++++-----------
 1 file changed, 211 insertions(+), 62 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/client.py b/python/agent/client.py
index a698c51..989c5e8 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -3,6 +3,7 @@ Agent client implementation for OpenAI-compatible API.
 
 Clean, simple implementation with proper streaming support and tool integration.
 Includes agentic loop for tool calling and agent delegation.
+Instrumented with OpenTelemetry for tracing and metrics.
 
 Key design principles:
 - Agent decides when to delegate/call tools based on model response
@@ -13,14 +14,37 @@ Key design principles:
 
 import json
 import re
+import time
 import logging
 from typing import List, Dict, Any, Optional, AsyncIterator, Union, cast
 import httpx
 from dataclasses import dataclass
 
+from opentelemetry import trace
+from opentelemetry.trace import SpanKind, Status, StatusCode
+
 from modelapi.client import ModelAPI
 from agent.memory import LocalMemory, NullMemory
 from mcptools.client import MCPClient
+from agent.telemetry.tracing import (
+    get_tracer,
+    inject_context,
+    AGENT_NAME,
+    AGENT_STEP,
+    AGENT_MAX_STEPS,
+    SESSION_ID,
+    MODEL_NAME,
+    TOOL_NAME,
+    DELEGATION_TARGET,
+    DELEGATION_TASK,
+)
+from agent.telemetry.metrics import (
+    record_request,
+    record_model_call,
+    record_tool_call,
+    record_delegation,
+    record_agentic_step,
+)
 
 logger = logging.getLogger(__name__)
 
@@ -311,6 +335,9 @@ class Agent:
             For testing, set DEBUG_MOCK_RESPONSES env var to a JSON array of responses
             that will be used instead of calling the model API.
         """
+        tracer = get_tracer()
+        request_start = time.perf_counter()
+
         # Get or create session
         if session_id:
             session_id = await self.memory.get_or_create_session(session_id, "agent", "user")
@@ -319,51 +346,128 @@ class Agent:
 
         logger.debug(f"Processing message for session {session_id}, streaming={stream}")
 
-        # Extract user-provided system prompt (if any) from message array
-        user_system_prompt: Optional[str] = None
-        if isinstance(message, list):
-            for msg in message:
-                if msg.get("role") == "system":
-                    user_system_prompt = msg.get("content", "")
-                    break
-
-        # Build enhanced system prompt with tools/agents info
-        system_prompt = await self._build_system_prompt(user_system_prompt)
-        messages = [{"role": "system", "content": system_prompt}]
-
-        # Handle both string and array input formats
-        if isinstance(message, str):
-            user_event = self.memory.create_event("user_message", message)
-            await self.memory.add_event(session_id, user_event)
-            messages.append({"role": "user", "content": message})
-        else:
-            for msg in message:
-                role = msg.get("role", "user")
-                content = msg.get("content", "")
-                if role == "system":
-                    continue  # Already captured above
-
-                if role == "task-delegation":
-                    delegation_event = self.memory.create_event("task_delegation_received", content)
-                    await self.memory.add_event(session_id, delegation_event)
-                    messages.append({"role": "user", "content": content})
-                else:
-                    messages.append({"role": role, "content": content})
-                    if role == "user":
-                        user_event = self.memory.create_event("user_message", content)
-                        await self.memory.add_event(session_id, user_event)
-
-        try:
-            # Agentic loop - iterate up to max_steps
-            for step in range(self.max_steps):
-                logger.debug(f"Agentic loop step {step + 1}/{self.max_steps}")
+        # Start the main agent processing span
+        with tracer.start_as_current_span(
+            "agent.process_message",
+            kind=SpanKind.SERVER,
+            attributes={
+                AGENT_NAME: self.name,
+                SESSION_ID: session_id,
+                AGENT_MAX_STEPS: self.max_steps,
+                "stream": stream,
+            },
+        ) as span:
+            # Extract user-provided system prompt (if any) from message array
+            user_system_prompt: Optional[str] = None
+            if isinstance(message, list):
+                for msg in message:
+                    if msg.get("role") == "system":
+                        user_system_prompt = msg.get("content", "")
+                        break
+
+            # Build enhanced system prompt with tools/agents info
+            system_prompt = await self._build_system_prompt(user_system_prompt)
+            messages = [{"role": "system", "content": system_prompt}]
+
+            # Handle both string and array input formats
+            if isinstance(message, str):
+                user_event = self.memory.create_event("user_message", message)
+                await self.memory.add_event(session_id, user_event)
+                messages.append({"role": "user", "content": message})
+            else:
+                for msg in message:
+                    role = msg.get("role", "user")
+                    content = msg.get("content", "")
+                    if role == "system":
+                        continue  # Already captured above
+
+                    if role == "task-delegation":
+                        delegation_event = self.memory.create_event(
+                            "task_delegation_received", content
+                        )
+                        await self.memory.add_event(session_id, delegation_event)
+                        messages.append({"role": "user", "content": content})
+                    else:
+                        messages.append({"role": role, "content": content})
+                        if role == "user":
+                            user_event = self.memory.create_event("user_message", content)
+                            await self.memory.add_event(session_id, user_event)
 
-                # Get model response (stream=False always returns str)
-                content = cast(str, await self.model_api.process_message(messages, stream=False))
+            try:
+                # Agentic loop - iterate up to max_steps
+                async for chunk in self._agentic_loop(messages, session_id, stream, span, tracer):
+                    yield chunk
+
+                # Record successful request metric
+                duration_ms = (time.perf_counter() - request_start) * 1000
+                record_request(self.name, duration_ms, success=True, stream=stream)
+                span.set_status(Status(StatusCode.OK))
+
+            except Exception as e:
+                error_msg = f"Error processing message: {str(e)}"
+                logger.error(error_msg)
+                error_event = self.memory.create_event("error", error_msg)
+                await self.memory.add_event(session_id, error_event)
+
+                # Record failed request metric
+                duration_ms = (time.perf_counter() - request_start) * 1000
+                record_request(self.name, duration_ms, success=False, stream=stream)
+                span.set_status(Status(StatusCode.ERROR, str(e)))
+                span.record_exception(e)
+
+                yield f"Sorry, I encountered an error: {str(e)}"
+
+    async def _agentic_loop(
+        self,
+        messages: List[Dict[str, str]],
+        session_id: str,
+        stream: bool,
+        parent_span: trace.Span,
+        tracer: trace.Tracer,
+    ) -> AsyncIterator[str]:
+        """Execute the agentic loop with tracing."""
+        for step in range(self.max_steps):
+            logger.debug(f"Agentic loop step {step + 1}/{self.max_steps}")
+
+            # Create span for this step
+            with tracer.start_as_current_span(
+                f"agent.step.{step + 1}",
+                kind=SpanKind.INTERNAL,
+                attributes={
+                    AGENT_NAME: self.name,
+                    AGENT_STEP: step + 1,
+                    AGENT_MAX_STEPS: self.max_steps,
+                },
+            ) as step_span:
+                # Get model response with tracing
+                model_start = time.perf_counter()
+                with tracer.start_as_current_span(
+                    "model.inference",
+                    kind=SpanKind.CLIENT,
+                    attributes={
+                        AGENT_NAME: self.name,
+                        MODEL_NAME: self.model_api.model if self.model_api else "unknown",
+                        AGENT_STEP: step + 1,
+                    },
+                ) as model_span:
+                    content = cast(
+                        str, await self.model_api.process_message(messages, stream=False)
+                    )
+                    model_duration = (time.perf_counter() - model_start) * 1000
+                    record_model_call(
+                        self.name,
+                        self.model_api.model if self.model_api else "unknown",
+                        model_duration,
+                        success=True,
+                    )
+                    model_span.set_status(Status(StatusCode.OK))
 
                 # Check for tool call
                 tool_call = self._parse_block(content, "tool_call")
                 if tool_call:
+                    record_agentic_step(self.name, step + 1, "tool")
+                    step_span.set_attribute("step.type", "tool")
+
                     tool_event = self.memory.create_event("tool_call", tool_call)
                     await self.memory.add_event(session_id, tool_event)
 
@@ -373,14 +477,38 @@ class Agent:
                         if not tool_name:
                             raise ValueError("Tool name not specified")
 
-                        # Execute tool inline
-                        tool_result = None
-                        for mcp_client in self.mcp_clients:
-                            if tool_name in mcp_client._tools:
-                                tool_result = await mcp_client.call_tool(tool_name, tool_args)
-                                break
-                        if tool_result is None:
-                            raise ValueError(f"Tool '{tool_name}' not found")
+                        # Execute tool with tracing
+                        tool_start = time.perf_counter()
+                        with tracer.start_as_current_span(
+                            f"tool.{tool_name}",
+                            kind=SpanKind.CLIENT,
+                            attributes={
+                                AGENT_NAME: self.name,
+                                TOOL_NAME: tool_name,
+                                "tool.arguments": str(tool_args)[:500],
+                            },
+                        ) as tool_span:
+                            tool_result = None
+                            mcp_server_name = None
+                            for mcp_client in self.mcp_clients:
+                                if tool_name in mcp_client._tools:
+                                    mcp_server_name = getattr(mcp_client, "name", "unknown")
+                                    tool_result = await mcp_client.call_tool(tool_name, tool_args)
+                                    break
+
+                            if tool_result is None:
+                                raise ValueError(f"Tool '{tool_name}' not found")
+
+                            tool_duration = (time.perf_counter() - tool_start) * 1000
+                            record_tool_call(
+                                self.name,
+                                tool_name,
+                                tool_duration,
+                                success=True,
+                                mcp_server=mcp_server_name,
+                            )
+                            tool_span.set_status(Status(StatusCode.OK))
+                            tool_span.set_attribute("tool.result_length", len(str(tool_result)))
 
                         result_event = self.memory.create_event(
                             "tool_result", {"tool": tool_name, "result": tool_result}
@@ -401,6 +529,9 @@ class Agent:
                 # Check for delegation
                 delegation = self._parse_block(content, "delegate")
                 if delegation:
+                    record_agentic_step(self.name, step + 1, "delegation")
+                    step_span.set_attribute("step.type", "delegation")
+
                     agent_name = delegation.get("agent", "")
                     task = delegation.get("task", "")
 
@@ -416,9 +547,31 @@ class Agent:
 
                     try:
                         context_messages = [m for m in messages if m.get("role") != "system"]
-                        delegation_result = await self.delegate_to_sub_agent(
-                            agent_name, task, context_messages, session_id
-                        )
+
+                        # Delegate with tracing
+                        delegation_start = time.perf_counter()
+                        with tracer.start_as_current_span(
+                            f"delegate.{agent_name}",
+                            kind=SpanKind.CLIENT,
+                            attributes={
+                                AGENT_NAME: self.name,
+                                DELEGATION_TARGET: agent_name,
+                                DELEGATION_TASK: task[:500],
+                            },
+                        ) as delegation_span:
+                            # Inject trace context for propagation
+                            headers: Dict[str, str] = {}
+                            inject_context(headers)
+
+                            delegation_result = await self.delegate_to_sub_agent(
+                                agent_name, task, context_messages, session_id
+                            )
+
+                            delegation_duration = (time.perf_counter() - delegation_start) * 1000
+                            record_delegation(
+                                self.name, agent_name, delegation_duration, success=True
+                            )
+                            delegation_span.set_status(Status(StatusCode.OK))
 
                         messages.append({"role": "assistant", "content": content})
                         messages.append(
@@ -432,6 +585,9 @@ class Agent:
                         continue
 
                 # No tool call or delegation - this is the final response
+                record_agentic_step(self.name, step + 1, "final")
+                step_span.set_attribute("step.type", "final")
+
                 response_event = self.memory.create_event("agent_response", content)
                 await self.memory.add_event(session_id, response_event)
 
@@ -442,17 +598,10 @@ class Agent:
                     yield content
                 return
 
-            # Max steps reached
-            max_steps_msg = f"Reached maximum reasoning steps ({self.max_steps})"
-            logger.warning(max_steps_msg)
-            yield max_steps_msg
-
-        except Exception as e:
-            error_msg = f"Error processing message: {str(e)}"
-            logger.error(error_msg)
-            error_event = self.memory.create_event("error", error_msg)
-            await self.memory.add_event(session_id, error_event)
-            yield f"Sorry, I encountered an error: {str(e)}"
+        # Max steps reached
+        max_steps_msg = f"Reached maximum reasoning steps ({self.max_steps})"
+        logger.warning(max_steps_msg)
+        yield max_steps_msg
 
     async def delegate_to_sub_agent(
         self,
```

---

### Commit: d966d7c
**Subject:** feat(telemetry): add OpenTelemetry log correlation with trace_id/span_id
**Date:** 2026-01-24 18:54:40 +0100
**Author:** Alejandro Saucedo

- Update configure_logging to support OTel correlation format
- Add LoggingInstrumentor when otel_log_correlation is enabled
- Include trace_id and span_id in log messages for correlation


```diff
 python/agent/server.py | 32 +++++++++++++++++++++++++++++---
 1 file changed, 29 insertions(+), 3 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/server.py b/python/agent/server.py
index c6755a7..d03ebe4 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -26,23 +26,45 @@ from mcptools.client import MCPClient
 from agent.telemetry.config import TelemetryConfig, init_telemetry, shutdown_telemetry
 
 
-def configure_logging(level: str = "INFO") -> None:
+def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
     """Configure logging for the application.
 
     Sets up a consistent logging format and ensures all application loggers
     are properly configured to output to stdout.
+
+    Args:
+        level: Log level (DEBUG, INFO, WARNING, ERROR)
+        otel_correlation: If True, include trace_id and span_id in log format
     """
     log_level = getattr(logging, level.upper(), logging.INFO)
 
+    # Log format with optional OTel correlation
+    if otel_correlation:
+        log_format = (
+            "%(asctime)s - %(name)s - %(levelname)s - "
+            "[trace_id=%(otelTraceID)s span_id=%(otelSpanID)s] - %(message)s"
+        )
+    else:
+        log_format = "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
+
     # Configure root logger
     logging.basicConfig(
         level=log_level,
-        format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
+        format=log_format,
         datefmt="%Y-%m-%d %H:%M:%S",
         stream=sys.stdout,
         force=True,  # Override any existing configuration
     )
 
+    # If OTel correlation is enabled, add the LoggingInstrumentor
+    if otel_correlation:
+        try:
+            from opentelemetry.instrumentation.logging import LoggingInstrumentor
+
+            LoggingInstrumentor().instrument(set_logging_format=False)
+        except Exception as e:
+            logging.getLogger(__name__).warning(f"Failed to enable OTel log correlation: {e}")
+
     # Ensure our application loggers are at the right level
     for logger_name in [
         "agent",
@@ -468,7 +490,11 @@ def create_agent_server(
         settings = AgentServerSettings()  # type: ignore[call-arg]
 
     # Configure logging before anything else
-    configure_logging(settings.agent_log_level)
+    # Configure logging with optional OTel correlation
+    configure_logging(
+        settings.agent_log_level,
+        otel_correlation=settings.otel_enabled and settings.otel_log_correlation,
+    )
 
     model_api = ModelAPI(model=settings.model_name, api_base=settings.model_api_url)
 
```

---

### Commit: 79e78de
**Subject:** feat(telemetry): add OpenTelemetry support to MCP server
**Date:** 2026-01-24 18:55:50 +0100
**Author:** Alejandro Saucedo

- Add OTel environment variables to MCPServerSettings
- Initialize telemetry in MCPServer when enabled
- Add log correlation support for MCP server


```diff
 python/mcptools/server.py | 70 +++++++++++++++++++++++++++++++++++++++++++----
 1 file changed, 65 insertions(+), 5 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/mcptools/server.py b/python/mcptools/server.py
index 02b3034..20a64a5 100644
--- a/python/mcptools/server.py
+++ b/python/mcptools/server.py
@@ -2,7 +2,7 @@ import logging
 import sys
 import time
 from types import FunctionType
-from typing import Dict, Any, Callable, List, Literal
+from typing import Dict, Any, Callable, List, Literal, Optional
 from fastmcp import FastMCP
 import uvicorn
 from fastmcp.server.http import StarletteWithLifespan
@@ -11,23 +11,45 @@ from starlette.routing import Route
 from starlette.responses import JSONResponse
 
 
-def configure_logging(level: str = "INFO") -> None:
+def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
     """Configure logging for the application.
 
     Sets up a consistent logging format and ensures all application loggers
     are properly configured to output to stdout.
+
+    Args:
+        level: Log level (DEBUG, INFO, WARNING, ERROR)
+        otel_correlation: If True, include trace_id and span_id in log format
     """
     log_level = getattr(logging, level.upper(), logging.INFO)
 
+    # Log format with optional OTel correlation
+    if otel_correlation:
+        log_format = (
+            "%(asctime)s - %(name)s - %(levelname)s - "
+            "[trace_id=%(otelTraceID)s span_id=%(otelSpanID)s] - %(message)s"
+        )
+    else:
+        log_format = "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
+
     # Configure root logger
     logging.basicConfig(
         level=log_level,
-        format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
+        format=log_format,
         datefmt="%Y-%m-%d %H:%M:%S",
         stream=sys.stdout,
         force=True,  # Override any existing configuration
     )
 
+    # If OTel correlation is enabled, add the LoggingInstrumentor
+    if otel_correlation:
+        try:
+            from opentelemetry.instrumentation.logging import LoggingInstrumentor
+
+            LoggingInstrumentor().instrument(set_logging_format=False)
+        except Exception as e:
+            logging.getLogger(__name__).warning(f"Failed to enable OTel log correlation: {e}")
+
     # Ensure our application loggers are at the right level
     for logger_name in ["mcptools", "mcptools.server", "mcptools.client"]:
         logging.getLogger(logger_name).setLevel(log_level)
@@ -51,6 +73,16 @@ class MCPServerSettings(BaseSettings):
     mcp_log_level: str = "INFO"
     mcp_access_log: bool = False  # Mute uvicorn access logs by default
 
+    # OpenTelemetry configuration
+    otel_enabled: bool = False
+    otel_service_name: Optional[str] = None  # Defaults to "mcp-server"
+    otel_service_version: str = "0.0.1"
+    otel_exporter_otlp_endpoint: str = "http://localhost:4317"
+    otel_exporter_otlp_insecure: bool = True
+    otel_traces_enabled: bool = True
+    otel_metrics_enabled: bool = True
+    otel_log_correlation: bool = True
+
 
 class MCPServer:
     """MCP server that hosts tools via FastMCP Streamable HTTP protocol.
@@ -61,20 +93,48 @@ class MCPServer:
 
     def __init__(self, settings: MCPServerSettings):
         """Initialize MCP server."""
-        # Configure logging first
-        configure_logging(settings.mcp_log_level)
+        # Configure logging with optional OTel correlation
+        configure_logging(
+            settings.mcp_log_level,
+            otel_correlation=settings.otel_enabled and settings.otel_log_correlation,
+        )
 
         self._host = settings.mcp_host
         self._port = settings.mcp_port
         self._log_level = settings.mcp_log_level
         self._access_log = settings.mcp_access_log
+        self._otel_enabled = settings.otel_enabled
         self.mcp = FastMCP("Dynamic MCP Server")
         self.tools_registry: Dict[str, Callable] = {}
 
+        # Initialize OpenTelemetry if enabled
+        if settings.otel_enabled:
+            self._init_telemetry(settings)
+
         # Register provided tools
         if settings.mcp_tools_string:
             self.register_tools_from_string(settings.mcp_tools_string)
 
+    def _init_telemetry(self, settings: MCPServerSettings):
+        """Initialize OpenTelemetry for the MCP server."""
+        try:
+            from agent.telemetry.config import TelemetryConfig, init_telemetry
+
+            config = TelemetryConfig(
+                enabled=settings.otel_enabled,
+                service_name=settings.otel_service_name or "mcp-server",
+                service_version=settings.otel_service_version,
+                otlp_endpoint=settings.otel_exporter_otlp_endpoint,
+                otlp_insecure=settings.otel_exporter_otlp_insecure,
+                traces_enabled=settings.otel_traces_enabled,
+                metrics_enabled=settings.otel_metrics_enabled,
+                log_correlation=settings.otel_log_correlation,
+            )
+            init_telemetry(config)
+            logger.info("OpenTelemetry initialized for MCP server")
+        except Exception as e:
+            logger.warning(f"Failed to initialize OpenTelemetry: {e}")
+
     def _log_startup_config(self):
         """Log server configuration on startup for debugging."""
         logger.info("=" * 60)
```

---

### Commit: 91fc10a
**Subject:** test(telemetry): add unit tests for OpenTelemetry instrumentation
**Date:** 2026-01-24 18:57:13 +0100
**Author:** Alejandro Saucedo

- Test TelemetryConfig env var loading
- Test tracing utilities (get_tracer, context injection/extraction)
- Test metrics recording functions
- Test AgentServerSettings OTel env vars
- Test MCPServerSettings OTel env vars


```diff
 python/tests/test_telemetry.py | 185 +++++++++++++++++++++++++++++++++++++++++
 1 file changed, 185 insertions(+)
```

#### Detailed Changes

```diff
diff --git a/python/tests/test_telemetry.py b/python/tests/test_telemetry.py
new file mode 100644
index 0000000..7510fb5
--- /dev/null
+++ b/python/tests/test_telemetry.py
@@ -0,0 +1,185 @@
+"""
+Tests for OpenTelemetry instrumentation.
+"""
+
+import pytest
+import os
+from unittest.mock import patch, MagicMock
+
+
+class TestTelemetryConfig:
+    """Tests for TelemetryConfig."""
+
+    def test_config_from_env_disabled_by_default(self):
+        """Test that telemetry is disabled by default."""
+        from agent.telemetry.config import TelemetryConfig
+
+        config = TelemetryConfig.from_env()
+        assert config.enabled is False
+        assert config.service_name == "kaos-agent"
+
+    def test_config_from_env_with_custom_values(self):
+        """Test configuration from environment variables."""
+        from agent.telemetry.config import TelemetryConfig
+
+        with patch.dict(
+            os.environ,
+            {
+                "OTEL_ENABLED": "true",
+                "OTEL_SERVICE_NAME": "test-agent",
+                "OTEL_SERVICE_VERSION": "1.0.0",
+                "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
+            },
+        ):
+            config = TelemetryConfig.from_env()
+            assert config.enabled is True
+            assert config.service_name == "test-agent"
+            assert config.service_version == "1.0.0"
+            assert config.otlp_endpoint == "http://collector:4317"
+
+    def test_config_from_env_with_agent_name(self):
+        """Test that agent_name is used as default service name."""
+        from agent.telemetry.config import TelemetryConfig
+
+        config = TelemetryConfig.from_env(agent_name="my-agent")
+        assert config.service_name == "my-agent"
+
+
+class TestTracingUtilities:
+    """Tests for tracing utilities."""
+
+    def test_get_tracer(self):
+        """Test getting a tracer."""
+        from agent.telemetry.tracing import get_tracer
+
+        tracer = get_tracer()
+        assert tracer is not None
+
+    def test_inject_extract_context(self):
+        """Test context injection and extraction."""
+        from agent.telemetry.tracing import inject_context, extract_context
+
+        carrier: dict = {}
+        inject_context(carrier)
+        # Context may or may not have traceparent depending on active span
+        context = extract_context(carrier)
+        assert context is not None
+
+
+class TestMetrics:
+    """Tests for metrics recording."""
+
+    def test_record_request_no_error(self):
+        """Test that record_request doesn't raise errors."""
+        from agent.telemetry.metrics import record_request
+
+        # Should not raise
+        record_request("test-agent", 100.0, success=True, stream=False)
+
+    def test_record_model_call_no_error(self):
+        """Test that record_model_call doesn't raise errors."""
+        from agent.telemetry.metrics import record_model_call
+
+        # Should not raise
+        record_model_call("test-agent", "gpt-4", 500.0, success=True)
+
+    def test_record_tool_call_no_error(self):
+        """Test that record_tool_call doesn't raise errors."""
+        from agent.telemetry.metrics import record_tool_call
+
+        # Should not raise
+        record_tool_call("test-agent", "calculator", 50.0, success=True)
+
+    def test_record_delegation_no_error(self):
+        """Test that record_delegation doesn't raise errors."""
+        from agent.telemetry.metrics import record_delegation
+
+        # Should not raise
+        record_delegation("test-agent", "worker-1", 200.0, success=True)
+
+    def test_record_agentic_step_no_error(self):
+        """Test that record_agentic_step doesn't raise errors."""
+        from agent.telemetry.metrics import record_agentic_step
+
+        # Should not raise
+        record_agentic_step("test-agent", 1, "model")
+
+    def test_timed_operation(self):
+        """Test timed_operation context manager."""
+        import time
+        from agent.telemetry.metrics import timed_operation
+
+        with timed_operation("test") as result:
+            time.sleep(0.01)
+
+        assert "duration_ms" in result
+        assert result["duration_ms"] >= 10  # At least 10ms
+
+
+class TestAgentServerTelemetrySettings:
+    """Tests for AgentServer telemetry settings."""
+
+    def test_default_otel_settings(self):
+        """Test that OTel is disabled by default in AgentServerSettings."""
+        with patch.dict(
+            os.environ,
+            {
+                "AGENT_NAME": "test",
+                "MODEL_API_URL": "http://localhost:8000",
+                "MODEL_NAME": "test-model",
+            },
+            clear=True,
+        ):
+            from agent.server import AgentServerSettings
+
+            settings = AgentServerSettings()  # type: ignore[call-arg]
+            assert settings.otel_enabled is False
+            assert settings.otel_traces_enabled is True
+            assert settings.otel_metrics_enabled is True
+
+    def test_otel_settings_from_env(self):
+        """Test OTel settings from environment variables."""
+        with patch.dict(
+            os.environ,
+            {
+                "AGENT_NAME": "test",
+                "MODEL_API_URL": "http://localhost:8000",
+                "MODEL_NAME": "test-model",
+                "OTEL_ENABLED": "true",
+                "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
+            },
+            clear=True,
+        ):
+            from agent.server import AgentServerSettings
+
+            settings = AgentServerSettings()  # type: ignore[call-arg]
+            assert settings.otel_enabled is True
+            assert settings.otel_exporter_otlp_endpoint == "http://collector:4317"
+
+
+class TestMCPServerTelemetrySettings:
+    """Tests for MCPServer telemetry settings."""
+
+    def test_default_otel_settings(self):
+        """Test that OTel is disabled by default in MCPServerSettings."""
+        from mcptools.server import MCPServerSettings
+
+        settings = MCPServerSettings()
+        assert settings.otel_enabled is False
+        assert settings.otel_traces_enabled is True
+
+    def test_otel_settings_from_env(self):
+        """Test OTel settings from environment variables."""
+        with patch.dict(
+            os.environ,
+            {
+                "OTEL_ENABLED": "true",
+                "OTEL_SERVICE_NAME": "my-mcp-server",
+            },
+            clear=True,
+        ):
+            from mcptools.server import MCPServerSettings
+
+            settings = MCPServerSettings()
+            assert settings.otel_enabled is True
+            assert settings.otel_service_name == "my-mcp-server"
```

---

### Commit: 89aa37c
**Subject:** docs(telemetry): add OpenTelemetry documentation page and update instructions
**Date:** 2026-01-24 19:01:50 +0100
**Author:** Alejandro Saucedo

- Create docs/operator/telemetry.md with full OTel configuration guide
- Add telemetry page to VitePress sidebar navigation
- Update python.instructions.md with telemetry module reference
- Include examples for SigNoz, Uptrace, and OTel Collector setup


```diff
 .github/instructions/python.instructions.md |   3 +
 docs/.vitepress/config.ts                   |   3 +-
 docs/operator/telemetry.md                  | 329 ++++++++++++++++++++++++++++
 3 files changed, 334 insertions(+), 1 deletion(-)
```

#### Detailed Changes

```diff
diff --git a/.github/instructions/python.instructions.md b/.github/instructions/python.instructions.md
index ba5733c..c377f8f 100644
--- a/.github/instructions/python.instructions.md
+++ b/.github/instructions/python.instructions.md
@@ -17,6 +17,7 @@ make format                     # Auto-format code
 - `agent/client.py`: Agent, RemoteAgent, AgentCard classes
 - `agent/server.py`: AgentServer with A2A endpoints
 - `agent/memory.py`: LocalMemory for session/event management
+- `agent/telemetry/`: OpenTelemetry instrumentation (tracing, metrics)
 - `mcptools/`: MCP (Model Context Protocol) tools
 - `modelapi/`: Model API client for OpenAI-compatible servers
 
@@ -28,6 +29,8 @@ make format                     # Auto-format code
 | `MODEL_NAME` | Model name (required) |
 | `AGENT_SUB_AGENTS` | Direct format: `"name:url,name:url"` |
 | `DEBUG_MOCK_RESPONSES` | JSON array of mock responses for testing |
+| `OTEL_ENABLED` | Enable OpenTelemetry instrumentation |
+| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP exporter endpoint |
 
 ## Testing Patterns
 - Use `DEBUG_MOCK_RESPONSES` for deterministic tests
diff --git a/docs/.vitepress/config.ts b/docs/.vitepress/config.ts
index 65ab6f6..31bcd8f 100644
--- a/docs/.vitepress/config.ts
+++ b/docs/.vitepress/config.ts
@@ -96,7 +96,8 @@ export default withMermaid(defineConfig({
           { text: 'Agent CRD', link: '/operator/agent-crd' },
           { text: 'ModelAPI CRD', link: '/operator/modelapi-crd' },
           { text: 'MCPServer CRD', link: '/operator/mcpserver-crd' },
-          { text: 'Gateway API', link: '/operator/gateway-api' }
+          { text: 'Gateway API', link: '/operator/gateway-api' },
+          { text: 'OpenTelemetry', link: '/operator/telemetry' }
         ]
       },
       {
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
new file mode 100644
index 0000000..b2454f4
--- /dev/null
+++ b/docs/operator/telemetry.md
@@ -0,0 +1,329 @@
+# OpenTelemetry
+
+KAOS supports OpenTelemetry for observability, including distributed tracing, metrics, and log correlation across all agent operations.
+
+## Overview
+
+When enabled, OpenTelemetry instrumentation provides:
+
+- **Tracing**: Distributed traces across agent requests, model calls, tool executions, and delegations
+- **Metrics**: Counters and histograms for requests, latency, and error rates
+- **Log Correlation**: Automatic injection of trace_id and span_id into log entries
+
+## Enabling Telemetry
+
+Add a `telemetry` section to your Agent's config:
+
+```yaml
+apiVersion: kaos.tools/v1alpha1
+kind: Agent
+metadata:
+  name: my-agent
+spec:
+  modelAPI: my-modelapi
+  model: "openai/gpt-4o"
+  config:
+    description: "Agent with OpenTelemetry enabled"
+    telemetry:
+      enabled: true
+      endpoint: "http://otel-collector.monitoring.svc.cluster.local:4317"
+      insecure: true
+```
+
+## Configuration Fields
+
+| Field | Type | Default | Description |
+|-------|------|---------|-------------|
+| `enabled` | bool | `false` | Enable OpenTelemetry instrumentation |
+| `endpoint` | string | `http://localhost:4317` | OTLP exporter endpoint |
+| `insecure` | bool | `true` | Use insecure connection (no TLS) |
+| `serviceName` | string | Agent name | Service name for traces/metrics |
+| `tracesEnabled` | bool | `true` | Enable trace collection |
+| `metricsEnabled` | bool | `true` | Enable metrics collection |
+| `logCorrelation` | bool | `true` | Inject trace context into logs |
+
+## Trace Spans
+
+The following spans are automatically created:
+
+### agent.process_message
+
+Root span for each request to the agent. Attributes:
+- `agent.name`: Name of the agent
+- `session.id`: Session identifier
+
+### agent.step.{n}
+
+Span for each iteration of the agentic reasoning loop. Attributes:
+- `agent.step`: Step number (1-based)
+- `agent.name`: Agent name
+
+### model.inference
+
+Span for LLM API calls. Attributes:
+- `model.name`: Model identifier
+- `model.api_url`: API endpoint
+
+### tool.{name}
+
+Span for MCP tool executions. Attributes:
+- `tool.name`: Tool name
+- `mcp.server`: MCP server name
+
+### delegate.{agent}
+
+Span for agent-to-agent delegations. Attributes:
+- `delegation.target`: Target agent name
+- `delegation.task`: Task description
+
+## Span Hierarchy Example
+
+```
+agent.process_message (SERVER)
+├── agent.step.1 (INTERNAL)
+│   └── model.inference (CLIENT)
+├── agent.step.2 (INTERNAL)
+│   ├── model.inference (CLIENT)
+│   └── tool.calculator (CLIENT)
+├── agent.step.3 (INTERNAL)
+│   ├── model.inference (CLIENT)
+│   └── delegate.researcher (CLIENT)
+└── agent.step.4 (INTERNAL)
+    └── model.inference (CLIENT)
+```
+
+## Metrics
+
+The following metrics are collected:
+
+| Metric | Type | Description |
+|--------|------|-------------|
+| `kaos.agent.requests` | Counter | Total requests processed |
+| `kaos.agent.request.duration` | Histogram | Request duration in seconds |
+| `kaos.model.calls` | Counter | Total model API calls |
+| `kaos.model.duration` | Histogram | Model call duration in seconds |
+| `kaos.tool.calls` | Counter | Total tool executions |
+| `kaos.tool.duration` | Histogram | Tool execution duration in seconds |
+| `kaos.delegations` | Counter | Total agent delegations |
+| `kaos.delegation.duration` | Histogram | Delegation duration in seconds |
+
+All metrics include labels:
+- `agent_name`: Name of the agent
+- `status`: "success" or "error"
+
+Tool metrics also include:
+- `tool_name`: Name of the tool
+- `mcp_server`: Name of the MCP server
+
+Delegation metrics also include:
+- `target_agent`: Name of the target agent
+
+## Log Correlation
+
+When `logCorrelation` is enabled, log entries include trace context:
+
+```
+2024-01-15 10:30:45 INFO [trace_id=abc123 span_id=def456] Processing message...
+```
+
+This allows correlating logs with traces in your observability backend.
+
+## Example: Agent with Full Telemetry
+
+```yaml
+apiVersion: kaos.tools/v1alpha1
+kind: Agent
+metadata:
+  name: traced-agent
+spec:
+  modelAPI: my-modelapi
+  model: "openai/gpt-4o"
+  mcpServers:
+  - calculator
+  config:
+    description: "Agent with full OpenTelemetry observability"
+    instructions: "You are a helpful assistant with calculator access."
+    telemetry:
+      enabled: true
+      endpoint: "http://otel-collector.monitoring.svc.cluster.local:4317"
+      insecure: true
+      serviceName: "traced-agent"
+      tracesEnabled: true
+      metricsEnabled: true
+      logCorrelation: true
+  agentNetwork:
+    access:
+    - researcher
+```
+
+## Setting Up an OTel Collector
+
+To collect telemetry, deploy an OpenTelemetry Collector in your cluster:
+
+```yaml
+apiVersion: v1
+kind: ConfigMap
+metadata:
+  name: otel-collector-config
+  namespace: monitoring
+data:
+  config.yaml: |
+    receivers:
+      otlp:
+        protocols:
+          grpc:
+            endpoint: 0.0.0.0:4317
+          http:
+            endpoint: 0.0.0.0:4318
+
+    processors:
+      batch:
+        timeout: 10s
+
+    exporters:
+      otlp:
+        endpoint: "your-backend:4317"
+        tls:
+          insecure: true
+
+    service:
+      pipelines:
+        traces:
+          receivers: [otlp]
+          processors: [batch]
+          exporters: [otlp]
+        metrics:
+          receivers: [otlp]
+          processors: [batch]
+          exporters: [otlp]
+---
+apiVersion: apps/v1
+kind: Deployment
+metadata:
+  name: otel-collector
+  namespace: monitoring
+spec:
+  replicas: 1
+  selector:
+    matchLabels:
+      app: otel-collector
+  template:
+    metadata:
+      labels:
+        app: otel-collector
+    spec:
+      containers:
+      - name: collector
+        image: otel/opentelemetry-collector:latest
+        args: ["--config=/etc/otel/config.yaml"]
+        ports:
+        - containerPort: 4317
+          name: otlp-grpc
+        - containerPort: 4318
+          name: otlp-http
+        volumeMounts:
+        - name: config
+          mountPath: /etc/otel
+      volumes:
+      - name: config
+        configMap:
+          name: otel-collector-config
+---
+apiVersion: v1
+kind: Service
+metadata:
+  name: otel-collector
+  namespace: monitoring
+spec:
+  selector:
+    app: otel-collector
+  ports:
+  - port: 4317
+    name: otlp-grpc
+  - port: 4318
+    name: otlp-http
+```
+
+## Using with SigNoz
+
+[SigNoz](https://signoz.io/) is an open-source APM that works well with KAOS:
+
+1. Deploy SigNoz in your cluster:
+```bash
+helm repo add signoz https://charts.signoz.io
+helm install signoz signoz/signoz -n monitoring --create-namespace
+```
+
+2. Configure agents to send telemetry to SigNoz:
+```yaml
+config:
+  telemetry:
+    enabled: true
+    endpoint: "http://signoz-otel-collector.monitoring.svc.cluster.local:4317"
+```
+
+3. Access the SigNoz UI to view traces, metrics, and logs.
+
+## Using with Uptrace
+
+[Uptrace](https://uptrace.dev/) is another excellent option:
+
+1. Deploy Uptrace:
+```bash
+helm repo add uptrace https://charts.uptrace.dev
+helm install uptrace uptrace/uptrace -n monitoring --create-namespace
+```
+
+2. Configure agents:
+```yaml
+config:
+  telemetry:
+    enabled: true
+    endpoint: "http://uptrace.monitoring.svc.cluster.local:14317"
+```
+
+## Environment Variables
+
+For advanced configuration, the following environment variables are passed to agent pods when telemetry is enabled:
+
+| Variable | Description |
+|----------|-------------|
+| `OTEL_ENABLED` | "true" when telemetry is enabled |
+| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL |
+| `OTEL_EXPORTER_OTLP_INSECURE` | "true" for insecure connections |
+| `OTEL_SERVICE_NAME` | Service name for traces/metrics |
+| `OTEL_TRACES_ENABLED` | "true" when traces are enabled |
+| `OTEL_METRICS_ENABLED` | "true" when metrics are enabled |
+| `OTEL_LOG_CORRELATION` | "true" when log correlation is enabled |
+
+## Troubleshooting
+
+### No traces appearing
+
+1. Verify telemetry is enabled:
+```bash
+kubectl get agent my-agent -o jsonpath='{.spec.config.telemetry}'
+```
+
+2. Check agent logs for OTel initialization:
+```bash
+kubectl logs -l agent=my-agent | grep -i otel
+```
+
+3. Verify collector is reachable:
+```bash
+kubectl exec -it deploy/agent-my-agent -- curl -v http://otel-collector.monitoring:4317
+```
+
+### High latency
+
+If telemetry adds noticeable latency:
+- Use batching in the OTel collector
+- Consider sampling for high-throughput agents
+- Disable metrics if only traces are needed
+
+### Missing spans
+
+Ensure all sub-agents and MCP servers are instrumented:
+- Each agent should have its own telemetry config
+- MCP servers share the agent's trace context via W3C Trace Context headers
```

---

### Commit: f46e82c
**Subject:** refactor(telemetry): simplify Python OTel with KaosOtelManager
**Date:** 2026-01-25 14:19:15 +0100
**Author:** Alejandro Saucedo

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


```diff
 python/agent/client.py             | 301 +++++++++++++------------------------
 python/agent/server.py             |  63 +++-----
 python/agent/telemetry/__init__.py |  49 ++----
 python/agent/telemetry/config.py   | 173 ---------------------
 python/agent/telemetry/manager.py  | 292 +++++++++++++++++++++++++++++++++++
 python/agent/telemetry/metrics.py  | 248 ------------------------------
 python/agent/telemetry/tracing.py  | 168 ---------------------
 python/mcptools/server.py          |  56 +++----
 python/tests/test_telemetry.py     | 282 +++++++++++++++++++---------------
 9 files changed, 613 insertions(+), 1019 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/client.py b/python/agent/client.py
index 989c5e8..b79b0c5 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -14,37 +14,15 @@ Key design principles:
 
 import json
 import re
-import time
 import logging
 from typing import List, Dict, Any, Optional, AsyncIterator, Union, cast
 import httpx
 from dataclasses import dataclass
 
-from opentelemetry import trace
-from opentelemetry.trace import SpanKind, Status, StatusCode
-
 from modelapi.client import ModelAPI
 from agent.memory import LocalMemory, NullMemory
 from mcptools.client import MCPClient
-from agent.telemetry.tracing import (
-    get_tracer,
-    inject_context,
-    AGENT_NAME,
-    AGENT_STEP,
-    AGENT_MAX_STEPS,
-    SESSION_ID,
-    MODEL_NAME,
-    TOOL_NAME,
-    DELEGATION_TARGET,
-    DELEGATION_TASK,
-)
-from agent.telemetry.metrics import (
-    record_request,
-    record_model_call,
-    record_tool_call,
-    record_delegation,
-    record_agentic_step,
-)
+from agent.telemetry import KaosOtelManager, timed
 
 logger = logging.getLogger(__name__)
 
@@ -208,6 +186,9 @@ class Agent:
         self.memory_context_limit = memory_context_limit
         self.memory_enabled = memory_enabled
 
+        # Telemetry manager (lightweight, always created - no-ops if OTel disabled)
+        self._otel = KaosOtelManager(name)
+
         logger.info(f"Agent initialized: {name}")
 
     async def _get_tools_prompt(self) -> Optional[str]:
@@ -335,139 +316,104 @@ class Agent:
             For testing, set DEBUG_MOCK_RESPONSES env var to a JSON array of responses
             that will be used instead of calling the model API.
         """
-        tracer = get_tracer()
-        request_start = time.perf_counter()
-
-        # Get or create session
-        if session_id:
-            session_id = await self.memory.get_or_create_session(session_id, "agent", "user")
-        else:
-            session_id = await self.memory.create_session("agent", "user")
-
-        logger.debug(f"Processing message for session {session_id}, streaming={stream}")
-
-        # Start the main agent processing span
-        with tracer.start_as_current_span(
-            "agent.process_message",
-            kind=SpanKind.SERVER,
-            attributes={
-                AGENT_NAME: self.name,
-                SESSION_ID: session_id,
-                AGENT_MAX_STEPS: self.max_steps,
-                "stream": stream,
-            },
-        ) as span:
-            # Extract user-provided system prompt (if any) from message array
-            user_system_prompt: Optional[str] = None
-            if isinstance(message, list):
-                for msg in message:
-                    if msg.get("role") == "system":
-                        user_system_prompt = msg.get("content", "")
-                        break
-
-            # Build enhanced system prompt with tools/agents info
-            system_prompt = await self._build_system_prompt(user_system_prompt)
-            messages = [{"role": "system", "content": system_prompt}]
-
-            # Handle both string and array input formats
-            if isinstance(message, str):
-                user_event = self.memory.create_event("user_message", message)
-                await self.memory.add_event(session_id, user_event)
-                messages.append({"role": "user", "content": message})
+        # Use the KaosOtelManager for simpler telemetry
+        with timed() as timing:
+            # Get or create session
+            if session_id:
+                session_id = await self.memory.get_or_create_session(session_id, "agent", "user")
             else:
-                for msg in message:
-                    role = msg.get("role", "user")
-                    content = msg.get("content", "")
-                    if role == "system":
-                        continue  # Already captured above
-
-                    if role == "task-delegation":
-                        delegation_event = self.memory.create_event(
-                            "task_delegation_received", content
-                        )
-                        await self.memory.add_event(session_id, delegation_event)
-                        messages.append({"role": "user", "content": content})
-                    else:
-                        messages.append({"role": role, "content": content})
-                        if role == "user":
-                            user_event = self.memory.create_event("user_message", content)
-                            await self.memory.add_event(session_id, user_event)
-
-            try:
-                # Agentic loop - iterate up to max_steps
-                async for chunk in self._agentic_loop(messages, session_id, stream, span, tracer):
-                    yield chunk
-
-                # Record successful request metric
-                duration_ms = (time.perf_counter() - request_start) * 1000
-                record_request(self.name, duration_ms, success=True, stream=stream)
-                span.set_status(Status(StatusCode.OK))
-
-            except Exception as e:
-                error_msg = f"Error processing message: {str(e)}"
-                logger.error(error_msg)
-                error_event = self.memory.create_event("error", error_msg)
-                await self.memory.add_event(session_id, error_event)
-
-                # Record failed request metric
-                duration_ms = (time.perf_counter() - request_start) * 1000
-                record_request(self.name, duration_ms, success=False, stream=stream)
-                span.set_status(Status(StatusCode.ERROR, str(e)))
-                span.record_exception(e)
-
-                yield f"Sorry, I encountered an error: {str(e)}"
+                session_id = await self.memory.create_session("agent", "user")
+
+            logger.debug(f"Processing message for session {session_id}, streaming={stream}")
+
+            # Use INTERNAL span (FastAPI already creates the SERVER span via auto-instrumentation)
+            span_attrs: Dict[str, Any] = {"agent.max_steps": self.max_steps, "stream": stream}
+            with self._otel.span(
+                "agent.agentic_loop",
+                session_id=session_id,
+                **span_attrs,
+            ) as span:
+                # Extract user-provided system prompt (if any) from message array
+                user_system_prompt: Optional[str] = None
+                if isinstance(message, list):
+                    for msg in message:
+                        if msg.get("role") == "system":
+                            user_system_prompt = msg.get("content", "")
+                            break
+
+                # Build enhanced system prompt with tools/agents info
+                system_prompt = await self._build_system_prompt(user_system_prompt)
+                messages = [{"role": "system", "content": system_prompt}]
+
+                # Handle both string and array input formats
+                if isinstance(message, str):
+                    user_event = self.memory.create_event("user_message", message)
+                    await self.memory.add_event(session_id, user_event)
+                    messages.append({"role": "user", "content": message})
+                else:
+                    for msg in message:
+                        role = msg.get("role", "user")
+                        content = msg.get("content", "")
+                        if role == "system":
+                            continue  # Already captured above
+
+                        if role == "task-delegation":
+                            delegation_event = self.memory.create_event(
+                                "task_delegation_received", content
+                            )
+                            await self.memory.add_event(session_id, delegation_event)
+                            messages.append({"role": "user", "content": content})
+                        else:
+                            messages.append({"role": role, "content": content})
+                            if role == "user":
+                                user_event = self.memory.create_event("user_message", content)
+                                await self.memory.add_event(session_id, user_event)
+
+                try:
+                    # Agentic loop - iterate up to max_steps
+                    async for chunk in self._agentic_loop(messages, session_id, stream):
+                        yield chunk
+
+                except Exception as e:
+                    error_msg = f"Error processing message: {str(e)}"
+                    logger.error(error_msg)
+                    error_event = self.memory.create_event("error", error_msg)
+                    await self.memory.add_event(session_id, error_event)
+                    yield f"Sorry, I encountered an error: {str(e)}"
+                    raise
+
+        # Record request metrics after completion
+        self._otel.record_request(timing["duration_ms"], success=True)
 
     async def _agentic_loop(
         self,
         messages: List[Dict[str, str]],
         session_id: str,
         stream: bool,
-        parent_span: trace.Span,
-        tracer: trace.Tracer,
     ) -> AsyncIterator[str]:
         """Execute the agentic loop with tracing."""
         for step in range(self.max_steps):
             logger.debug(f"Agentic loop step {step + 1}/{self.max_steps}")
 
             # Create span for this step
-            with tracer.start_as_current_span(
-                f"agent.step.{step + 1}",
-                kind=SpanKind.INTERNAL,
-                attributes={
-                    AGENT_NAME: self.name,
-                    AGENT_STEP: step + 1,
-                    AGENT_MAX_STEPS: self.max_steps,
-                },
-            ) as step_span:
+            step_attrs: Dict[str, Any] = {"step": step + 1, "max_steps": self.max_steps}
+            with self._otel.span(f"agent.step.{step + 1}", **step_attrs):
                 # Get model response with tracing
-                model_start = time.perf_counter()
-                with tracer.start_as_current_span(
-                    "model.inference",
-                    kind=SpanKind.CLIENT,
-                    attributes={
-                        AGENT_NAME: self.name,
-                        MODEL_NAME: self.model_api.model if self.model_api else "unknown",
-                        AGENT_STEP: step + 1,
-                    },
-                ) as model_span:
-                    content = cast(
-                        str, await self.model_api.process_message(messages, stream=False)
-                    )
-                    model_duration = (time.perf_counter() - model_start) * 1000
-                    record_model_call(
-                        self.name,
-                        self.model_api.model if self.model_api else "unknown",
-                        model_duration,
-                        success=True,
-                    )
-                    model_span.set_status(Status(StatusCode.OK))
+                with timed() as model_timing:
+                    with self._otel.model_span(
+                        self.model_api.model if self.model_api else "unknown"
+                    ):
+                        content = cast(
+                            str, await self.model_api.process_message(messages, stream=False)
+                        )
+                self._otel.record_model_call(
+                    self.model_api.model if self.model_api else "unknown",
+                    model_timing["duration_ms"],
+                )
 
                 # Check for tool call
                 tool_call = self._parse_block(content, "tool_call")
                 if tool_call:
-                    record_agentic_step(self.name, step + 1, "tool")
-                    step_span.set_attribute("step.type", "tool")
-
                     tool_event = self.memory.create_event("tool_call", tool_call)
                     await self.memory.add_event(session_id, tool_event)
 
@@ -478,37 +424,20 @@ class Agent:
                             raise ValueError("Tool name not specified")
 
                         # Execute tool with tracing
-                        tool_start = time.perf_counter()
-                        with tracer.start_as_current_span(
-                            f"tool.{tool_name}",
-                            kind=SpanKind.CLIENT,
-                            attributes={
-                                AGENT_NAME: self.name,
-                                TOOL_NAME: tool_name,
-                                "tool.arguments": str(tool_args)[:500],
-                            },
-                        ) as tool_span:
-                            tool_result = None
-                            mcp_server_name = None
-                            for mcp_client in self.mcp_clients:
-                                if tool_name in mcp_client._tools:
-                                    mcp_server_name = getattr(mcp_client, "name", "unknown")
-                                    tool_result = await mcp_client.call_tool(tool_name, tool_args)
-                                    break
-
-                            if tool_result is None:
-                                raise ValueError(f"Tool '{tool_name}' not found")
-
-                            tool_duration = (time.perf_counter() - tool_start) * 1000
-                            record_tool_call(
-                                self.name,
-                                tool_name,
-                                tool_duration,
-                                success=True,
-                                mcp_server=mcp_server_name,
-                            )
-                            tool_span.set_status(Status(StatusCode.OK))
-                            tool_span.set_attribute("tool.result_length", len(str(tool_result)))
+                        with timed() as tool_timing:
+                            with self._otel.tool_span(tool_name):
+                                tool_result = None
+                                for mcp_client in self.mcp_clients:
+                                    if tool_name in mcp_client._tools:
+                                        tool_result = await mcp_client.call_tool(
+                                            tool_name, tool_args
+                                        )
+                                        break
+
+                                if tool_result is None:
+                                    raise ValueError(f"Tool '{tool_name}' not found")
+
+                        self._otel.record_tool_call(tool_name, tool_timing["duration_ms"])
 
                         result_event = self.memory.create_event(
                             "tool_result", {"tool": tool_name, "result": tool_result}
@@ -529,9 +458,6 @@ class Agent:
                 # Check for delegation
                 delegation = self._parse_block(content, "delegate")
                 if delegation:
-                    record_agentic_step(self.name, step + 1, "delegation")
-                    step_span.set_attribute("step.type", "delegation")
-
                     agent_name = delegation.get("agent", "")
                     task = delegation.get("task", "")
 
@@ -549,29 +475,13 @@ class Agent:
                         context_messages = [m for m in messages if m.get("role") != "system"]
 
                         # Delegate with tracing
-                        delegation_start = time.perf_counter()
-                        with tracer.start_as_current_span(
-                            f"delegate.{agent_name}",
-                            kind=SpanKind.CLIENT,
-                            attributes={
-                                AGENT_NAME: self.name,
-                                DELEGATION_TARGET: agent_name,
-                                DELEGATION_TASK: task[:500],
-                            },
-                        ) as delegation_span:
-                            # Inject trace context for propagation
-                            headers: Dict[str, str] = {}
-                            inject_context(headers)
-
-                            delegation_result = await self.delegate_to_sub_agent(
-                                agent_name, task, context_messages, session_id
-                            )
+                        with timed() as delegation_timing:
+                            with self._otel.delegation_span(agent_name):
+                                delegation_result = await self.delegate_to_sub_agent(
+                                    agent_name, task, context_messages, session_id
+                                )
 
-                            delegation_duration = (time.perf_counter() - delegation_start) * 1000
-                            record_delegation(
-                                self.name, agent_name, delegation_duration, success=True
-                            )
-                            delegation_span.set_status(Status(StatusCode.OK))
+                        self._otel.record_delegation(agent_name, delegation_timing["duration_ms"])
 
                         messages.append({"role": "assistant", "content": content})
                         messages.append(
@@ -585,9 +495,6 @@ class Agent:
                         continue
 
                 # No tool call or delegation - this is the final response
-                record_agentic_step(self.name, step + 1, "final")
-                step_span.set_attribute("step.type", "final")
-
                 response_event = self.memory.create_event("agent_response", content)
                 await self.memory.add_event(session_id, response_event)
 
diff --git a/python/agent/server.py b/python/agent/server.py
index d03ebe4..602f442 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -23,7 +23,7 @@ from modelapi.client import ModelAPI
 from agent.client import Agent, RemoteAgent
 from agent.memory import LocalMemory
 from mcptools.client import MCPClient
-from agent.telemetry.config import TelemetryConfig, init_telemetry, shutdown_telemetry
+from agent.telemetry import init_otel, is_otel_enabled
 
 
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
@@ -127,17 +127,6 @@ class AgentServerSettings(BaseSettings):
     # Logging settings
     agent_access_log: bool = False  # Mute uvicorn access logs by default
 
-    # OpenTelemetry configuration
-    otel_enabled: bool = False  # Enable OpenTelemetry instrumentation
-    otel_service_name: Optional[str] = None  # Defaults to agent_name
-    otel_service_version: str = "0.0.1"
-    otel_exporter_otlp_endpoint: str = "http://localhost:4317"
-    otel_exporter_otlp_insecure: bool = True
-    otel_traces_enabled: bool = True
-    otel_metrics_enabled: bool = True
-    otel_log_correlation: bool = True
-    otel_console_export: bool = False  # Export to console for debugging
-
     class Config:
         env_file = ".env"
         case_sensitive = False
@@ -161,7 +150,6 @@ class AgentServer:
         agent: Agent,
         port: int = 8000,
         access_log: bool = False,
-        telemetry_config: Optional[TelemetryConfig] = None,
     ):
         """Initialize AgentServer with an agent.
 
@@ -169,13 +157,10 @@ class AgentServer:
             agent: Agent instance to serve
             port: Port to serve on
             access_log: Whether to enable uvicorn access logs (default: False)
-            telemetry_config: OpenTelemetry configuration (optional)
         """
         self.agent = agent
         self.port = port
         self.access_log = access_log
-        self.telemetry_config = telemetry_config
-        self._telemetry_enabled = False
 
         # Create FastAPI app
         self.app = FastAPI(
@@ -190,13 +175,14 @@ class AgentServer:
 
     def _setup_telemetry(self):
         """Setup OpenTelemetry instrumentation for FastAPI."""
-        if self.telemetry_config and self.telemetry_config.enabled:
+        if is_otel_enabled():
             try:
                 from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
+                from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
 
                 FastAPIInstrumentor.instrument_app(self.app)
-                self._telemetry_enabled = True
-                logger.info("OpenTelemetry FastAPI instrumentation enabled")
+                HTTPXClientInstrumentor().instrument()
+                logger.info("OpenTelemetry instrumentation enabled (FastAPI + HTTPX)")
             except Exception as e:
                 logger.warning(f"Failed to enable OpenTelemetry instrumentation: {e}")
 
@@ -206,8 +192,6 @@ class AgentServer:
         self._log_startup_config()
         yield
         logger.info("AgentServer shutdown")
-        if self._telemetry_enabled:
-            shutdown_telemetry()
         await self.agent.close()
 
     def _log_startup_config(self):
@@ -489,12 +473,11 @@ def create_agent_server(
         # Load from environment variables - requires AGENT_NAME and MODEL_API_URL
         settings = AgentServerSettings()  # type: ignore[call-arg]
 
-    # Configure logging before anything else
+    # Check if OTel enabled via env var
+    otel_enabled = is_otel_enabled()
+
     # Configure logging with optional OTel correlation
-    configure_logging(
-        settings.agent_log_level,
-        otel_correlation=settings.otel_enabled and settings.otel_log_correlation,
-    )
+    configure_logging(settings.agent_log_level, otel_correlation=otel_enabled)
 
     model_api = ModelAPI(model=settings.model_name, api_base=settings.model_api_url)
 
@@ -563,22 +546,17 @@ def create_agent_server(
     else:
         memory = NullMemory()
 
-    # Initialize OpenTelemetry if enabled
-    telemetry_config = None
-    if settings.otel_enabled:
-        telemetry_config = TelemetryConfig(
-            enabled=settings.otel_enabled,
-            service_name=settings.otel_service_name or settings.agent_name,
-            service_version=settings.otel_service_version,
-            otlp_endpoint=settings.otel_exporter_otlp_endpoint,
-            otlp_insecure=settings.otel_exporter_otlp_insecure,
-            traces_enabled=settings.otel_traces_enabled,
-            metrics_enabled=settings.otel_metrics_enabled,
-            log_correlation=settings.otel_log_correlation,
-            console_export=settings.otel_console_export,
-            extra_attributes={"agent.name": settings.agent_name},
-        )
-        init_telemetry(telemetry_config)
+    # Initialize OpenTelemetry if enabled (uses standard OTEL_* env vars)
+    init_otel(settings.agent_name)
+
+    # Configure log correlation if OTel is enabled
+    if is_otel_enabled():
+        try:
+            from opentelemetry.instrumentation.logging import LoggingInstrumentor
+
+            LoggingInstrumentor().instrument(set_logging_format=False)
+        except Exception as e:
+            logger.warning(f"Failed to enable OTel log correlation: {e}")
 
     agent = Agent(
         name=settings.agent_name,
@@ -597,7 +575,6 @@ def create_agent_server(
         agent,
         port=settings.agent_port,
         access_log=settings.agent_access_log,
-        telemetry_config=telemetry_config,
     )
 
     return server
diff --git a/python/agent/telemetry/__init__.py b/python/agent/telemetry/__init__.py
index cb8eb27..83b38d5 100644
--- a/python/agent/telemetry/__init__.py
+++ b/python/agent/telemetry/__init__.py
@@ -1,43 +1,22 @@
 """
-OpenTelemetry instrumentation for KAOS agents.
+OpenTelemetry instrumentation for KAOS.
 
-Provides tracing, metrics, and log correlation for:
-- Agent processing (agentic loop, tool calls, delegations)
-- Model API calls (LLM inference)
-- MCP tool execution
-- A2A agent communication
+Simple interface using standard OTEL_* environment variables.
+When OTEL_ENABLED=true, traces, metrics, and log correlation are enabled.
 """
 
-from agent.telemetry.config import TelemetryConfig, init_telemetry
-from agent.telemetry.tracing import (
-    get_tracer,
-    get_current_span,
-    inject_context,
-    extract_context,
-    span_attributes,
-)
-from agent.telemetry.metrics import (
-    get_meter,
-    record_request,
-    record_model_call,
-    record_tool_call,
-    record_delegation,
+from agent.telemetry.manager import (
+    OtelConfig,
+    KaosOtelManager,
+    init_otel,
+    is_otel_enabled,
+    timed,
 )
 
 __all__ = [
-    # Configuration
-    "TelemetryConfig",
-    "init_telemetry",
-    # Tracing
-    "get_tracer",
-    "get_current_span",
-    "inject_context",
-    "extract_context",
-    "span_attributes",
-    # Metrics
-    "get_meter",
-    "record_request",
-    "record_model_call",
-    "record_tool_call",
-    "record_delegation",
+    "OtelConfig",
+    "KaosOtelManager",
+    "init_otel",
+    "is_otel_enabled",
+    "timed",
 ]
diff --git a/python/agent/telemetry/config.py b/python/agent/telemetry/config.py
deleted file mode 100644
index c32c5f6..0000000
--- a/python/agent/telemetry/config.py
+++ /dev/null
@@ -1,173 +0,0 @@
-"""
-OpenTelemetry configuration and initialization for KAOS agents.
-
-Configures tracing, metrics, and log correlation based on environment variables.
-Supports OTLP export to any OpenTelemetry-compatible backend.
-"""
-
-import logging
-import os
-from dataclasses import dataclass, field
-from typing import Optional
-
-from opentelemetry import trace, metrics
-from opentelemetry.sdk.trace import TracerProvider
-from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
-from opentelemetry.sdk.metrics import MeterProvider
-from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader, ConsoleMetricExporter
-from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
-from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
-from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
-from opentelemetry.propagate import set_global_textmap
-from opentelemetry.propagators.composite import CompositePropagator
-from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
-from opentelemetry.baggage.propagation import W3CBaggagePropagator
-
-logger = logging.getLogger(__name__)
-
-
-@dataclass
-class TelemetryConfig:
-    """Configuration for OpenTelemetry instrumentation.
-
-    Environment variables:
-        OTEL_ENABLED: Enable/disable telemetry (default: false)
-        OTEL_SERVICE_NAME: Service name for traces (default: agent name)
-        OTEL_SERVICE_VERSION: Service version (default: 0.0.1)
-        OTEL_EXPORTER_OTLP_ENDPOINT: OTLP endpoint (default: http://localhost:4317)
-        OTEL_EXPORTER_OTLP_INSECURE: Use insecure connection (default: true)
-        OTEL_TRACES_ENABLED: Enable tracing (default: true when OTEL_ENABLED)
-        OTEL_METRICS_ENABLED: Enable metrics (default: true when OTEL_ENABLED)
-        OTEL_LOG_CORRELATION: Enable log correlation (default: true when OTEL_ENABLED)
-        OTEL_CONSOLE_EXPORT: Export to console for debugging (default: false)
-    """
-
-    enabled: bool = False
-    service_name: str = "kaos-agent"
-    service_version: str = "0.0.1"
-    otlp_endpoint: str = "http://localhost:4317"
-    otlp_insecure: bool = True
-    traces_enabled: bool = True
-    metrics_enabled: bool = True
-    log_correlation: bool = True
-    console_export: bool = False
-    extra_attributes: dict = field(default_factory=dict)
-
-    @classmethod
-    def from_env(cls, agent_name: Optional[str] = None) -> "TelemetryConfig":
-        """Create configuration from environment variables."""
-        enabled = os.getenv("OTEL_ENABLED", "false").lower() in ("true", "1", "yes")
-
-        return cls(
-            enabled=enabled,
-            service_name=os.getenv("OTEL_SERVICE_NAME", agent_name or "kaos-agent"),
-            service_version=os.getenv("OTEL_SERVICE_VERSION", "0.0.1"),
-            otlp_endpoint=os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4317"),
-            otlp_insecure=os.getenv("OTEL_EXPORTER_OTLP_INSECURE", "true").lower()
-            in ("true", "1", "yes"),
-            traces_enabled=os.getenv("OTEL_TRACES_ENABLED", "true").lower() in ("true", "1", "yes"),
-            metrics_enabled=os.getenv("OTEL_METRICS_ENABLED", "true").lower()
-            in ("true", "1", "yes"),
-            log_correlation=os.getenv("OTEL_LOG_CORRELATION", "true").lower()
-            in ("true", "1", "yes"),
-            console_export=os.getenv("OTEL_CONSOLE_EXPORT", "false").lower()
-            in ("true", "1", "yes"),
-        )
-
-
-# Global state for initialized providers
-_tracer_provider: Optional[TracerProvider] = None
-_meter_provider: Optional[MeterProvider] = None
-_initialized: bool = False
-
-
-def init_telemetry(config: TelemetryConfig) -> bool:
-    """Initialize OpenTelemetry with the given configuration.
-
-    Returns True if telemetry was initialized, False if disabled or already initialized.
-    """
-    global _tracer_provider, _meter_provider, _initialized
-
-    if _initialized:
-        logger.debug("Telemetry already initialized")
-        return False
-
-    if not config.enabled:
-        logger.info("OpenTelemetry disabled (OTEL_ENABLED=false)")
-        _initialized = True
-        return False
-
-    # Create resource with service information
-    resource = Resource.create(
-        {
-            SERVICE_NAME: config.service_name,
-            SERVICE_VERSION: config.service_version,
-            "deployment.environment": os.getenv("DEPLOYMENT_ENVIRONMENT", "development"),
-            **config.extra_attributes,
-        }
-    )
-
-    # Set up context propagation (W3C Trace Context + Baggage)
-    set_global_textmap(
-        CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
-    )
-
-    # Initialize tracing
-    if config.traces_enabled:
-        _tracer_provider = TracerProvider(resource=resource)
-
-        if config.console_export:
-            _tracer_provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
-
-        if config.otlp_endpoint:
-            otlp_exporter = OTLPSpanExporter(
-                endpoint=config.otlp_endpoint,
-                insecure=config.otlp_insecure,
-            )
-            _tracer_provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
-
-        trace.set_tracer_provider(_tracer_provider)
-        logger.info(f"OpenTelemetry tracing initialized: {config.otlp_endpoint}")
-
-    # Initialize metrics
-    if config.metrics_enabled:
-        readers = []
-
-        if config.console_export:
-            readers.append(PeriodicExportingMetricReader(ConsoleMetricExporter()))
-
-        if config.otlp_endpoint:
-            otlp_metric_exporter = OTLPMetricExporter(
-                endpoint=config.otlp_endpoint,
-                insecure=config.otlp_insecure,
-            )
-            readers.append(PeriodicExportingMetricReader(otlp_metric_exporter))
-
-        if readers:
-            _meter_provider = MeterProvider(resource=resource, metric_readers=readers)
-            metrics.set_meter_provider(_meter_provider)
-            logger.info(f"OpenTelemetry metrics initialized: {config.otlp_endpoint}")
-
-    _initialized = True
-    return True
-
-
-def shutdown_telemetry() -> None:
-    """Shutdown OpenTelemetry providers gracefully."""
-    global _tracer_provider, _meter_provider, _initialized
-
-    if _tracer_provider:
-        _tracer_provider.shutdown()
-        _tracer_provider = None
-
-    if _meter_provider:
-        _meter_provider.shutdown()
-        _meter_provider = None
-
-    _initialized = False
-    logger.info("OpenTelemetry shutdown complete")
-
-
-def is_telemetry_enabled() -> bool:
-    """Check if telemetry is enabled and initialized."""
-    return _initialized and (_tracer_provider is not None or _meter_provider is not None)
diff --git a/python/agent/telemetry/manager.py b/python/agent/telemetry/manager.py
new file mode 100644
index 0000000..e0bad90
--- /dev/null
+++ b/python/agent/telemetry/manager.py
@@ -0,0 +1,292 @@
+"""
+OpenTelemetry Manager for KAOS.
+
+Provides a simplified interface for OpenTelemetry instrumentation using standard
+OTEL_* environment variables. When OTEL_ENABLED=true, traces, metrics, and log
+correlation are all enabled.
+"""
+
+import logging
+import os
+import time
+from contextlib import contextmanager
+from dataclasses import dataclass
+from typing import Any, Dict, Iterator, Optional
+
+from opentelemetry import trace, metrics
+from opentelemetry.sdk.trace import TracerProvider
+from opentelemetry.sdk.trace.export import BatchSpanProcessor
+from opentelemetry.sdk.metrics import MeterProvider
+from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
+from opentelemetry.sdk.resources import Resource, SERVICE_NAME
+from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
+from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
+from opentelemetry.propagate import set_global_textmap, inject, extract
+from opentelemetry.propagators.composite import CompositePropagator
+from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
+from opentelemetry.baggage.propagation import W3CBaggagePropagator
+from opentelemetry.trace import Span, SpanKind, Status, StatusCode
+from opentelemetry.context import Context
+
+logger = logging.getLogger(__name__)
+
+# Semantic conventions for KAOS spans
+ATTR_AGENT_NAME = "agent.name"
+ATTR_SESSION_ID = "session.id"
+ATTR_MODEL_NAME = "gen_ai.request.model"
+ATTR_TOOL_NAME = "tool.name"
+ATTR_DELEGATION_TARGET = "agent.delegation.target"
+
+# Process-global initialization state
+_initialized: bool = False
+
+
+@dataclass
+class OtelConfig:
+    """OpenTelemetry configuration from environment variables."""
+
+    enabled: bool
+    service_name: str
+    endpoint: str
+
+    @classmethod
+    def from_env(cls, default_service_name: str = "kaos") -> "OtelConfig":
+        """Create config from standard OTEL_* environment variables."""
+        return cls(
+            enabled=os.getenv("OTEL_ENABLED", "false").lower() in ("true", "1", "yes"),
+            service_name=os.getenv("OTEL_SERVICE_NAME", default_service_name),
+            endpoint=os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4317"),
+        )
+
+
+def init_otel(service_name: Optional[str] = None) -> bool:
+    """Initialize OpenTelemetry with standard OTEL_* env vars.
+
+    Should be called once at process startup. Idempotent - safe to call multiple times.
+
+    Args:
+        service_name: Default service name if OTEL_SERVICE_NAME not set
+
+    Returns:
+        True if OTel was initialized, False if disabled or already initialized
+    """
+    global _initialized
+    if _initialized:
+        return False
+
+    config = OtelConfig.from_env(service_name or "kaos")
+    if not config.enabled:
+        logger.debug("OpenTelemetry disabled (OTEL_ENABLED != true)")
+        _initialized = True
+        return False
+
+    # Create resource with service name
+    resource = Resource.create({SERVICE_NAME: config.service_name})
+
+    # Set up W3C Trace Context propagation
+    set_global_textmap(
+        CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
+    )
+
+    # Initialize tracing
+    tracer_provider = TracerProvider(resource=resource)
+    otlp_span_exporter = OTLPSpanExporter(endpoint=config.endpoint, insecure=True)
+    tracer_provider.add_span_processor(BatchSpanProcessor(otlp_span_exporter))
+    trace.set_tracer_provider(tracer_provider)
+
+    # Initialize metrics
+    otlp_metric_exporter = OTLPMetricExporter(endpoint=config.endpoint, insecure=True)
+    metric_reader = PeriodicExportingMetricReader(otlp_metric_exporter)
+    meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
+    metrics.set_meter_provider(meter_provider)
+
+    logger.info(f"OpenTelemetry initialized: {config.endpoint} (service: {config.service_name})")
+    _initialized = True
+    return True
+
+
+def is_otel_enabled() -> bool:
+    """Check if OTel is enabled via environment variable."""
+    return os.getenv("OTEL_ENABLED", "false").lower() in ("true", "1", "yes")
+
+
+class KaosOtelManager:
+    """Lightweight helper for creating spans and recording metrics.
+
+    Provides convenience methods for KAOS-specific telemetry. Each server/client
+    should create one instance. The underlying providers are process-global.
+
+    Example:
+        otel = KaosOtelManager("my-agent")
+        with otel.span("process_request", session_id="abc123") as span:
+            # do work
+            span.set_attribute("custom", "value")
+    """
+
+    def __init__(self, service_name: str):
+        """Initialize manager with service context.
+
+        Args:
+            service_name: Name of the service (e.g., agent name)
+        """
+        self.service_name = service_name
+        self._tracer = trace.get_tracer(f"kaos.{service_name}")
+        self._meter = metrics.get_meter(f"kaos.{service_name}")
+
+        # Lazily initialized metrics
+        self._request_counter: Optional[metrics.Counter] = None
+        self._request_duration: Optional[metrics.Histogram] = None
+        self._model_counter: Optional[metrics.Counter] = None
+        self._model_duration: Optional[metrics.Histogram] = None
+        self._tool_counter: Optional[metrics.Counter] = None
+        self._tool_duration: Optional[metrics.Histogram] = None
+        self._delegation_counter: Optional[metrics.Counter] = None
+        self._delegation_duration: Optional[metrics.Histogram] = None
+
+    def _ensure_metrics(self) -> None:
+        """Lazily initialize metric instruments."""
+        if self._request_counter is not None:
+            return
+
+        self._request_counter = self._meter.create_counter(
+            "kaos.requests", description="Request count", unit="1"
+        )
+        self._request_duration = self._meter.create_histogram(
+            "kaos.request.duration", description="Request duration", unit="ms"
+        )
+        self._model_counter = self._meter.create_counter(
+            "kaos.model.calls", description="Model API call count", unit="1"
+        )
+        self._model_duration = self._meter.create_histogram(
+            "kaos.model.duration", description="Model API call duration", unit="ms"
+        )
+        self._tool_counter = self._meter.create_counter(
+            "kaos.tool.calls", description="Tool call count", unit="1"
+        )
+        self._tool_duration = self._meter.create_histogram(
+            "kaos.tool.duration", description="Tool call duration", unit="ms"
+        )
+        self._delegation_counter = self._meter.create_counter(
+            "kaos.delegations", description="Delegation count", unit="1"
+        )
+        self._delegation_duration = self._meter.create_histogram(
+            "kaos.delegation.duration", description="Delegation duration", unit="ms"
+        )
+
+    @contextmanager
+    def span(
+        self,
+        name: str,
+        kind: SpanKind = SpanKind.INTERNAL,
+        session_id: Optional[str] = None,
+        **attributes: Any,
+    ) -> Iterator[Span]:
+        """Create a span with automatic end and status handling.
+
+        Args:
+            name: Span name
+            kind: Span kind (INTERNAL, CLIENT, SERVER)
+            session_id: Optional session ID to attach
+            **attributes: Additional span attributes
+
+        Yields:
+            The active span
+        """
+        attrs = {ATTR_AGENT_NAME: self.service_name}
+        if session_id:
+            attrs[ATTR_SESSION_ID] = session_id
+        attrs.update({k: v for k, v in attributes.items() if v is not None})
+
+        with self._tracer.start_as_current_span(name, kind=kind, attributes=attrs) as span:
+            try:
+                yield span
+                span.set_status(Status(StatusCode.OK))
+            except Exception as e:
+                span.set_status(Status(StatusCode.ERROR, str(e)))
+                span.record_exception(e)
+                raise
+
+    @contextmanager
+    def model_span(self, model_name: str) -> Iterator[Span]:
+        """Create a span for model API calls."""
+        with self.span("model.inference", SpanKind.CLIENT, **{ATTR_MODEL_NAME: model_name}) as s:
+            yield s
+
+    @contextmanager
+    def tool_span(self, tool_name: str) -> Iterator[Span]:
+        """Create a span for tool execution."""
+        with self.span(f"tool.{tool_name}", SpanKind.CLIENT, **{ATTR_TOOL_NAME: tool_name}) as s:
+            yield s
+
+    @contextmanager
+    def delegation_span(self, target_agent: str) -> Iterator[Span]:
+        """Create a span for A2A delegation."""
+        with self.span(
+            f"delegate.{target_agent}", SpanKind.CLIENT, **{ATTR_DELEGATION_TARGET: target_agent}
+        ) as s:
+            yield s
+
+    def record_request(self, duration_ms: float, success: bool = True) -> None:
+        """Record request metrics."""
+        self._ensure_metrics()
+        labels = {"agent.name": self.service_name, "success": str(success).lower()}
+        if self._request_counter:
+            self._request_counter.add(1, labels)
+        if self._request_duration:
+            self._request_duration.record(duration_ms, labels)
+
+    def record_model_call(self, model: str, duration_ms: float, success: bool = True) -> None:
+        """Record model API call metrics."""
+        self._ensure_metrics()
+        labels = {"agent.name": self.service_name, "model": model, "success": str(success).lower()}
+        if self._model_counter:
+            self._model_counter.add(1, labels)
+        if self._model_duration:
+            self._model_duration.record(duration_ms, labels)
+
+    def record_tool_call(self, tool: str, duration_ms: float, success: bool = True) -> None:
+        """Record tool call metrics."""
+        self._ensure_metrics()
+        labels = {"agent.name": self.service_name, "tool": tool, "success": str(success).lower()}
+        if self._tool_counter:
+            self._tool_counter.add(1, labels)
+        if self._tool_duration:
+            self._tool_duration.record(duration_ms, labels)
+
+    def record_delegation(self, target: str, duration_ms: float, success: bool = True) -> None:
+        """Record delegation metrics."""
+        self._ensure_metrics()
+        labels = {
+            "agent.name": self.service_name,
+            "target": target,
+            "success": str(success).lower(),
+        }
+        if self._delegation_counter:
+            self._delegation_counter.add(1, labels)
+        if self._delegation_duration:
+            self._delegation_duration.record(duration_ms, labels)
+
+    @staticmethod
+    def inject_context(carrier: Dict[str, str]) -> Dict[str, str]:
+        """Inject trace context into headers for propagation."""
+        inject(carrier)
+        return carrier
+
+    @staticmethod
+    def extract_context(carrier: Dict[str, str]) -> Context:
+        """Extract trace context from headers."""
+        return extract(carrier)
+
+
+@contextmanager
+def timed() -> Iterator[Dict[str, float]]:
+    """Context manager for timing operations.
+
+    Yields a dict that will contain 'duration_ms' after the block.
+    """
+    result: Dict[str, float] = {}
+    start = time.perf_counter()
+    try:
+        yield result
+    finally:
+        result["duration_ms"] = (time.perf_counter() - start) * 1000
diff --git a/python/agent/telemetry/metrics.py b/python/agent/telemetry/metrics.py
deleted file mode 100644
index 1305ea6..0000000
--- a/python/agent/telemetry/metrics.py
+++ /dev/null
@@ -1,248 +0,0 @@
-"""
-Metrics utilities for KAOS agents.
-
-Provides counters, histograms, and gauges for:
-- Request processing
-- Model API calls
-- Tool execution
-- A2A delegation
-- Memory/session management
-"""
-
-import logging
-import time
-from contextlib import contextmanager
-from typing import Any, Dict, Iterator, Optional
-
-from opentelemetry import metrics
-from opentelemetry.metrics import Counter, Histogram, UpDownCounter
-
-logger = logging.getLogger(__name__)
-
-# Meter name for KAOS components
-METER_NAME = "kaos.agent"
-
-# Cached meter and instruments
-_meter: Optional[metrics.Meter] = None
-_request_counter: Optional[Counter] = None
-_request_duration: Optional[Histogram] = None
-_model_call_counter: Optional[Counter] = None
-_model_call_duration: Optional[Histogram] = None
-_tool_call_counter: Optional[Counter] = None
-_tool_call_duration: Optional[Histogram] = None
-_delegation_counter: Optional[Counter] = None
-_delegation_duration: Optional[Histogram] = None
-_active_sessions: Optional[UpDownCounter] = None
-_agentic_steps: Optional[Counter] = None
-
-
-def get_meter(name: str = METER_NAME) -> metrics.Meter:
-    """Get a meter for creating instruments."""
-    return metrics.get_meter(name)
-
-
-def _ensure_instruments() -> None:
-    """Lazily initialize metric instruments."""
-    global _meter, _request_counter, _request_duration
-    global _model_call_counter, _model_call_duration
-    global _tool_call_counter, _tool_call_duration
-    global _delegation_counter, _delegation_duration
-    global _active_sessions, _agentic_steps
-
-    if _meter is not None:
-        return
-
-    _meter = get_meter()
-
-    # Request metrics
-    _request_counter = _meter.create_counter(
-        name="kaos.agent.requests",
-        description="Number of agent requests processed",
-        unit="1",
-    )
-    _request_duration = _meter.create_histogram(
-        name="kaos.agent.request.duration",
-        description="Duration of agent request processing",
-        unit="ms",
-    )
-
-    # Model API metrics
-    _model_call_counter = _meter.create_counter(
-        name="kaos.agent.model.calls",
-        description="Number of model API calls",
-        unit="1",
-    )
-    _model_call_duration = _meter.create_histogram(
-        name="kaos.agent.model.duration",
-        description="Duration of model API calls",
-        unit="ms",
-    )
-
-    # Tool execution metrics
-    _tool_call_counter = _meter.create_counter(
-        name="kaos.agent.tool.calls",
-        description="Number of tool calls",
-        unit="1",
-    )
-    _tool_call_duration = _meter.create_histogram(
-        name="kaos.agent.tool.duration",
-        description="Duration of tool calls",
-        unit="ms",
-    )
-
-    # Delegation metrics
-    _delegation_counter = _meter.create_counter(
-        name="kaos.agent.delegations",
-        description="Number of A2A delegations",
-        unit="1",
-    )
-    _delegation_duration = _meter.create_histogram(
-        name="kaos.agent.delegation.duration",
-        description="Duration of A2A delegations",
-        unit="ms",
-    )
-
-    # Session metrics
-    _active_sessions = _meter.create_up_down_counter(
-        name="kaos.agent.sessions.active",
-        description="Number of active sessions",
-        unit="1",
-    )
-
-    # Agentic loop metrics
-    _agentic_steps = _meter.create_counter(
-        name="kaos.agent.agentic.steps",
-        description="Number of agentic loop steps",
-        unit="1",
-    )
-
-
-def record_request(
-    agent_name: str,
-    duration_ms: float,
-    success: bool = True,
-    stream: bool = False,
-    **attributes: Any,
-) -> None:
-    """Record a request metric."""
-    _ensure_instruments()
-    if _request_counter and _request_duration:
-        labels = {
-            "agent.name": agent_name,
-            "success": str(success).lower(),
-            "stream": str(stream).lower(),
-            **{k: str(v) for k, v in attributes.items() if v is not None},
-        }
```

---

### Commit: 8932ef9
**Subject:** docs(telemetry): update for simplified CRD configuration
**Date:** 2026-01-25 14:20:39 +0100
**Author:** Alejandro Saucedo

- Document simplified CRD with just enabled + endpoint fields
- Add MCPServer telemetry example
- Document advanced config via standard OTEL_* env vars
- Update span hierarchy to reflect FastAPI auto-instrumentation
- Remove obsolete fields (insecure, serviceName, tracesEnabled, etc.)


```diff
 docs/operator/telemetry.md | 130 +++++++++++++++++++++++++++++----------------
 1 file changed, 85 insertions(+), 45 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index b2454f4..c05992b 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -1,6 +1,6 @@
 # OpenTelemetry
 
-KAOS supports OpenTelemetry for observability, including distributed tracing, metrics, and log correlation across all agent operations.
+KAOS supports OpenTelemetry for observability, including distributed tracing, metrics, and log correlation across all agent and MCP server operations.
 
 ## Overview
 
@@ -12,7 +12,7 @@ When enabled, OpenTelemetry instrumentation provides:
 
 ## Enabling Telemetry
 
-Add a `telemetry` section to your Agent's config:
+Add a `telemetry` section to your Agent's or MCPServer's config:
 
 ```yaml
 apiVersion: kaos.tools/v1alpha1
@@ -27,69 +27,115 @@ spec:
     telemetry:
       enabled: true
       endpoint: "http://otel-collector.monitoring.svc.cluster.local:4317"
-      insecure: true
+```
+
+```yaml
+apiVersion: kaos.tools/v1alpha1
+kind: MCPServer
+metadata:
+  name: my-tools
+spec:
+  type: python-runtime
+  config:
+    telemetry:
+      enabled: true
+      endpoint: "http://otel-collector.monitoring.svc.cluster.local:4317"
+    tools:
+      fromString: |
+        def echo(msg: str) -> str:
+            return msg
 ```
 
 ## Configuration Fields
 
+The CRD configuration is intentionally minimal. For advanced settings, use standard `OTEL_*` environment variables via `spec.config.env`.
+
 | Field | Type | Default | Description |
 |-------|------|---------|-------------|
 | `enabled` | bool | `false` | Enable OpenTelemetry instrumentation |
-| `endpoint` | string | `http://localhost:4317` | OTLP exporter endpoint |
-| `insecure` | bool | `true` | Use insecure connection (no TLS) |
-| `serviceName` | string | Agent name | Service name for traces/metrics |
-| `tracesEnabled` | bool | `true` | Enable trace collection |
-| `metricsEnabled` | bool | `true` | Enable metrics collection |
-| `logCorrelation` | bool | `true` | Inject trace context into logs |
+| `endpoint` | string | `http://localhost:4317` | OTLP exporter endpoint (gRPC) |
+
+### Advanced Configuration via Environment Variables
+
+For advanced configuration, use the standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html):
+
+```yaml
+spec:
+  config:
+    telemetry:
+      enabled: true
+      endpoint: "http://otel-collector:4317"
+    env:
+    - name: OTEL_EXPORTER_OTLP_INSECURE
+      value: "false"  # Use TLS
+    - name: OTEL_EXPORTER_OTLP_HEADERS
+      value: "x-api-key=YOUR_KEY"
+    - name: OTEL_TRACES_SAMPLER
+      value: "parentbased_traceidratio"
+    - name: OTEL_TRACES_SAMPLER_ARG
+      value: "0.1"  # Sample 10% of traces
+    - name: OTEL_RESOURCE_ATTRIBUTES
+      value: "deployment.environment=production"
+```
+
+The operator automatically sets:
+- `OTEL_ENABLED`: Set to "true" when telemetry is enabled
+- `OTEL_SERVICE_NAME`: Defaults to the CR name (e.g., agent name)
+- `OTEL_EXPORTER_OTLP_ENDPOINT`: From `telemetry.endpoint`
+- `OTEL_RESOURCE_ATTRIBUTES`: Includes `service.namespace` and `kaos.resource.name`
 
 ## Trace Spans
 
 The following spans are automatically created:
 
-### agent.process_message
+### HTTP Request (auto-instrumented)
 
-Root span for each request to the agent. Attributes:
+FastAPI/Starlette auto-instrumentation creates the root SERVER span for each HTTP request. This provides standard HTTP attributes like `http.method`, `http.url`, `http.status_code`.
+
+### agent.agentic_loop
+
+Main processing span for agent reasoning. Attributes:
 - `agent.name`: Name of the agent
 - `session.id`: Session identifier
+- `agent.max_steps`: Maximum reasoning steps
+- `stream`: Whether streaming is enabled
 
 ### agent.step.{n}
 
 Span for each iteration of the agentic reasoning loop. Attributes:
-- `agent.step`: Step number (1-based)
-- `agent.name`: Agent name
+- `step`: Step number (1-based)
+- `max_steps`: Maximum allowed steps
 
 ### model.inference
 
 Span for LLM API calls. Attributes:
-- `model.name`: Model identifier
-- `model.api_url`: API endpoint
+- `gen_ai.request.model`: Model identifier
 
 ### tool.{name}
 
 Span for MCP tool executions. Attributes:
 - `tool.name`: Tool name
-- `mcp.server`: MCP server name
 
 ### delegate.{agent}
 
 Span for agent-to-agent delegations. Attributes:
-- `delegation.target`: Target agent name
-- `delegation.task`: Task description
+- `agent.delegation.target`: Target agent name
 
 ## Span Hierarchy Example
 
 ```
-agent.process_message (SERVER)
-├── agent.step.1 (INTERNAL)
-│   └── model.inference (CLIENT)
-├── agent.step.2 (INTERNAL)
-│   ├── model.inference (CLIENT)
-│   └── tool.calculator (CLIENT)
-├── agent.step.3 (INTERNAL)
-│   ├── model.inference (CLIENT)
-│   └── delegate.researcher (CLIENT)
-└── agent.step.4 (INTERNAL)
-    └── model.inference (CLIENT)
+HTTP POST /v1/chat/completions (SERVER, auto-instrumented)
+└── agent.agentic_loop (INTERNAL)
+    ├── agent.step.1 (INTERNAL)
+    │   └── model.inference (CLIENT)
+    ├── agent.step.2 (INTERNAL)
+    │   ├── model.inference (CLIENT)
+    │   └── tool.calculator (CLIENT)
+    ├── agent.step.3 (INTERNAL)
+    │   ├── model.inference (CLIENT)
+    │   └── delegate.researcher (CLIENT)
+    └── agent.step.4 (INTERNAL)
+        └── model.inference (CLIENT)
 ```
 
 ## Metrics
@@ -120,7 +166,7 @@ Delegation metrics also include:
 
 ## Log Correlation
 
-When `logCorrelation` is enabled, log entries include trace context:
+When OpenTelemetry is enabled, log entries automatically include trace context:
 
 ```
 2024-01-15 10:30:45 INFO [trace_id=abc123 span_id=def456] Processing message...
@@ -146,11 +192,6 @@ spec:
     telemetry:
       enabled: true
       endpoint: "http://otel-collector.monitoring.svc.cluster.local:4317"
-      insecure: true
-      serviceName: "traced-agent"
-      tracesEnabled: true
-      metricsEnabled: true
-      logCorrelation: true
   agentNetwork:
     access:
     - researcher
@@ -284,17 +325,16 @@ config:
 
 ## Environment Variables
 
-For advanced configuration, the following environment variables are passed to agent pods when telemetry is enabled:
+The operator automatically sets these environment variables when telemetry is enabled:
 
 | Variable | Description |
 |----------|-------------|
 | `OTEL_ENABLED` | "true" when telemetry is enabled |
-| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL |
-| `OTEL_EXPORTER_OTLP_INSECURE` | "true" for insecure connections |
-| `OTEL_SERVICE_NAME` | Service name for traces/metrics |
-| `OTEL_TRACES_ENABLED` | "true" when traces are enabled |
-| `OTEL_METRICS_ENABLED` | "true" when metrics are enabled |
-| `OTEL_LOG_CORRELATION` | "true" when log correlation is enabled |
+| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL from `telemetry.endpoint` |
+| `OTEL_SERVICE_NAME` | Defaults to CR name (agent or MCP server name) |
+| `OTEL_RESOURCE_ATTRIBUTES` | Includes `service.namespace` and `kaos.resource.name` |
+
+For additional configuration, use standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html) via `spec.config.env`.
 
 ## Troubleshooting
 
@@ -319,11 +359,11 @@ kubectl exec -it deploy/agent-my-agent -- curl -v http://otel-collector.monitori
 
 If telemetry adds noticeable latency:
 - Use batching in the OTel collector
-- Consider sampling for high-throughput agents
-- Disable metrics if only traces are needed
+- Configure sampling via `OTEL_TRACES_SAMPLER` and `OTEL_TRACES_SAMPLER_ARG` env vars
 
 ### Missing spans
 
 Ensure all sub-agents and MCP servers are instrumented:
 - Each agent should have its own telemetry config
-- MCP servers share the agent's trace context via W3C Trace Context headers
+- MCP servers should also have telemetry enabled
+- Trace context propagates automatically via W3C Trace Context headers
```

---

### Commit: 39c69af
**Subject:** fix(deps): add opentelemetry-instrumentation-starlette to pyproject.toml
**Date:** 2026-01-25 14:23:56 +0100
**Author:** Alejandro Saucedo



```diff
 python/pyproject.toml | 1 +
 1 file changed, 1 insertion(+)
```

#### Detailed Changes

```diff
diff --git a/python/pyproject.toml b/python/pyproject.toml
index 49030e4..8fa08c8 100644
--- a/python/pyproject.toml
+++ b/python/pyproject.toml
@@ -19,6 +19,7 @@ dependencies = [
     "opentelemetry-instrumentation-fastapi>=0.41b0",
     "opentelemetry-instrumentation-httpx>=0.41b0",
     "opentelemetry-instrumentation-logging>=0.41b0",
+    "opentelemetry-instrumentation-starlette>=0.41b0",
     # OpenTelemetry exporters
     "opentelemetry-exporter-otlp>=1.20.0",
 ]
```

---

### Commit: 21ac2e8
**Subject:** refactor(telemetry): move to library scope, use OTEL_SDK_DISABLED, BaseSettings
**Date:** 2026-01-25 17:11:30 +0100
**Author:** Alejandro Saucedo

- Move telemetry module from python/agent/telemetry/ to python/telemetry/
- Use OTEL_SDK_DISABLED (standard OTel env var) instead of OTEL_ENABLED
- Convert OtelConfig to pydantic BaseSettings with required fields
- Add _OtelSingleton for process-global SDK state management
- Fix BuildTelemetryEnvVars to append (not overwrite) OTEL_RESOURCE_ATTRIBUTES
- Track request success/failure properly in record_request metrics
- Remove duplicate LoggingInstrumentor call in agent/server.py
- Update tests and documentation


```diff
 docs/operator/telemetry.md              |  10 +-
 operator/pkg/util/telemetry.go          |  26 +++--
 python/agent/client.py                  |   6 +-
 python/agent/server.py                  |  12 +--
 python/agent/telemetry/__init__.py      |  22 ----
 python/mcptools/server.py               |   7 +-
 python/telemetry/__init__.py            |   6 ++
 python/{agent => }/telemetry/manager.py | 176 ++++++++++++++++++++++----------
 python/tests/test_telemetry.py          | 133 +++++++++++++-----------
 9 files changed, 236 insertions(+), 162 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index c05992b..b3724d2 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -53,7 +53,7 @@ The CRD configuration is intentionally minimal. For advanced settings, use stand
 | Field | Type | Default | Description |
 |-------|------|---------|-------------|
 | `enabled` | bool | `false` | Enable OpenTelemetry instrumentation |
-| `endpoint` | string | `http://localhost:4317` | OTLP exporter endpoint (gRPC) |
+| `endpoint` | string | - | OTLP exporter endpoint (gRPC, required when enabled) |
 
 ### Advanced Configuration via Environment Variables
 
@@ -79,10 +79,10 @@ spec:
 ```
 
 The operator automatically sets:
-- `OTEL_ENABLED`: Set to "true" when telemetry is enabled
+- `OTEL_SDK_DISABLED`: Set to "false" when telemetry is enabled (standard OTel env var)
 - `OTEL_SERVICE_NAME`: Defaults to the CR name (e.g., agent name)
 - `OTEL_EXPORTER_OTLP_ENDPOINT`: From `telemetry.endpoint`
-- `OTEL_RESOURCE_ATTRIBUTES`: Includes `service.namespace` and `kaos.resource.name`
+- `OTEL_RESOURCE_ATTRIBUTES`: Appends `service.namespace` and `kaos.resource.name` to any user-provided values
 
 ## Trace Spans
 
@@ -329,10 +329,10 @@ The operator automatically sets these environment variables when telemetry is en
 
 | Variable | Description |
 |----------|-------------|
-| `OTEL_ENABLED` | "true" when telemetry is enabled |
+| `OTEL_SDK_DISABLED` | "false" when telemetry is enabled (standard OTel env var) |
 | `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL from `telemetry.endpoint` |
 | `OTEL_SERVICE_NAME` | Defaults to CR name (agent or MCP server name) |
-| `OTEL_RESOURCE_ATTRIBUTES` | Includes `service.namespace` and `kaos.resource.name` |
+| `OTEL_RESOURCE_ATTRIBUTES` | Appends `service.namespace` and `kaos.resource.name` to user values |
 
 For additional configuration, use standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html) via `spec.config.env`.
 
diff --git a/operator/pkg/util/telemetry.go b/operator/pkg/util/telemetry.go
index 94af18a..b68fad1 100644
--- a/operator/pkg/util/telemetry.go
+++ b/operator/pkg/util/telemetry.go
@@ -1,6 +1,9 @@
 package util
 
 import (
+	"os"
+	"strings"
+
 	corev1 "k8s.io/api/core/v1"
 
 	kaosv1alpha1 "github.com/axsaucedo/kaos/operator/api/v1alpha1"
@@ -9,7 +12,7 @@ import (
 // BuildTelemetryEnvVars creates environment variables for OpenTelemetry configuration.
 // Uses standard OTEL_* env vars so the SDK auto-configures.
 // serviceName is used as OTEL_SERVICE_NAME (typically the CR name).
-// namespace is added to OTEL_RESOURCE_ATTRIBUTES.
+// namespace is added to OTEL_RESOURCE_ATTRIBUTES (appended to existing user values).
 func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, namespace string) []corev1.EnvVar {
 	if tel == nil || !tel.Enabled {
 		return nil
@@ -17,8 +20,8 @@ func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, names
 
 	envVars := []corev1.EnvVar{
 		{
-			Name:  "OTEL_ENABLED",
-			Value: "true",
+			Name:  "OTEL_SDK_DISABLED",
+			Value: "false",
 		},
 		{
 			Name:  "OTEL_SERVICE_NAME",
@@ -33,11 +36,22 @@ func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, names
 		})
 	}
 
-	// Add resource attributes for Kubernetes context
-	resourceAttrs := "service.namespace=" + namespace + ",kaos.resource.name=" + serviceName
+	// Build resource attributes - append to existing user values if OTEL_RESOURCE_ATTRIBUTES is set
+	kaosAttrs := "service.namespace=" + namespace + ",kaos.resource.name=" + serviceName
+	existingAttrs := os.Getenv("OTEL_RESOURCE_ATTRIBUTES")
+	var finalAttrs string
+	if existingAttrs != "" {
+		// Append KAOS attrs to user attrs (user attrs take precedence for same keys)
+		finalAttrs = existingAttrs + "," + kaosAttrs
+	} else {
+		finalAttrs = kaosAttrs
+	}
+	// Remove any duplicate trailing/leading commas
+	finalAttrs = strings.Trim(finalAttrs, ",")
+
 	envVars = append(envVars, corev1.EnvVar{
 		Name:  "OTEL_RESOURCE_ATTRIBUTES",
-		Value: resourceAttrs,
+		Value: finalAttrs,
 	})
 
 	return envVars
diff --git a/python/agent/client.py b/python/agent/client.py
index 12eed20..7a0bd8b 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -22,7 +22,7 @@ from dataclasses import dataclass
 from modelapi.client import ModelAPI
 from agent.memory import LocalMemory, NullMemory
 from mcptools.client import MCPClient
-from agent.telemetry import KaosOtelManager, timed
+from telemetry.manager import KaosOtelManager, timed
 
 logger = logging.getLogger(__name__)
 
@@ -317,6 +317,7 @@ class Agent:
             that will be used instead of calling the model API.
         """
         # Use the KaosOtelManager for simpler telemetry
+        request_success = True
         with timed() as timing:
             # Get or create session
             if session_id:
@@ -375,6 +376,7 @@ class Agent:
                         yield chunk
 
                 except Exception as e:
+                    request_success = False
                     error_msg = f"Error processing message: {str(e)}"
                     logger.error(error_msg)
                     error_event = self.memory.create_event("error", error_msg)
@@ -382,7 +384,7 @@ class Agent:
                     yield f"Sorry, I encountered an error: {str(e)}"
 
         # Record request metrics after completion
-        self._otel.record_request(timing["duration_ms"], success=True)
+        self._otel.record_request(timing["duration_ms"], success=request_success)
 
     async def _agentic_loop(
         self,
diff --git a/python/agent/server.py b/python/agent/server.py
index 602f442..45ee04f 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -23,7 +23,7 @@ from modelapi.client import ModelAPI
 from agent.client import Agent, RemoteAgent
 from agent.memory import LocalMemory
 from mcptools.client import MCPClient
-from agent.telemetry import init_otel, is_otel_enabled
+from telemetry.manager import init_otel, is_otel_enabled
 
 
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
@@ -547,17 +547,9 @@ def create_agent_server(
         memory = NullMemory()
 
     # Initialize OpenTelemetry if enabled (uses standard OTEL_* env vars)
+    # Note: LoggingInstrumentor is already called in configure_logging() above
     init_otel(settings.agent_name)
 
-    # Configure log correlation if OTel is enabled
-    if is_otel_enabled():
-        try:
-            from opentelemetry.instrumentation.logging import LoggingInstrumentor
-
-            LoggingInstrumentor().instrument(set_logging_format=False)
-        except Exception as e:
-            logger.warning(f"Failed to enable OTel log correlation: {e}")
-
     agent = Agent(
         name=settings.agent_name,
         description=settings.agent_description,
diff --git a/python/agent/telemetry/__init__.py b/python/agent/telemetry/__init__.py
deleted file mode 100644
index 83b38d5..0000000
--- a/python/agent/telemetry/__init__.py
+++ /dev/null
@@ -1,22 +0,0 @@
-"""
-OpenTelemetry instrumentation for KAOS.
-
-Simple interface using standard OTEL_* environment variables.
-When OTEL_ENABLED=true, traces, metrics, and log correlation are enabled.
-"""
-
-from agent.telemetry.manager import (
-    OtelConfig,
-    KaosOtelManager,
-    init_otel,
-    is_otel_enabled,
-    timed,
-)
-
-__all__ = [
-    "OtelConfig",
-    "KaosOtelManager",
-    "init_otel",
-    "is_otel_enabled",
-    "timed",
-]
diff --git a/python/mcptools/server.py b/python/mcptools/server.py
index bd61d01..41c6cd8 100644
--- a/python/mcptools/server.py
+++ b/python/mcptools/server.py
@@ -84,8 +84,9 @@ class MCPServer:
 
     def __init__(self, settings: MCPServerSettings):
         """Initialize MCP server."""
-        # Check if OTel enabled via env var
-        otel_enabled = os.getenv("OTEL_ENABLED", "false").lower() in ("true", "1", "yes")
+        # Check if OTel enabled via env var (OTEL_SDK_DISABLED=true disables)
+        otel_disabled = os.getenv("OTEL_SDK_DISABLED", "false").lower() in ("true", "1", "yes")
+        otel_enabled = not otel_disabled
 
         # Configure logging with optional OTel correlation
         configure_logging(settings.mcp_log_level, otel_correlation=otel_enabled)
@@ -109,7 +110,7 @@ class MCPServer:
     def _init_telemetry(self):
         """Initialize OpenTelemetry for the MCP server."""
         try:
-            from agent.telemetry import init_otel
+            from telemetry.manager import init_otel
 
             service_name = os.getenv("OTEL_SERVICE_NAME", "mcp-server")
             init_otel(service_name)
diff --git a/python/telemetry/__init__.py b/python/telemetry/__init__.py
new file mode 100644
index 0000000..65b24b4
--- /dev/null
+++ b/python/telemetry/__init__.py
@@ -0,0 +1,6 @@
+"""
+OpenTelemetry instrumentation for KAOS.
+
+Uses standard OTEL_* environment variables. When OTEL_SDK_DISABLED is not set or "false",
+traces, metrics, and log correlation are enabled (if endpoint is configured).
+"""
diff --git a/python/agent/telemetry/manager.py b/python/telemetry/manager.py
similarity index 64%
rename from python/agent/telemetry/manager.py
rename to python/telemetry/manager.py
index e0bad90..1e13cf7 100644
--- a/python/agent/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -2,17 +2,22 @@
 OpenTelemetry Manager for KAOS.
 
 Provides a simplified interface for OpenTelemetry instrumentation using standard
-OTEL_* environment variables. When OTEL_ENABLED=true, traces, metrics, and log
-correlation are all enabled.
+OTEL_* environment variables. Uses OTEL_SDK_DISABLED (standard OTel env var) to
+control whether telemetry is enabled.
+
+Key design:
+- Process-global SDK initialization via _OtelSingleton (providers, propagators)
+- Lightweight KaosOtelManager instances for per-service span/metric creation
+- OtelConfig uses pydantic BaseSettings with OTEL-compliant env var names
 """
 
 import logging
 import os
 import time
 from contextlib import contextmanager
-from dataclasses import dataclass
 from typing import Any, Dict, Iterator, Optional
 
+from pydantic_settings import BaseSettings, SettingsConfigDict
 from opentelemetry import trace, metrics
 from opentelemetry.sdk.trace import TracerProvider
 from opentelemetry.sdk.trace.export import BatchSpanProcessor
@@ -37,27 +42,102 @@ ATTR_MODEL_NAME = "gen_ai.request.model"
 ATTR_TOOL_NAME = "tool.name"
 ATTR_DELEGATION_TARGET = "agent.delegation.target"
 
-# Process-global initialization state
-_initialized: bool = False
 
+class OtelConfig(BaseSettings):
+    """OpenTelemetry configuration from standard OTEL_* environment variables.
+
+    Uses pydantic BaseSettings for automatic env var parsing.
+    OTEL_SDK_DISABLED=true disables telemetry (standard OTel env var).
+    """
+
+    model_config = SettingsConfigDict(env_prefix="", case_sensitive=False)
+
+    # Standard OTel env vars - required when telemetry enabled
+    otel_service_name: str
+    otel_exporter_otlp_endpoint: str
+
+    # Standard OTel env var for disabling SDK (default: false = enabled)
+    otel_sdk_disabled: bool = False
+
+    # Resource attributes (optional, we append to existing)
+    otel_resource_attributes: str = ""
+
+    @property
+    def enabled(self) -> bool:
+        """Check if OTel is enabled (not disabled)."""
+        return not self.otel_sdk_disabled
+
+
+class _OtelSingleton:
+    """Private singleton holding process-global OTel state.
+
+    SDK providers are process-global and should only be initialized once.
+    This class ensures idempotent initialization.
+    """
+
+    _instance: Optional["_OtelSingleton"] = None
+    _initialized: bool = False
+
+    def __new__(cls) -> "_OtelSingleton":
+        if cls._instance is None:
+            cls._instance = super().__new__(cls)
+        return cls._instance
+
+    def initialize(self, config: OtelConfig) -> bool:
+        """Initialize OTel SDK with given config. Idempotent.
+
+        Returns True if initialization happened, False if already initialized or disabled.
+        """
+        if self._initialized:
+            return False
 
-@dataclass
-class OtelConfig:
-    """OpenTelemetry configuration from environment variables."""
+        if not config.enabled:
+            logger.debug("OpenTelemetry disabled (OTEL_SDK_DISABLED=true)")
+            self._initialized = True
+            return False
 
-    enabled: bool
-    service_name: str
-    endpoint: str
+        # Create resource with service name
+        resource = Resource.create({SERVICE_NAME: config.otel_service_name})
 
-    @classmethod
-    def from_env(cls, default_service_name: str = "kaos") -> "OtelConfig":
-        """Create config from standard OTEL_* environment variables."""
-        return cls(
-            enabled=os.getenv("OTEL_ENABLED", "false").lower() in ("true", "1", "yes"),
-            service_name=os.getenv("OTEL_SERVICE_NAME", default_service_name),
-            endpoint=os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4317"),
+        # Set up W3C Trace Context propagation (standard)
+        set_global_textmap(
+            CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
         )
 
+        # Initialize tracing - let SDK use OTEL_EXPORTER_OTLP_* env vars for advanced config
+        tracer_provider = TracerProvider(resource=resource)
+        # Only set endpoint explicitly, let SDK handle insecure/headers via env vars
+        otlp_span_exporter = OTLPSpanExporter(endpoint=config.otel_exporter_otlp_endpoint)
+        tracer_provider.add_span_processor(BatchSpanProcessor(otlp_span_exporter))
+        trace.set_tracer_provider(tracer_provider)
+
+        # Initialize metrics
+        otlp_metric_exporter = OTLPMetricExporter(endpoint=config.otel_exporter_otlp_endpoint)
+        metric_reader = PeriodicExportingMetricReader(otlp_metric_exporter)
+        meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
+        metrics.set_meter_provider(meter_provider)
+
+        logger.info(
+            f"OpenTelemetry initialized: {config.otel_exporter_otlp_endpoint} "
+            f"(service: {config.otel_service_name})"
+        )
+        self._initialized = True
+        return True
+
+    @property
+    def is_initialized(self) -> bool:
+        """Check if OTel has been initialized."""
+        return self._initialized
+
+
+def is_otel_enabled() -> bool:
+    """Check if OTel is enabled via environment variable.
+
+    Uses standard OTEL_SDK_DISABLED env var (inverted logic).
+    """
+    disabled = os.getenv("OTEL_SDK_DISABLED", "false").lower() in ("true", "1", "yes")
+    return not disabled
+
 
 def init_otel(service_name: Optional[str] = None) -> bool:
     """Initialize OpenTelemetry with standard OTEL_* env vars.
@@ -65,56 +145,44 @@ def init_otel(service_name: Optional[str] = None) -> bool:
     Should be called once at process startup. Idempotent - safe to call multiple times.
 
     Args:
-        service_name: Default service name if OTEL_SERVICE_NAME not set
+        service_name: Default service name if OTEL_SERVICE_NAME not set (for backward compat)
 
     Returns:
         True if OTel was initialized, False if disabled or already initialized
     """
-    global _initialized
-    if _initialized:
+    # Check if OTel is disabled first
+    if not is_otel_enabled():
+        logger.debug("OpenTelemetry disabled (OTEL_SDK_DISABLED=true or not configured)")
         return False
 
-    config = OtelConfig.from_env(service_name or "kaos")
-    if not config.enabled:
-        logger.debug("OpenTelemetry disabled (OTEL_ENABLED != true)")
-        _initialized = True
+    # Try to load config from env vars
+    try:
+        # If service_name provided and OTEL_SERVICE_NAME not set, use it as fallback
+        if service_name and not os.getenv("OTEL_SERVICE_NAME"):
+            os.environ["OTEL_SERVICE_NAME"] = service_name
+
+        # Require endpoint and service_name when enabled
+        if not os.getenv("OTEL_SERVICE_NAME") or not os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT"):
+            logger.debug(
+                "OpenTelemetry not configured: "
+                "OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT required"
+            )
+            return False
+
+        config = OtelConfig()  # type: ignore[call-arg]
+    except Exception as e:
+        logger.warning(f"OpenTelemetry config error: {e}")
         return False
 
-    # Create resource with service name
-    resource = Resource.create({SERVICE_NAME: config.service_name})
-
-    # Set up W3C Trace Context propagation
-    set_global_textmap(
-        CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
-    )
-
-    # Initialize tracing
-    tracer_provider = TracerProvider(resource=resource)
-    otlp_span_exporter = OTLPSpanExporter(endpoint=config.endpoint, insecure=True)
-    tracer_provider.add_span_processor(BatchSpanProcessor(otlp_span_exporter))
-    trace.set_tracer_provider(tracer_provider)
-
-    # Initialize metrics
-    otlp_metric_exporter = OTLPMetricExporter(endpoint=config.endpoint, insecure=True)
-    metric_reader = PeriodicExportingMetricReader(otlp_metric_exporter)
-    meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
-    metrics.set_meter_provider(meter_provider)
-
-    logger.info(f"OpenTelemetry initialized: {config.endpoint} (service: {config.service_name})")
-    _initialized = True
-    return True
-
-
-def is_otel_enabled() -> bool:
-    """Check if OTel is enabled via environment variable."""
-    return os.getenv("OTEL_ENABLED", "false").lower() in ("true", "1", "yes")
+    return _OtelSingleton().initialize(config)
 
 
 class KaosOtelManager:
     """Lightweight helper for creating spans and recording metrics.
 
     Provides convenience methods for KAOS-specific telemetry. Each server/client
-    should create one instance. The underlying providers are process-global.
+    creates one instance holding thin metadata. The underlying providers are
+    process-global via _OtelSingleton.
 
     Example:
         otel = KaosOtelManager("my-agent")
diff --git a/python/tests/test_telemetry.py b/python/tests/test_telemetry.py
index b85514e..233671c 100644
--- a/python/tests/test_telemetry.py
+++ b/python/tests/test_telemetry.py
@@ -11,66 +11,79 @@ from unittest.mock import patch
 class TestIsOtelEnabled:
     """Tests for is_otel_enabled utility."""
 
-    def test_disabled_by_default(self):
-        """Test that telemetry is disabled by default."""
+    def test_enabled_by_default(self):
+        """Test that telemetry is enabled by default (OTEL_SDK_DISABLED not set)."""
         with patch.dict(os.environ, {}, clear=True):
-            from agent.telemetry import is_otel_enabled
+            from telemetry.manager import is_otel_enabled
+
+            assert is_otel_enabled() is True
+
+    def test_disabled_with_true(self):
+        """Test disabling with OTEL_SDK_DISABLED=true."""
+        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "true"}, clear=True):
+            from telemetry.manager import is_otel_enabled
 
             assert is_otel_enabled() is False
 
-    def test_enabled_with_true(self):
-        """Test enabling with OTEL_ENABLED=true."""
-        with patch.dict(os.environ, {"OTEL_ENABLED": "true"}, clear=True):
-            from agent.telemetry import is_otel_enabled
+    def test_disabled_with_one(self):
+        """Test disabling with OTEL_SDK_DISABLED=1."""
+        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "1"}, clear=True):
+            from telemetry.manager import is_otel_enabled
 
-            assert is_otel_enabled() is True
+            assert is_otel_enabled() is False
 
-    def test_enabled_with_one(self):
-        """Test enabling with OTEL_ENABLED=1."""
-        with patch.dict(os.environ, {"OTEL_ENABLED": "1"}, clear=True):
-            from agent.telemetry import is_otel_enabled
+    def test_enabled_with_false(self):
+        """Test explicitly enabled with OTEL_SDK_DISABLED=false."""
+        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "false"}, clear=True):
+            from telemetry.manager import is_otel_enabled
 
             assert is_otel_enabled() is True
 
 
 class TestOtelConfig:
-    """Tests for OtelConfig dataclass."""
+    """Tests for OtelConfig pydantic BaseSettings."""
 
-    def test_from_env_default_values(self):
-        """Test default configuration values."""
+    def test_config_requires_service_name_and_endpoint(self):
+        """Test that config requires OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT."""
         with patch.dict(os.environ, {}, clear=True):
-            from agent.telemetry.manager import OtelConfig
+            from telemetry.manager import OtelConfig
+            from pydantic import ValidationError
 
-            config = OtelConfig.from_env()
-            assert config.enabled is False
-            assert config.service_name == "kaos"
-            assert config.endpoint == "http://localhost:4317"
+            with pytest.raises(ValidationError):
+                OtelConfig()  # type: ignore[call-arg]
 
-    def test_from_env_custom_values(self):
+    def test_config_with_required_values(self):
         """Test configuration from environment variables."""
         with patch.dict(
             os.environ,
             {
-                "OTEL_ENABLED": "true",
                 "OTEL_SERVICE_NAME": "test-agent",
                 "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
             },
             clear=True,
         ):
-            from agent.telemetry.manager import OtelConfig
+            from telemetry.manager import OtelConfig
 
-            config = OtelConfig.from_env()
+            config = OtelConfig()  # type: ignore[call-arg]
+            assert config.otel_service_name == "test-agent"
+            assert config.otel_exporter_otlp_endpoint == "http://collector:4317"
             assert config.enabled is True
-            assert config.service_name == "test-agent"
-            assert config.endpoint == "http://collector:4317"
 
-    def test_from_env_with_default_service_name(self):
-        """Test from_env with custom default service name."""
-        with patch.dict(os.environ, {}, clear=True):
-            from agent.telemetry.manager import OtelConfig
+    def test_config_disabled_with_sdk_disabled(self):
+        """Test config.enabled is False when OTEL_SDK_DISABLED=true."""
+        with patch.dict(
+            os.environ,
+            {
+                "OTEL_SDK_DISABLED": "true",
+                "OTEL_SERVICE_NAME": "test-agent",
+                "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
+            },
+            clear=True,
+        ):
+            from telemetry.manager import OtelConfig
 
-            config = OtelConfig.from_env(default_service_name="my-agent")
-            assert config.service_name == "my-agent"
+            config = OtelConfig()  # type: ignore[call-arg]
+            assert config.enabled is False
 
 
 class TestKaosOtelManager:
@@ -78,30 +91,28 @@ class TestKaosOtelManager:
 
     def test_manager_creation(self):
         """Test creating a KaosOtelManager."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
         assert manager.service_name == "test-agent"
 
     def test_tracer_available(self):
         """Test getting a tracer from manager."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        # Private attribute _tracer is used internally
         assert manager._tracer is not None
 
     def test_meter_available(self):
         """Test getting a meter from manager."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        # Private attribute _meter is used internally
         assert manager._meter is not None
 
     def test_span_context_manager(self):
         """Test span context manager."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
         with manager.span("test-operation") as span:
@@ -109,39 +120,42 @@ class TestKaosOtelManager:
 
     def test_record_request(self):
         """Test record_request doesn't raise errors."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        # Should not raise - no stream parameter in new simplified API
         manager.record_request(100.0, success=True)
 
+    def test_record_request_with_failure(self):
+        """Test record_request with success=False."""
+        from telemetry.manager import KaosOtelManager
+
+        manager = KaosOtelManager("test-agent")
+        manager.record_request(100.0, success=False)
+
     def test_record_model_call(self):
         """Test record_model_call doesn't raise errors."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        # Should not raise
         manager.record_model_call("gpt-4", 500.0, success=True)
 
     def test_record_tool_call(self):
         """Test record_tool_call doesn't raise errors."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        # Should not raise
         manager.record_tool_call("calculator", 50.0, success=True)
 
     def test_record_delegation(self):
         """Test record_delegation doesn't raise errors."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        # Should not raise
         manager.record_delegation("worker-1", 200.0, success=True)
 
     def test_model_span_context_manager(self):
         """Test model_span context manager."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
         with manager.model_span("gpt-4") as span:
@@ -149,7 +163,7 @@ class TestKaosOtelManager:
 
     def test_tool_span_context_manager(self):
         """Test tool_span context manager."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
         with manager.tool_span("calculator") as span:
@@ -157,7 +171,7 @@ class TestKaosOtelManager:
 
     def test_delegation_span_context_manager(self):
         """Test delegation_span context manager."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
         with manager.delegation_span("worker-1") as span:
@@ -169,16 +183,15 @@ class TestContextPropagation:
 
     def test_inject_context(self):
         """Test context injection into headers."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         carrier: dict = {}
         result = KaosOtelManager.inject_context(carrier)
-        # May or may not have traceparent depending on active span
         assert isinstance(result, dict)
 
     def test_extract_context(self):
         """Test context extraction from headers."""
-        from agent.telemetry import KaosOtelManager
+        from telemetry.manager import KaosOtelManager
 
         carrier: dict = {}
         context = KaosOtelManager.extract_context(carrier)
@@ -190,7 +203,7 @@ class TestTimedContextManager:
 
     def test_timed_operation(self):
         """Test timed context manager tracks duration."""
-        from agent.telemetry import timed
+        from telemetry.manager import timed
 
         with timed() as result:
             time.sleep(0.01)
@@ -202,20 +215,20 @@ class TestTimedContextManager:
 class TestMCPServerTelemetrySimplified:
     """Tests for MCPServer simplified telemetry settings."""
 
-    def test_otel_disabled_by_default(self):
-        """Test that OTel is disabled by default for MCPServer."""
+    def test_otel_enabled_by_default(self):
+        """Test that OTel is enabled by default (OTEL_SDK_DISABLED not set)."""
         with patch.dict(os.environ, {}, clear=True):
             from mcptools.server import MCPServer, MCPServerSettings
 
             settings = MCPServerSettings()
             server = MCPServer(settings)
-            assert server._otel_enabled is False
+            assert server._otel_enabled is True
 
-    def test_otel_enabled_from_env(self):
-        """Test that OTel can be enabled via OTEL_ENABLED env var."""
-        with patch.dict(os.environ, {"OTEL_ENABLED": "true"}, clear=True):
+    def test_otel_disabled_from_env(self):
+        """Test that OTel can be disabled via OTEL_SDK_DISABLED env var."""
+        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "true"}, clear=True):
             from mcptools.server import MCPServer, MCPServerSettings
 
             settings = MCPServerSettings()
             server = MCPServer(settings)
-            assert server._otel_enabled is True
+            assert server._otel_enabled is False
```

---

### Commit: 9fd6c7d
**Subject:** fix(docker): add telemetry module to Dockerfile COPY
**Date:** 2026-01-25 17:41:59 +0100
**Author:** Alejandro Saucedo



```diff
 python/Dockerfile | 1 +
 1 file changed, 1 insertion(+)
```

#### Detailed Changes

```diff
diff --git a/python/Dockerfile b/python/Dockerfile
index 35582a3..eabc73a 100644
--- a/python/Dockerfile
+++ b/python/Dockerfile
@@ -19,6 +19,7 @@ RUN --mount=type=cache,target=/root/.cache/uv \
 COPY agent/ agent/
 COPY mcptools/ mcptools/
 COPY modelapi/ modelapi/
+COPY telemetry/ telemetry/
 
 # Create non-root user
 RUN useradd -m -u 65532 agentic && chown -R agentic:agentic /app
```

---

### Commit: f90e18f
**Subject:** refactor(telemetry): implement inline span API with contextvars
**Date:** 2026-01-25 19:47:02 +0100
**Author:** Alejandro Saucedo

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


```diff
 python/agent/client.py         | 272 ++++++++++++++++-----------
 python/telemetry/manager.py    | 408 +++++++++++++++++++++++------------------
 python/tests/test_telemetry.py | 149 ++++++---------
 3 files changed, 449 insertions(+), 380 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/client.py b/python/agent/client.py
index 7a0bd8b..b5cab6b 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -22,7 +22,14 @@ from dataclasses import dataclass
 from modelapi.client import ModelAPI
 from agent.memory import LocalMemory, NullMemory
 from mcptools.client import MCPClient
-from telemetry.manager import KaosOtelManager, timed
+from telemetry.manager import (
+    KaosOtelManager,
+    ATTR_SESSION_ID,
+    ATTR_MODEL_NAME,
+    ATTR_TOOL_NAME,
+    ATTR_DELEGATION_TARGET,
+)
+from opentelemetry.trace import SpanKind
 
 logger = logging.getLogger(__name__)
 
@@ -316,75 +323,75 @@ class Agent:
             For testing, set DEBUG_MOCK_RESPONSES env var to a JSON array of responses
             that will be used instead of calling the model API.
         """
-        # Use the KaosOtelManager for simpler telemetry
-        request_success = True
-        with timed() as timing:
-            # Get or create session
-            if session_id:
-                session_id = await self.memory.get_or_create_session(session_id, "agent", "user")
+        # Get or create session
+        if session_id:
+            session_id = await self.memory.get_or_create_session(session_id, "agent", "user")
+        else:
+            session_id = await self.memory.create_session("agent", "user")
+
+        logger.debug(f"Processing message for session {session_id}, streaming={stream}")
+
+        # Start agentic loop span (INTERNAL - FastAPI auto-instruments SERVER span)
+        span_attrs = {
+            "agent.max_steps": self.max_steps,
+            "stream": stream,
+            ATTR_SESSION_ID: session_id,
+        }
+        self._otel.span_begin(
+            "agent.agentic_loop",
+            attrs=span_attrs,
+            metric_kind="request",
+        )
+        try:
+            # Extract user-provided system prompt (if any) from message array
+            user_system_prompt: Optional[str] = None
+            if isinstance(message, list):
+                for msg in message:
+                    if msg.get("role") == "system":
+                        user_system_prompt = msg.get("content", "")
+                        break
+
+            # Build enhanced system prompt with tools/agents info
+            system_prompt = await self._build_system_prompt(user_system_prompt)
+            messages = [{"role": "system", "content": system_prompt}]
+
+            # Handle both string and array input formats
+            if isinstance(message, str):
+                user_event = self.memory.create_event("user_message", message)
+                await self.memory.add_event(session_id, user_event)
+                messages.append({"role": "user", "content": message})
             else:
-                session_id = await self.memory.create_session("agent", "user")
-
-            logger.debug(f"Processing message for session {session_id}, streaming={stream}")
-
-            # Use INTERNAL span (FastAPI already creates the SERVER span via auto-instrumentation)
-            span_attrs: Dict[str, Any] = {"agent.max_steps": self.max_steps, "stream": stream}
-            with self._otel.span(
-                "agent.agentic_loop",
-                session_id=session_id,
-                **span_attrs,
-            ) as span:
-                # Extract user-provided system prompt (if any) from message array
-                user_system_prompt: Optional[str] = None
-                if isinstance(message, list):
-                    for msg in message:
-                        if msg.get("role") == "system":
-                            user_system_prompt = msg.get("content", "")
-                            break
-
-                # Build enhanced system prompt with tools/agents info
-                system_prompt = await self._build_system_prompt(user_system_prompt)
-                messages = [{"role": "system", "content": system_prompt}]
-
-                # Handle both string and array input formats
-                if isinstance(message, str):
-                    user_event = self.memory.create_event("user_message", message)
-                    await self.memory.add_event(session_id, user_event)
-                    messages.append({"role": "user", "content": message})
-                else:
-                    for msg in message:
-                        role = msg.get("role", "user")
-                        content = msg.get("content", "")
-                        if role == "system":
-                            continue  # Already captured above
-
-                        if role == "task-delegation":
-                            delegation_event = self.memory.create_event(
-                                "task_delegation_received", content
-                            )
-                            await self.memory.add_event(session_id, delegation_event)
-                            messages.append({"role": "user", "content": content})
-                        else:
-                            messages.append({"role": role, "content": content})
-                            if role == "user":
-                                user_event = self.memory.create_event("user_message", content)
-                                await self.memory.add_event(session_id, user_event)
-
-                try:
-                    # Agentic loop - iterate up to max_steps
-                    async for chunk in self._agentic_loop(messages, session_id, stream):
-                        yield chunk
-
-                except Exception as e:
-                    request_success = False
-                    error_msg = f"Error processing message: {str(e)}"
-                    logger.error(error_msg)
-                    error_event = self.memory.create_event("error", error_msg)
-                    await self.memory.add_event(session_id, error_event)
-                    yield f"Sorry, I encountered an error: {str(e)}"
-
-        # Record request metrics after completion
-        self._otel.record_request(timing["duration_ms"], success=request_success)
+                for msg in message:
+                    role = msg.get("role", "user")
+                    content = msg.get("content", "")
+                    if role == "system":
+                        continue  # Already captured above
+
+                    if role == "task-delegation":
+                        delegation_event = self.memory.create_event(
+                            "task_delegation_received", content
+                        )
+                        await self.memory.add_event(session_id, delegation_event)
+                        messages.append({"role": "user", "content": content})
+                    else:
+                        messages.append({"role": role, "content": content})
+                        if role == "user":
+                            user_event = self.memory.create_event("user_message", content)
+                            await self.memory.add_event(session_id, user_event)
+
+            # Agentic loop - iterate up to max_steps
+            async for chunk in self._agentic_loop(messages, session_id, stream):
+                yield chunk
+
+        except Exception as e:
+            self._otel.span_failure(e)
+            error_msg = f"Error processing message: {str(e)}"
+            logger.error(error_msg)
+            error_event = self.memory.create_event("error", error_msg)
+            await self.memory.add_event(session_id, error_event)
+            yield f"Sorry, I encountered an error: {str(e)}"
+        finally:
+            self._otel.span_success()
 
     async def _agentic_loop(
         self,
@@ -396,21 +403,13 @@ class Agent:
         for step in range(self.max_steps):
             logger.debug(f"Agentic loop step {step + 1}/{self.max_steps}")
 
-            # Create span for this step
-            step_attrs: Dict[str, Any] = {"step": step + 1, "max_steps": self.max_steps}
-            with self._otel.span(f"agent.step.{step + 1}", **step_attrs):
-                # Get model response with tracing
-                with timed() as model_timing:
-                    with self._otel.model_span(
-                        self.model_api.model if self.model_api else "unknown"
-                    ):
-                        content = cast(
-                            str, await self.model_api.process_message(messages, stream=False)
-                        )
-                self._otel.record_model_call(
-                    self.model_api.model if self.model_api else "unknown",
-                    model_timing["duration_ms"],
-                )
+            # Start step span
+            step_attrs = {"step": step + 1, "max_steps": self.max_steps}
+            self._otel.span_begin(f"agent.step.{step + 1}", attrs=step_attrs)
+            try:
+                # Get model response
+                model_name = self.model_api.model if self.model_api else "unknown"
+                content = await self._call_model(messages, model_name)
 
                 # Check for tool call
                 tool_call = self._parse_block(content, "tool_call")
@@ -424,21 +423,8 @@ class Agent:
                         if not tool_name:
                             raise ValueError("Tool name not specified")
 
-                        # Execute tool with tracing
-                        with timed() as tool_timing:
-                            with self._otel.tool_span(tool_name):
-                                tool_result = None
-                                for mcp_client in self.mcp_clients:
-                                    if tool_name in mcp_client._tools:
-                                        tool_result = await mcp_client.call_tool(
-                                            tool_name, tool_args
-                                        )
-                                        break
-
-                                if tool_result is None:
-                                    raise ValueError(f"Tool '{tool_name}' not found")
-
-                        self._otel.record_tool_call(tool_name, tool_timing["duration_ms"])
+                        # Execute tool
+                        tool_result = await self._execute_tool(tool_name, tool_args)
 
                         result_event = self.memory.create_event(
                             "tool_result", {"tool": tool_name, "result": tool_result}
@@ -475,14 +461,10 @@ class Agent:
                     try:
                         context_messages = [m for m in messages if m.get("role") != "system"]
 
-                        # Delegate with tracing
-                        with timed() as delegation_timing:
-                            with self._otel.delegation_span(agent_name):
-                                delegation_result = await self.delegate_to_sub_agent(
-                                    agent_name, task, context_messages, session_id
-                                )
-
-                        self._otel.record_delegation(agent_name, delegation_timing["duration_ms"])
+                        # Delegate to sub-agent
+                        delegation_result = await self._execute_delegation(
+                            agent_name, task, context_messages, session_id
+                        )
 
                         messages.append({"role": "assistant", "content": content})
                         messages.append(
@@ -506,11 +488,87 @@ class Agent:
                     yield content
                 return
 
+            except Exception as e:
+                self._otel.span_failure(e)
+                raise
+            finally:
+                self._otel.span_success()
+
         # Max steps reached
         max_steps_msg = f"Reached maximum reasoning steps ({self.max_steps})"
         logger.warning(max_steps_msg)
         yield max_steps_msg
 
+    async def _call_model(self, messages: List[Dict[str, str]], model_name: str) -> str:
+        """Call the model API with tracing."""
+        self._otel.span_begin(
+            "model.inference",
+            kind=SpanKind.CLIENT,
+            attrs={ATTR_MODEL_NAME: model_name},
+            metric_kind="model",
+            metric_attrs={"model": model_name},
+        )
+        try:
+            content = cast(str, await self.model_api.process_message(messages, stream=False))
+            return content
+        except Exception as e:
+            self._otel.span_failure(e)
+            raise
+        finally:
+            self._otel.span_success()
+
+    async def _execute_tool(self, tool_name: str, tool_args: Dict[str, Any]) -> Any:
+        """Execute a tool with tracing."""
+        self._otel.span_begin(
+            f"tool.{tool_name}",
+            kind=SpanKind.CLIENT,
+            attrs={ATTR_TOOL_NAME: tool_name},
+            metric_kind="tool",
+            metric_attrs={"tool": tool_name},
+        )
+        try:
+            tool_result = None
+            for mcp_client in self.mcp_clients:
+                if tool_name in mcp_client._tools:
+                    tool_result = await mcp_client.call_tool(tool_name, tool_args)
+                    break
+
+            if tool_result is None:
+                raise ValueError(f"Tool '{tool_name}' not found")
+
+            return tool_result
+        except Exception as e:
+            self._otel.span_failure(e)
+            raise
+        finally:
+            self._otel.span_success()
+
+    async def _execute_delegation(
+        self,
+        agent_name: str,
+        task: str,
+        context_messages: List[Dict[str, str]],
+        session_id: str,
+    ) -> str:
+        """Execute delegation to a sub-agent with tracing."""
+        self._otel.span_begin(
+            f"delegate.{agent_name}",
+            kind=SpanKind.CLIENT,
+            attrs={ATTR_DELEGATION_TARGET: agent_name},
+            metric_kind="delegation",
+            metric_attrs={"target": agent_name},
+        )
+        try:
+            result = await self.delegate_to_sub_agent(
+                agent_name, task, context_messages, session_id
+            )
+            return result
+        except Exception as e:
+            self._otel.span_failure(e)
+            raise
+        finally:
+            self._otel.span_success()
+
     async def delegate_to_sub_agent(
         self,
         agent_name: str,
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 1e13cf7..bc51e81 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -6,19 +6,21 @@ OTEL_* environment variables. Uses OTEL_SDK_DISABLED (standard OTel env var) to
 control whether telemetry is enabled.
 
 Key design:
-- Process-global SDK initialization via _OtelSingleton (providers, propagators)
-- Lightweight KaosOtelManager instances for per-service span/metric creation
+- Process-global SDK initialization via module-level _initialized flag
+- Inline span management via span_begin/span_success/span_failure (no context managers)
+- Async-safe span stack via contextvars for nesting support
 - OtelConfig uses pydantic BaseSettings with OTEL-compliant env var names
 """
 
 import logging
 import os
 import time
-from contextlib import contextmanager
-from typing import Any, Dict, Iterator, Optional
+from contextvars import ContextVar
+from dataclasses import dataclass, field
+from typing import Any, Dict, List, Optional
 
 from pydantic_settings import BaseSettings, SettingsConfigDict
-from opentelemetry import trace, metrics
+from opentelemetry import trace, metrics, context as otel_context
 from opentelemetry.sdk.trace import TracerProvider
 from opentelemetry.sdk.trace.export import BatchSpanProcessor
 from opentelemetry.sdk.metrics import MeterProvider
@@ -42,6 +44,25 @@ ATTR_MODEL_NAME = "gen_ai.request.model"
 ATTR_TOOL_NAME = "tool.name"
 ATTR_DELEGATION_TARGET = "agent.delegation.target"
 
+# Process-global initialization state
+_initialized: bool = False
+
+
+@dataclass
+class SpanState:
+    """State for an active span on the stack."""
+
+    span: Span
+    token: object  # Context token for detaching
+    start_time: float
+    metric_kind: Optional[str] = None  # "request", "model", "tool", "delegation"
+    metric_attrs: Dict[str, Any] = field(default_factory=dict)
+    ended: bool = False
+
+
+# Async-safe span stack per context (supports nesting)
+_span_stack: ContextVar[List[SpanState]] = ContextVar("kaos_span_stack", default=[])
+
 
 class OtelConfig(BaseSettings):
     """OpenTelemetry configuration from standard OTEL_* environment variables.
@@ -68,75 +89,12 @@ class OtelConfig(BaseSettings):
         return not self.otel_sdk_disabled
 
 
-class _OtelSingleton:
-    """Private singleton holding process-global OTel state.
-
-    SDK providers are process-global and should only be initialized once.
-    This class ensures idempotent initialization.
-    """
-
-    _instance: Optional["_OtelSingleton"] = None
-    _initialized: bool = False
-
-    def __new__(cls) -> "_OtelSingleton":
-        if cls._instance is None:
-            cls._instance = super().__new__(cls)
-        return cls._instance
-
-    def initialize(self, config: OtelConfig) -> bool:
-        """Initialize OTel SDK with given config. Idempotent.
-
-        Returns True if initialization happened, False if already initialized or disabled.
-        """
-        if self._initialized:
-            return False
-
-        if not config.enabled:
-            logger.debug("OpenTelemetry disabled (OTEL_SDK_DISABLED=true)")
-            self._initialized = True
-            return False
-
-        # Create resource with service name
-        resource = Resource.create({SERVICE_NAME: config.otel_service_name})
-
-        # Set up W3C Trace Context propagation (standard)
-        set_global_textmap(
-            CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
-        )
-
-        # Initialize tracing - let SDK use OTEL_EXPORTER_OTLP_* env vars for advanced config
-        tracer_provider = TracerProvider(resource=resource)
-        # Only set endpoint explicitly, let SDK handle insecure/headers via env vars
-        otlp_span_exporter = OTLPSpanExporter(endpoint=config.otel_exporter_otlp_endpoint)
-        tracer_provider.add_span_processor(BatchSpanProcessor(otlp_span_exporter))
-        trace.set_tracer_provider(tracer_provider)
-
-        # Initialize metrics
-        otlp_metric_exporter = OTLPMetricExporter(endpoint=config.otel_exporter_otlp_endpoint)
-        metric_reader = PeriodicExportingMetricReader(otlp_metric_exporter)
-        meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
-        metrics.set_meter_provider(meter_provider)
-
-        logger.info(
-            f"OpenTelemetry initialized: {config.otel_exporter_otlp_endpoint} "
-            f"(service: {config.otel_service_name})"
-        )
-        self._initialized = True
-        return True
-
-    @property
-    def is_initialized(self) -> bool:
-        """Check if OTel has been initialized."""
-        return self._initialized
-
-
 def is_otel_enabled() -> bool:
-    """Check if OTel is enabled via environment variable.
+    """Check if OTel is initialized and enabled.
 
-    Uses standard OTEL_SDK_DISABLED env var (inverted logic).
+    Returns True only if init_otel() was successfully called and OTel is active.
     """
-    disabled = os.getenv("OTEL_SDK_DISABLED", "false").lower() in ("true", "1", "yes")
-    return not disabled
+    return _initialized
 
 
 def init_otel(service_name: Optional[str] = None) -> bool:
@@ -150,9 +108,15 @@ def init_otel(service_name: Optional[str] = None) -> bool:
     Returns:
         True if OTel was initialized, False if disabled or already initialized
     """
-    # Check if OTel is disabled first
-    if not is_otel_enabled():
-        logger.debug("OpenTelemetry disabled (OTEL_SDK_DISABLED=true or not configured)")
+    global _initialized
+
+    if _initialized:
+        return False
+
+    # Check if OTel is disabled via standard env var
+    disabled = os.getenv("OTEL_SDK_DISABLED", "false").lower() in ("true", "1", "yes")
+    if disabled:
+        logger.debug("OpenTelemetry disabled (OTEL_SDK_DISABLED=true)")
         return False
 
     # Try to load config from env vars
@@ -174,21 +138,51 @@ def init_otel(service_name: Optional[str] = None) -> bool:
         logger.warning(f"OpenTelemetry config error: {e}")
         return False
 
-    return _OtelSingleton().initialize(config)
+    # Create resource with service name
+    resource = Resource.create({SERVICE_NAME: config.otel_service_name})
+
+    # Set up W3C Trace Context propagation (standard)
+    set_global_textmap(
+        CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
+    )
+
+    # Initialize tracing - let SDK use OTEL_EXPORTER_OTLP_* env vars for advanced config
+    tracer_provider = TracerProvider(resource=resource)
+    otlp_span_exporter = OTLPSpanExporter(endpoint=config.otel_exporter_otlp_endpoint)
+    tracer_provider.add_span_processor(BatchSpanProcessor(otlp_span_exporter))
+    trace.set_tracer_provider(tracer_provider)
+
+    # Initialize metrics
+    otlp_metric_exporter = OTLPMetricExporter(endpoint=config.otel_exporter_otlp_endpoint)
+    metric_reader = PeriodicExportingMetricReader(otlp_metric_exporter)
+    meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
+    metrics.set_meter_provider(meter_provider)
+
+    logger.info(
+        f"OpenTelemetry initialized: {config.otel_exporter_otlp_endpoint} "
+        f"(service: {config.otel_service_name})"
+    )
+    _initialized = True
+    return True
 
 
 class KaosOtelManager:
     """Lightweight helper for creating spans and recording metrics.
 
-    Provides convenience methods for KAOS-specific telemetry. Each server/client
-    creates one instance holding thin metadata. The underlying providers are
-    process-global via _OtelSingleton.
+    Uses inline span management via span_begin/span_success/span_failure instead
+    of context managers. Timing is handled internally via contextvars.
 
     Example:
         otel = KaosOtelManager("my-agent")
-        with otel.span("process_request", session_id="abc123") as span:
+        otel.span_begin("process_request", attrs={"session.id": "abc123"})
+        try:
             # do work
-            span.set_attribute("custom", "value")
+            pass
+        except Exception as e:
+            otel.span_failure(e)
+            raise
+        finally:
+            otel.span_success()
     """
 
     def __init__(self, service_name: str):
@@ -241,98 +235,172 @@ class KaosOtelManager:
             "kaos.delegation.duration", description="Delegation duration", unit="ms"
         )
 
-    @contextmanager
-    def span(
+    def _get_stack(self) -> List[SpanState]:
+        """Get or create the span stack for current async context."""
+        try:
+            return _span_stack.get()
+        except LookupError:
+            stack: List[SpanState] = []
+            _span_stack.set(stack)
+            return stack
+
+    def span_begin(
         self,
         name: str,
+        *,
         kind: SpanKind = SpanKind.INTERNAL,
-        session_id: Optional[str] = None,
-        **attributes: Any,
-    ) -> Iterator[Span]:
-        """Create a span with automatic end and status handling.
+        attrs: Optional[Dict[str, Any]] = None,
+        metric_kind: Optional[str] = None,
+        metric_attrs: Optional[Dict[str, Any]] = None,
+    ) -> None:
+        """Begin a span. Must be paired with span_success() or span_failure().
 
         Args:
             name: Span name
             kind: Span kind (INTERNAL, CLIENT, SERVER)
-            session_id: Optional session ID to attach
-            **attributes: Additional span attributes
-
-        Yields:
-            The active span
+            attrs: Span attributes
+            metric_kind: Type of metric to record ("request", "model", "tool", "delegation")
+            metric_attrs: Additional attributes for metric recording
         """
-        attrs = {ATTR_AGENT_NAME: self.service_name}
-        if session_id:
-            attrs[ATTR_SESSION_ID] = session_id
-        attrs.update({k: v for k, v in attributes.items() if v is not None})
-
-        with self._tracer.start_as_current_span(name, kind=kind, attributes=attrs) as span:
-            try:
-                yield span
-                span.set_status(Status(StatusCode.OK))
-            except Exception as e:
-                span.set_status(Status(StatusCode.ERROR, str(e)))
-                span.record_exception(e)
-                raise
-
-    @contextmanager
-    def model_span(self, model_name: str) -> Iterator[Span]:
-        """Create a span for model API calls."""
-        with self.span("model.inference", SpanKind.CLIENT, **{ATTR_MODEL_NAME: model_name}) as s:
-            yield s
-
-    @contextmanager
-    def tool_span(self, tool_name: str) -> Iterator[Span]:
-        """Create a span for tool execution."""
-        with self.span(f"tool.{tool_name}", SpanKind.CLIENT, **{ATTR_TOOL_NAME: tool_name}) as s:
-            yield s
-
-    @contextmanager
-    def delegation_span(self, target_agent: str) -> Iterator[Span]:
-        """Create a span for A2A delegation."""
-        with self.span(
-            f"delegate.{target_agent}", SpanKind.CLIENT, **{ATTR_DELEGATION_TARGET: target_agent}
-        ) as s:
-            yield s
-
-    def record_request(self, duration_ms: float, success: bool = True) -> None:
-        """Record request metrics."""
-        self._ensure_metrics()
-        labels = {"agent.name": self.service_name, "success": str(success).lower()}
-        if self._request_counter:
-            self._request_counter.add(1, labels)
-        if self._request_duration:
-            self._request_duration.record(duration_ms, labels)
-
-    def record_model_call(self, model: str, duration_ms: float, success: bool = True) -> None:
-        """Record model API call metrics."""
-        self._ensure_metrics()
-        labels = {"agent.name": self.service_name, "model": model, "success": str(success).lower()}
-        if self._model_counter:
-            self._model_counter.add(1, labels)
-        if self._model_duration:
-            self._model_duration.record(duration_ms, labels)
-
-    def record_tool_call(self, tool: str, duration_ms: float, success: bool = True) -> None:
-        """Record tool call metrics."""
-        self._ensure_metrics()
-        labels = {"agent.name": self.service_name, "tool": tool, "success": str(success).lower()}
-        if self._tool_counter:
-            self._tool_counter.add(1, labels)
-        if self._tool_duration:
-            self._tool_duration.record(duration_ms, labels)
-
-    def record_delegation(self, target: str, duration_ms: float, success: bool = True) -> None:
-        """Record delegation metrics."""
+        if not _initialized:
+            return
+
+        # Build attributes
+        span_attrs = {ATTR_AGENT_NAME: self.service_name}
+        if attrs:
+            span_attrs.update({k: v for k, v in attrs.items() if v is not None})
+
+        # Start span and make it current
+        span = self._tracer.start_span(name, kind=kind, attributes=span_attrs)
+        token = otel_context.attach(trace.set_span_in_context(span))
+
+        # Push state onto stack
+        state = SpanState(
+            span=span,
+            token=token,
+            start_time=time.perf_counter(),
+            metric_kind=metric_kind,
+            metric_attrs=metric_attrs or {},
+        )
+        stack = self._get_stack()
+        stack.append(state)
+
+    def span_success(self) -> None:
+        """End the current span with OK status. No-op if already ended or OTel disabled."""
+        if not _initialized:
+            return
+
+        stack = self._get_stack()
+        if not stack:
+            return
+
+        state = stack[-1]
+        if state.ended:
+            return
+
+        # Mark ended and calculate duration
+        state.ended = True
+        duration_ms = (time.perf_counter() - state.start_time) * 1000
+
+        # Set status and end span
+        state.span.set_status(Status(StatusCode.OK))
+        state.span.end()
+
+        # Detach context
+        otel_context.detach(state.token)
+
+        # Record metrics
+        self._record_metric(state.metric_kind, state.metric_attrs, duration_ms, success=True)
+
+        # Pop from stack
+        stack.pop()
+
+    def span_failure(self, exc: Exception) -> None:
+        """End the current span with ERROR status. Records the exception."""
+        if not _initialized:
+            return
+
+        stack = self._get_stack()
+        if not stack:
+            return
+
+        state = stack[-1]
+        if state.ended:
+            return
+
+        # Mark ended and calculate duration
+        state.ended = True
+        duration_ms = (time.perf_counter() - state.start_time) * 1000
+
+        # Set status, record exception, and end span
+        state.span.set_status(Status(StatusCode.ERROR, str(exc)))
+        state.span.record_exception(exc)
+        state.span.end()
+
+        # Detach context
+        otel_context.detach(state.token)
+
+        # Record metrics
+        self._record_metric(state.metric_kind, state.metric_attrs, duration_ms, success=False)
+
+        # Pop from stack
+        stack.pop()
+
+    def _record_metric(
+        self,
+        metric_kind: Optional[str],
+        metric_attrs: Dict[str, Any],
+        duration_ms: float,
+        success: bool,
+    ) -> None:
+        """Record metrics based on metric_kind."""
+        if not metric_kind:
+            return
+
         self._ensure_metrics()
-        labels = {
-            "agent.name": self.service_name,
-            "target": target,
-            "success": str(success).lower(),
-        }
-        if self._delegation_counter:
-            self._delegation_counter.add(1, labels)
-        if self._delegation_duration:
-            self._delegation_duration.record(duration_ms, labels)
+
+        if metric_kind == "request":
+            labels = {"agent.name": self.service_name, "success": str(success).lower()}
+            if self._request_counter:
+                self._request_counter.add(1, labels)
+            if self._request_duration:
+                self._request_duration.record(duration_ms, labels)
+
+        elif metric_kind == "model":
+            model = metric_attrs.get("model", "unknown")
+            labels = {
+                "agent.name": self.service_name,
+                "model": model,
+                "success": str(success).lower(),
+            }
+            if self._model_counter:
+                self._model_counter.add(1, labels)
+            if self._model_duration:
+                self._model_duration.record(duration_ms, labels)
+
+        elif metric_kind == "tool":
+            tool = metric_attrs.get("tool", "unknown")
+            labels = {
+                "agent.name": self.service_name,
+                "tool": tool,
+                "success": str(success).lower(),
+            }
+            if self._tool_counter:
+                self._tool_counter.add(1, labels)
+            if self._tool_duration:
+                self._tool_duration.record(duration_ms, labels)
+
+        elif metric_kind == "delegation":
+            target = metric_attrs.get("target", "unknown")
+            labels = {
+                "agent.name": self.service_name,
+                "target": target,
+                "success": str(success).lower(),
+            }
+            if self._delegation_counter:
+                self._delegation_counter.add(1, labels)
+            if self._delegation_duration:
+                self._delegation_duration.record(duration_ms, labels)
 
     @staticmethod
     def inject_context(carrier: Dict[str, str]) -> Dict[str, str]:
@@ -344,17 +412,3 @@ class KaosOtelManager:
     def extract_context(carrier: Dict[str, str]) -> Context:
         """Extract trace context from headers."""
         return extract(carrier)
-
-
-@contextmanager
-def timed() -> Iterator[Dict[str, float]]:
-    """Context manager for timing operations.
-
-    Yields a dict that will contain 'duration_ms' after the block.
-    """
-    result: Dict[str, float] = {}
-    start = time.perf_counter()
-    try:
-        yield result
-    finally:
-        result["duration_ms"] = (time.perf_counter() - start) * 1000
diff --git a/python/tests/test_telemetry.py b/python/tests/test_telemetry.py
index 233671c..81ccf42 100644
--- a/python/tests/test_telemetry.py
+++ b/python/tests/test_telemetry.py
@@ -2,42 +2,26 @@
 Tests for OpenTelemetry instrumentation.
 """
 
-import pytest
 import os
-import time
+import pytest
 from unittest.mock import patch
 
 
 class TestIsOtelEnabled:
     """Tests for is_otel_enabled utility."""
 
-    def test_enabled_by_default(self):
-        """Test that telemetry is enabled by default (OTEL_SDK_DISABLED not set)."""
-        with patch.dict(os.environ, {}, clear=True):
-            from telemetry.manager import is_otel_enabled
-
-            assert is_otel_enabled() is True
-
-    def test_disabled_with_true(self):
-        """Test disabling with OTEL_SDK_DISABLED=true."""
-        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "true"}, clear=True):
-            from telemetry.manager import is_otel_enabled
-
-            assert is_otel_enabled() is False
-
-    def test_disabled_with_one(self):
-        """Test disabling with OTEL_SDK_DISABLED=1."""
-        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "1"}, clear=True):
-            from telemetry.manager import is_otel_enabled
-
-            assert is_otel_enabled() is False
-
-    def test_enabled_with_false(self):
-        """Test explicitly enabled with OTEL_SDK_DISABLED=false."""
-        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "false"}, clear=True):
-            from telemetry.manager import is_otel_enabled
+    def test_returns_false_before_init(self):
+        """Test that is_otel_enabled returns False before initialization."""
+        # Import fresh module - is_otel_enabled checks _initialized flag, not env var
+        import telemetry.manager as tm
 
-            assert is_otel_enabled() is True
+        # Reset module state for testing
+        original = tm._initialized
+        tm._initialized = False
+        try:
+            assert tm.is_otel_enabled() is False
+        finally:
+            tm._initialized = original
 
 
 class TestOtelConfig:
@@ -110,72 +94,59 @@ class TestKaosOtelManager:
         manager = KaosOtelManager("test-agent")
         assert manager._meter is not None
 
-    def test_span_context_manager(self):
-        """Test span context manager."""
-        from telemetry.manager import KaosOtelManager
-
-        manager = KaosOtelManager("test-agent")
-        with manager.span("test-operation") as span:
-            assert span is not None
-
-    def test_record_request(self):
-        """Test record_request doesn't raise errors."""
-        from telemetry.manager import KaosOtelManager
-
-        manager = KaosOtelManager("test-agent")
-        manager.record_request(100.0, success=True)
-
-    def test_record_request_with_failure(self):
-        """Test record_request with success=False."""
-        from telemetry.manager import KaosOtelManager
-
-        manager = KaosOtelManager("test-agent")
-        manager.record_request(100.0, success=False)
-
-    def test_record_model_call(self):
-        """Test record_model_call doesn't raise errors."""
+    def test_span_begin_success_pattern(self):
+        """Test span_begin/span_success pattern (no-op when not initialized)."""
         from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        manager.record_model_call("gpt-4", 500.0, success=True)
-
-    def test_record_tool_call(self):
-        """Test record_tool_call doesn't raise errors."""
+        manager.span_begin("test-operation")
+        try:
+            pass  # do work
+        finally:
+            manager.span_success()
+
+    def test_span_begin_failure_pattern(self):
+        """Test span_begin/span_failure pattern."""
         from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        manager.record_tool_call("calculator", 50.0, success=True)
-
-    def test_record_delegation(self):
-        """Test record_delegation doesn't raise errors."""
+        manager.span_begin("test-operation")
+        try:
+            raise ValueError("test error")
+        except ValueError as e:
+            manager.span_failure(e)
+        finally:
+            manager.span_success()  # Should be no-op after failure
+
+    def test_nested_spans(self):
+        """Test nested span_begin calls."""
         from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        manager.record_delegation("worker-1", 200.0, success=True)
-
-    def test_model_span_context_manager(self):
-        """Test model_span context manager."""
+        manager.span_begin("outer")
+        try:
+            manager.span_begin("inner")
+            try:
+                pass
+            finally:
+                manager.span_success()
+        finally:
+            manager.span_success()
+
+    def test_span_with_metric_kind(self):
+        """Test span_begin with metric_kind parameter."""
         from telemetry.manager import KaosOtelManager
 
         manager = KaosOtelManager("test-agent")
-        with manager.model_span("gpt-4") as span:
-            assert span is not None
-
-    def test_tool_span_context_manager(self):
-        """Test tool_span context manager."""
-        from telemetry.manager import KaosOtelManager
-
-        manager = KaosOtelManager("test-agent")
-        with manager.tool_span("calculator") as span:
-            assert span is not None
-
-    def test_delegation_span_context_manager(self):
-        """Test delegation_span context manager."""
-        from telemetry.manager import KaosOtelManager
-
-        manager = KaosOtelManager("test-agent")
-        with manager.delegation_span("worker-1") as span:
-            assert span is not None
+        manager.span_begin(
+            "model.inference",
+            metric_kind="model",
+            metric_attrs={"model": "gpt-4"},
+        )
+        try:
+            pass
+        finally:
+            manager.span_success()
 
 
 class TestContextPropagation:
@@ -198,20 +169,6 @@ class TestContextPropagation:
         assert context is not None
 
 
-class TestTimedContextManager:
-    """Tests for timed context manager."""
-
-    def test_timed_operation(self):
-        """Test timed context manager tracks duration."""
-        from telemetry.manager import timed
-
-        with timed() as result:
-            time.sleep(0.01)
-
-        assert "duration_ms" in result
-        assert result["duration_ms"] >= 10  # At least 10ms
-
-
 class TestMCPServerTelemetrySimplified:
     """Tests for MCPServer simplified telemetry settings."""
 
```

---

### Commit: 1865e89
**Subject:** feat(helm): add global telemetry configuration
**Date:** 2026-01-26 09:28:32 +0100
**Author:** Alejandro Saucedo

- Add telemetry.enabled and telemetry.endpoint to values.yaml
- Add DEFAULT_TELEMETRY_* env vars to operator ConfigMap
- Add GetDefaultTelemetryConfig() and MergeTelemetryConfig() utilities
- Update agent and mcpserver controllers to use merged config
- Component-level config overrides global defaults
- Add unit tests for telemetry utilities
- Update documentation with global config section


```diff
 docs/operator/telemetry.md                       |  33 +++-
 operator/chart/templates/operator-configmap.yaml |   7 +
 operator/chart/values.yaml                       |   9 ++
 operator/controllers/agent_controller.go         |  11 +-
 operator/controllers/mcpserver_controller.go     |   7 +-
 operator/pkg/util/telemetry.go                   |  23 +++
 operator/pkg/util/telemetry_test.go              | 191 +++++++++++++++++++++++
 7 files changed, 273 insertions(+), 8 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index b3724d2..a84d3bb 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -10,9 +10,31 @@ When enabled, OpenTelemetry instrumentation provides:
 - **Metrics**: Counters and histograms for requests, latency, and error rates
 - **Log Correlation**: Automatic injection of trace_id and span_id into log entries
 
-## Enabling Telemetry
+## Global Telemetry Configuration
 
-Add a `telemetry` section to your Agent's or MCPServer's config:
+You can enable telemetry globally for all components via Helm values:
+
+```yaml
+# values.yaml
+telemetry:
+  enabled: true
+  endpoint: "http://otel-collector.observability.svc.cluster.local:4317"
+```
+
+Install with global telemetry enabled:
+
+```bash
+helm install kaos oci://ghcr.io/axsaucedo/kaos/chart \
+  --namespace kaos-system \
+  --set telemetry.enabled=true \
+  --set telemetry.endpoint="http://otel-collector:4317"
+```
+
+All Agents and MCPServers will have telemetry enabled by default with this configuration.
+
+## Component-Level Configuration
+
+Override global defaults or enable telemetry for specific components:
 
 ```yaml
 apiVersion: kaos.tools/v1alpha1
@@ -46,6 +68,13 @@ spec:
             return msg
 ```
 
+### Configuration Precedence
+
+Component-level configuration always overrides global defaults:
+
+1. **Component-level telemetry** (highest priority): If `spec.config.telemetry` is set on the Agent or MCPServer, it is used
+2. **Global Helm values** (default): If component-level telemetry is not set, the global `telemetry.enabled` and `telemetry.endpoint` values are used
+
 ## Configuration Fields
 
 The CRD configuration is intentionally minimal. For advanced settings, use standard `OTEL_*` environment variables via `spec.config.env`.
diff --git a/operator/chart/templates/operator-configmap.yaml b/operator/chart/templates/operator-configmap.yaml
index dca9547..26d0305 100644
--- a/operator/chart/templates/operator-configmap.yaml
+++ b/operator/chart/templates/operator-configmap.yaml
@@ -23,3 +23,10 @@ data:
   GATEWAY_DEFAULT_AGENT_TIMEOUT: {{ .Values.gateway.defaultTimeouts.agent | quote }}
   GATEWAY_DEFAULT_MODELAPI_TIMEOUT: {{ .Values.gateway.defaultTimeouts.modelAPI | quote }}
   GATEWAY_DEFAULT_MCP_TIMEOUT: {{ .Values.gateway.defaultTimeouts.mcp | quote }}
+  # Global telemetry configuration (defaults for all components)
+  {{- if .Values.telemetry.enabled }}
+  DEFAULT_TELEMETRY_ENABLED: "true"
+  DEFAULT_TELEMETRY_ENDPOINT: {{ .Values.telemetry.endpoint | quote }}
+  {{- else }}
+  DEFAULT_TELEMETRY_ENABLED: "false"
+  {{- end }}
diff --git a/operator/chart/values.yaml b/operator/chart/values.yaml
index 3f82e3b..64bf387 100644
--- a/operator/chart/values.yaml
+++ b/operator/chart/values.yaml
@@ -57,3 +57,12 @@ serviceAccount:
   automount: true
   create: true
   name: ""
+
+# Global telemetry configuration (applies to all components)
+# Component-level telemetry config overrides these defaults
+telemetry:
+  # enabled controls whether telemetry is enabled by default for all components
+  enabled: false
+  # endpoint is the OTLP endpoint URL (required when enabled)
+  # Example: "http://otel-collector.observability:4317"
+  endpoint: ""
diff --git a/operator/controllers/agent_controller.go b/operator/controllers/agent_controller.go
index 9ef7ef8..33fbeea 100644
--- a/operator/controllers/agent_controller.go
+++ b/operator/controllers/agent_controller.go
@@ -513,10 +513,15 @@ func (r *AgentReconciler) constructEnvVars(agent *kaosv1alpha1.Agent, modelapi *
 		}
 	}
 
-	// OpenTelemetry configuration
-	if agent.Spec.Config != nil && agent.Spec.Config.Telemetry != nil {
+	// OpenTelemetry configuration - merge with global defaults
+	var componentTelemetry *kaosv1alpha1.TelemetryConfig
+	if agent.Spec.Config != nil {
+		componentTelemetry = agent.Spec.Config.Telemetry
+	}
+	telemetryConfig := util.MergeTelemetryConfig(componentTelemetry)
+	if telemetryConfig != nil {
 		otelEnv := util.BuildTelemetryEnvVars(
-			agent.Spec.Config.Telemetry,
+			telemetryConfig,
 			agent.Name,
 			agent.Namespace,
 		)
diff --git a/operator/controllers/mcpserver_controller.go b/operator/controllers/mcpserver_controller.go
index d949753..3ea7314 100644
--- a/operator/controllers/mcpserver_controller.go
+++ b/operator/controllers/mcpserver_controller.go
@@ -305,10 +305,11 @@ func (r *MCPServerReconciler) constructPythonContainer(mcpserver *kaosv1alpha1.M
 		}
 	}
 
-	// OpenTelemetry configuration
-	if mcpserver.Spec.Config.Telemetry != nil {
+	// OpenTelemetry configuration - merge with global defaults
+	telemetryConfig := util.MergeTelemetryConfig(mcpserver.Spec.Config.Telemetry)
+	if telemetryConfig != nil {
 		otelEnv := util.BuildTelemetryEnvVars(
-			mcpserver.Spec.Config.Telemetry,
+			telemetryConfig,
 			mcpserver.Name,
 			mcpserver.Namespace,
 		)
diff --git a/operator/pkg/util/telemetry.go b/operator/pkg/util/telemetry.go
index b68fad1..9409c27 100644
--- a/operator/pkg/util/telemetry.go
+++ b/operator/pkg/util/telemetry.go
@@ -9,6 +9,29 @@ import (
 	kaosv1alpha1 "github.com/axsaucedo/kaos/operator/api/v1alpha1"
 )
 
+// GetDefaultTelemetryConfig returns a TelemetryConfig from global environment variables.
+// Returns nil if DEFAULT_TELEMETRY_ENABLED is not "true".
+func GetDefaultTelemetryConfig() *kaosv1alpha1.TelemetryConfig {
+	if os.Getenv("DEFAULT_TELEMETRY_ENABLED") != "true" {
+		return nil
+	}
+	return &kaosv1alpha1.TelemetryConfig{
+		Enabled:  true,
+		Endpoint: os.Getenv("DEFAULT_TELEMETRY_ENDPOINT"),
+	}
+}
+
+// MergeTelemetryConfig merges component-level telemetry config with global defaults.
+// Component-level config takes precedence over global defaults.
+func MergeTelemetryConfig(componentConfig *kaosv1alpha1.TelemetryConfig) *kaosv1alpha1.TelemetryConfig {
+	// If component has explicit config, use it
+	if componentConfig != nil {
+		return componentConfig
+	}
+	// Otherwise fall back to global defaults
+	return GetDefaultTelemetryConfig()
+}
+
 // BuildTelemetryEnvVars creates environment variables for OpenTelemetry configuration.
 // Uses standard OTEL_* env vars so the SDK auto-configures.
 // serviceName is used as OTEL_SERVICE_NAME (typically the CR name).
diff --git a/operator/pkg/util/telemetry_test.go b/operator/pkg/util/telemetry_test.go
new file mode 100644
index 0000000..03a09dd
--- /dev/null
+++ b/operator/pkg/util/telemetry_test.go
@@ -0,0 +1,191 @@
+package util
+
+import (
+	"os"
+	"testing"
+
+	kaosv1alpha1 "github.com/axsaucedo/kaos/operator/api/v1alpha1"
+)
+
+func TestGetDefaultTelemetryConfig(t *testing.T) {
+	tests := []struct {
+		name           string
+		envEnabled     string
+		envEndpoint    string
+		expectNil      bool
+		expectEnabled  bool
+		expectEndpoint string
+	}{
+		{
+			name:      "disabled when env not set",
+			expectNil: true,
+		},
+		{
+			name:       "disabled when env is false",
+			envEnabled: "false",
+			expectNil:  true,
+		},
+		{
+			name:           "enabled when env is true",
+			envEnabled:     "true",
+			envEndpoint:    "http://collector:4317",
+			expectNil:      false,
+			expectEnabled:  true,
+			expectEndpoint: "http://collector:4317",
+		},
+	}
+
+	for _, tt := range tests {
+		t.Run(tt.name, func(t *testing.T) {
+			// Clear env
+			os.Unsetenv("DEFAULT_TELEMETRY_ENABLED")
+			os.Unsetenv("DEFAULT_TELEMETRY_ENDPOINT")
+
+			// Set env for test
+			if tt.envEnabled != "" {
+				os.Setenv("DEFAULT_TELEMETRY_ENABLED", tt.envEnabled)
+			}
+			if tt.envEndpoint != "" {
+				os.Setenv("DEFAULT_TELEMETRY_ENDPOINT", tt.envEndpoint)
+			}
+
+			result := GetDefaultTelemetryConfig()
+
+			if tt.expectNil && result != nil {
+				t.Errorf("expected nil, got %+v", result)
+			}
+			if !tt.expectNil {
+				if result == nil {
+					t.Fatal("expected non-nil result")
+				}
+				if result.Enabled != tt.expectEnabled {
+					t.Errorf("expected Enabled=%v, got %v", tt.expectEnabled, result.Enabled)
+				}
+				if result.Endpoint != tt.expectEndpoint {
+					t.Errorf("expected Endpoint=%s, got %s", tt.expectEndpoint, result.Endpoint)
+				}
+			}
+		})
+	}
+}
+
+func TestMergeTelemetryConfig(t *testing.T) {
+	// Set up global defaults
+	os.Setenv("DEFAULT_TELEMETRY_ENABLED", "true")
+	os.Setenv("DEFAULT_TELEMETRY_ENDPOINT", "http://global:4317")
+	defer func() {
+		os.Unsetenv("DEFAULT_TELEMETRY_ENABLED")
+		os.Unsetenv("DEFAULT_TELEMETRY_ENDPOINT")
+	}()
+
+	tests := []struct {
+		name            string
+		componentConfig *kaosv1alpha1.TelemetryConfig
+		expectEnabled   bool
+		expectEndpoint  string
+	}{
+		{
+			name:            "uses global when component is nil",
+			componentConfig: nil,
+			expectEnabled:   true,
+			expectEndpoint:  "http://global:4317",
+		},
+		{
+			name: "uses component when specified",
+			componentConfig: &kaosv1alpha1.TelemetryConfig{
+				Enabled:  true,
+				Endpoint: "http://component:4317",
+			},
+			expectEnabled:  true,
+			expectEndpoint: "http://component:4317",
+		},
+		{
+			name: "component can disable telemetry",
+			componentConfig: &kaosv1alpha1.TelemetryConfig{
+				Enabled: false,
+			},
+			expectEnabled: false,
+		},
+	}
+
+	for _, tt := range tests {
+		t.Run(tt.name, func(t *testing.T) {
+			result := MergeTelemetryConfig(tt.componentConfig)
+
+			if result == nil {
+				t.Fatal("expected non-nil result")
+			}
+			if result.Enabled != tt.expectEnabled {
+				t.Errorf("expected Enabled=%v, got %v", tt.expectEnabled, result.Enabled)
+			}
+			if tt.expectEndpoint != "" && result.Endpoint != tt.expectEndpoint {
+				t.Errorf("expected Endpoint=%s, got %s", tt.expectEndpoint, result.Endpoint)
+			}
+		})
+	}
+}
+
+func TestBuildTelemetryEnvVars(t *testing.T) {
+	// Clear any existing env
+	os.Unsetenv("OTEL_RESOURCE_ATTRIBUTES")
+
+	tests := []struct {
+		name         string
+		tel          *kaosv1alpha1.TelemetryConfig
+		serviceName  string
+		namespace    string
+		expectCount  int
+		expectOTEL   bool
+	}{
+		{
+			name:        "nil config returns empty",
+			tel:         nil,
+			expectCount: 0,
+		},
+		{
+			name:        "disabled config returns empty",
+			tel:         &kaosv1alpha1.TelemetryConfig{Enabled: false},
+			expectCount: 0,
+		},
+		{
+			name: "enabled config returns env vars",
+			tel: &kaosv1alpha1.TelemetryConfig{
+				Enabled:  true,
+				Endpoint: "http://collector:4317",
+			},
+			serviceName: "test-agent",
+			namespace:   "default",
+			expectCount: 4, // OTEL_SDK_DISABLED, OTEL_SERVICE_NAME, OTEL_EXPORTER_OTLP_ENDPOINT, OTEL_RESOURCE_ATTRIBUTES
+			expectOTEL:  true,
+		},
+	}
+
+	for _, tt := range tests {
+		t.Run(tt.name, func(t *testing.T) {
+			result := BuildTelemetryEnvVars(tt.tel, tt.serviceName, tt.namespace)
+
+			if len(result) != tt.expectCount {
+				t.Errorf("expected %d env vars, got %d", tt.expectCount, len(result))
+			}
+
+			if tt.expectOTEL {
+				hasSDKDisabled := false
+				hasServiceName := false
+				for _, env := range result {
+					if env.Name == "OTEL_SDK_DISABLED" && env.Value == "false" {
+						hasSDKDisabled = true
+					}
+					if env.Name == "OTEL_SERVICE_NAME" && env.Value == tt.serviceName {
+						hasServiceName = true
+					}
+				}
+				if !hasSDKDisabled {
+					t.Error("expected OTEL_SDK_DISABLED=false")
+				}
+				if !hasServiceName {
+					t.Errorf("expected OTEL_SERVICE_NAME=%s", tt.serviceName)
+				}
+			}
+		})
+	}
+}
```

---

### Commit: ccc6ea2
**Subject:** fix(telemetry): critical bug fixes and ModelAPI extension
**Date:** 2026-01-26 20:45:32 +0100
**Author:** Alejandro Saucedo

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


```diff
 docs/operator/telemetry.md                         | 35 ++++++++-
 operator/api/v1alpha1/agent_types.go               |  5 +-
 operator/api/v1alpha1/modelapi_types.go            |  6 ++
 operator/api/v1alpha1/zz_generated.deepcopy.go     |  5 ++
 operator/config/crd/bases/kaos.tools_agents.yaml   |  6 +-
 .../config/crd/bases/kaos.tools_mcpservers.yaml    |  6 +-
 .../config/crd/bases/kaos.tools_modelapis.yaml     | 18 +++++
 operator/controllers/modelapi_controller.go        | 48 +++++++++++-
 operator/pkg/util/telemetry.go                     | 22 ++----
 python/agent/client.py                             | 17 ++---
 python/agent/server.py                             |  8 +-
 python/telemetry/manager.py                        | 47 +++++++++---
 python/tests/test_telemetry.py                     | 85 ++++++++++++++++++++--
 13 files changed, 248 insertions(+), 60 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index a84d3bb..903b325 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -72,7 +72,7 @@ spec:
 
 Component-level configuration always overrides global defaults:
 
-1. **Component-level telemetry** (highest priority): If `spec.config.telemetry` is set on the Agent or MCPServer, it is used
+1. **Component-level telemetry** (highest priority): If `spec.config.telemetry` is set on the Agent or MCPServer, or `spec.telemetry` on ModelAPI, it is used
 2. **Global Helm values** (default): If component-level telemetry is not set, the global `telemetry.enabled` and `telemetry.endpoint` values are used
 
 ## Configuration Fields
@@ -111,7 +111,38 @@ The operator automatically sets:
 - `OTEL_SDK_DISABLED`: Set to "false" when telemetry is enabled (standard OTel env var)
 - `OTEL_SERVICE_NAME`: Defaults to the CR name (e.g., agent name)
 - `OTEL_EXPORTER_OTLP_ENDPOINT`: From `telemetry.endpoint`
-- `OTEL_RESOURCE_ATTRIBUTES`: Appends `service.namespace` and `kaos.resource.name` to any user-provided values
+- `OTEL_RESOURCE_ATTRIBUTES`: Sets `service.namespace` and `kaos.resource.name`
+
+## ModelAPI Telemetry
+
+ModelAPI supports telemetry for the LiteLLM Proxy mode. For Ollama Hosted mode, telemetry is not supported (Ollama has no native OTel support).
+
+### LiteLLM Proxy Mode
+
+When `spec.telemetry.enabled: true` on a Proxy-mode ModelAPI:
+- LiteLLM config is automatically updated with `success_callback: ["otel"]` and `failure_callback: ["otel"]`
+- Environment variables `OTEL_EXPORTER`, `OTEL_ENDPOINT`, and `OTEL_SERVICE_NAME` are set
+- LiteLLM will send traces for each model call to the configured OTLP endpoint
+
+```yaml
+apiVersion: kaos.tools/v1alpha1
+kind: ModelAPI
+metadata:
+  name: my-modelapi
+spec:
+  mode: Proxy
+  telemetry:
+    enabled: true
+    endpoint: "http://otel-collector:4317"
+  proxyConfig:
+    models: ["*"]
+    provider: "openai"
+    apiBase: "https://api.openai.com/v1"
+```
+
+### Ollama Hosted Mode
+
+If telemetry is enabled for a Hosted-mode ModelAPI, the operator emits a warning in the status message. Ollama does not have native OpenTelemetry support, so traces and metrics will not be collected from the model server itself.
 
 ## Trace Spans
 
diff --git a/operator/api/v1alpha1/agent_types.go b/operator/api/v1alpha1/agent_types.go
index 25184af..f79d300 100644
--- a/operator/api/v1alpha1/agent_types.go
+++ b/operator/api/v1alpha1/agent_types.go
@@ -65,8 +65,9 @@ type TelemetryConfig struct {
 	// +kubebuilder:default=false
 	Enabled bool `json:"enabled,omitempty"`
 
-	// Endpoint is the OTLP endpoint URL (default: http://localhost:4317)
-	// +kubebuilder:default="http://localhost:4317"
+	// Endpoint is the OTLP gRPC endpoint URL (required when enabled).
+	// Example: "http://otel-collector.observability:4317"
+	// +kubebuilder:validation:Optional
 	Endpoint string `json:"endpoint,omitempty"`
 }
 
diff --git a/operator/api/v1alpha1/modelapi_types.go b/operator/api/v1alpha1/modelapi_types.go
index b01f755..1934ba1 100644
--- a/operator/api/v1alpha1/modelapi_types.go
+++ b/operator/api/v1alpha1/modelapi_types.go
@@ -124,6 +124,12 @@ type ModelAPISpec struct {
 	// +kubebuilder:validation:Optional
 	GatewayRoute *GatewayRoute `json:"gatewayRoute,omitempty"`
 
+	// Telemetry configures OpenTelemetry instrumentation.
+	// For Proxy mode (LiteLLM): Enables OTel callbacks for traces/metrics.
+	// For Hosted mode (Ollama): Not supported; a warning is emitted if enabled.
+	// +kubebuilder:validation:Optional
+	Telemetry *TelemetryConfig `json:"telemetry,omitempty"`
+
 	// PodSpec allows overriding the generated pod spec using strategic merge patch
 	// +kubebuilder:validation:Optional
 	PodSpec *corev1.PodSpec `json:"podSpec,omitempty"`
diff --git a/operator/api/v1alpha1/zz_generated.deepcopy.go b/operator/api/v1alpha1/zz_generated.deepcopy.go
index da64ee1..2db3638 100644
--- a/operator/api/v1alpha1/zz_generated.deepcopy.go
+++ b/operator/api/v1alpha1/zz_generated.deepcopy.go
@@ -601,6 +601,11 @@ func (in *ModelAPISpec) DeepCopyInto(out *ModelAPISpec) {
 		*out = new(GatewayRoute)
 		**out = **in
 	}
+	if in.Telemetry != nil {
+		in, out := &in.Telemetry, &out.Telemetry
+		*out = new(TelemetryConfig)
+		**out = **in
+	}
 	if in.PodSpec != nil {
 		in, out := &in.PodSpec, &out.PodSpec
 		*out = new(v1.PodSpec)
diff --git a/operator/config/crd/bases/kaos.tools_agents.yaml b/operator/config/crd/bases/kaos.tools_agents.yaml
index 10ea761..c5770aa 100644
--- a/operator/config/crd/bases/kaos.tools_agents.yaml
+++ b/operator/config/crd/bases/kaos.tools_agents.yaml
@@ -299,9 +299,9 @@ spec:
                           When enabled, traces, metrics, and log correlation are all active.
                         type: boolean
                       endpoint:
-                        default: http://localhost:4317
-                        description: 'Endpoint is the OTLP endpoint URL (default:
-                          http://localhost:4317)'
+                        description: |-
+                          Endpoint is the OTLP gRPC endpoint URL (required when enabled).
+                          Example: "http://otel-collector.observability:4317"
                         type: string
                     type: object
                 type: object
diff --git a/operator/config/crd/bases/kaos.tools_mcpservers.yaml b/operator/config/crd/bases/kaos.tools_mcpservers.yaml
index d062651..456ebf3 100644
--- a/operator/config/crd/bases/kaos.tools_mcpservers.yaml
+++ b/operator/config/crd/bases/kaos.tools_mcpservers.yaml
@@ -226,9 +226,9 @@ spec:
                           When enabled, traces, metrics, and log correlation are all active.
                         type: boolean
                       endpoint:
-                        default: http://localhost:4317
-                        description: 'Endpoint is the OTLP endpoint URL (default:
-                          http://localhost:4317)'
+                        description: |-
+                          Endpoint is the OTLP gRPC endpoint URL (required when enabled).
+                          Example: "http://otel-collector.observability:4317"
                         type: string
                     type: object
                   tools:
diff --git a/operator/config/crd/bases/kaos.tools_modelapis.yaml b/operator/config/crd/bases/kaos.tools_modelapis.yaml
index 79c2da2..794c7bc 100644
--- a/operator/config/crd/bases/kaos.tools_modelapis.yaml
+++ b/operator/config/crd/bases/kaos.tools_modelapis.yaml
@@ -8905,6 +8905,24 @@ spec:
                 required:
                 - models
                 type: object
+              telemetry:
+                description: |-
+                  Telemetry configures OpenTelemetry instrumentation.
+                  For Proxy mode (LiteLLM): Enables OTel callbacks for traces/metrics.
+                  For Hosted mode (Ollama): Not supported; a warning is emitted if enabled.
+                properties:
+                  enabled:
+                    default: false
+                    description: |-
+                      Enabled controls whether OpenTelemetry is enabled (default: false)
+                      When enabled, traces, metrics, and log correlation are all active.
+                    type: boolean
+                  endpoint:
+                    description: |-
+                      Endpoint is the OTLP gRPC endpoint URL (required when enabled).
+                      Example: "http://otel-collector.observability:4317"
+                    type: string
+                type: object
             required:
             - mode
             type: object
diff --git a/operator/controllers/modelapi_controller.go b/operator/controllers/modelapi_controller.go
index f443b72..c395d0b 100644
--- a/operator/controllers/modelapi_controller.go
+++ b/operator/controllers/modelapi_controller.go
@@ -90,6 +90,20 @@ func (r *ModelAPIReconciler) Reconcile(ctx context.Context, req ctrl.Request) (c
 	needsConfigMap := modelapi.Spec.Mode == kaosv1alpha1.ModelAPIModeProxy &&
 		modelapi.Spec.ProxyConfig != nil
 
+	// Warn if telemetry is enabled for Ollama (Hosted mode) - OTel not supported
+	if modelapi.Spec.Mode == kaosv1alpha1.ModelAPIModeHosted {
+		telemetry := util.MergeTelemetryConfig(modelapi.Spec.Telemetry)
+		if telemetry != nil && telemetry.Enabled {
+			log.Info("WARNING: OpenTelemetry telemetry is not supported for Ollama (Hosted mode). "+
+				"Traces and metrics will not be collected.", "modelapi", modelapi.Name)
+			// Update status message to warn user
+			if modelapi.Status.Message == "" {
+				modelapi.Status.Message = "Warning: Telemetry enabled but Ollama does not support OTel natively"
+				r.Status().Update(ctx, modelapi)
+			}
+		}
+	}
+
 	// Validate configYaml against models list if both are provided
 	if needsConfigMap && modelapi.Spec.ProxyConfig.ConfigYaml != nil &&
 		modelapi.Spec.ProxyConfig.ConfigYaml.FromString != "" {
@@ -456,6 +470,27 @@ func (r *ModelAPIReconciler) constructContainer(modelapi *kaosv1alpha1.ModelAPI)
 			})
 		}
 
+		// Add OTel env vars for LiteLLM when telemetry is enabled
+		telemetry := util.MergeTelemetryConfig(modelapi.Spec.Telemetry)
+		if telemetry != nil && telemetry.Enabled {
+			// LiteLLM uses OTEL_EXPORTER="otlp" to enable OTel exporter
+			env = append(env, corev1.EnvVar{
+				Name:  "OTEL_EXPORTER",
+				Value: "otlp",
+			})
+			if telemetry.Endpoint != "" {
+				env = append(env, corev1.EnvVar{
+					Name:  "OTEL_ENDPOINT",
+					Value: telemetry.Endpoint,
+				})
+			}
+			// Standard OTel service name
+			env = append(env, corev1.EnvVar{
+				Name:  "OTEL_SERVICE_NAME",
+				Value: modelapi.Name,
+			})
+		}
+
 	} else {
 		// Ollama Hosted mode
 		image = os.Getenv("DEFAULT_OLLAMA_IMAGE")
@@ -583,7 +618,9 @@ func (r *ModelAPIReconciler) constructConfigMap(modelapi *kaosv1alpha1.ModelAPI)
 			configYaml = modelapi.Spec.ProxyConfig.ConfigYaml.FromString
 		} else {
 			// Generate config from models list (models is required with MinItems=1)
-			configYaml = r.generateLiteLLMConfig(modelapi.Spec.ProxyConfig)
+			// Pass merged telemetry config for OTel callback
+			telemetry := util.MergeTelemetryConfig(modelapi.Spec.Telemetry)
+			configYaml = r.generateLiteLLMConfig(modelapi.Spec.ProxyConfig, telemetry)
 		}
 	}
 
@@ -611,7 +648,8 @@ func (r *ModelAPIReconciler) constructConfigMap(modelapi *kaosv1alpha1.ModelAPI)
 // Wildcard handling:
 // - models: ["*"] with provider: "nebius" → model_name: "*" → model: "nebius/*"
 // - models: ["*"] without provider → model_name: "*" → model: "*"
-func (r *ModelAPIReconciler) generateLiteLLMConfig(proxyConfig *kaosv1alpha1.ProxyConfig) string {
+// When telemetry is enabled, adds OTel callback for traces/metrics.
+func (r *ModelAPIReconciler) generateLiteLLMConfig(proxyConfig *kaosv1alpha1.ProxyConfig, telemetry *kaosv1alpha1.TelemetryConfig) string {
 	var sb strings.Builder
 
 	sb.WriteString("# Auto-generated LiteLLM config\n")
@@ -650,6 +688,12 @@ func (r *ModelAPIReconciler) generateLiteLLMConfig(proxyConfig *kaosv1alpha1.Pro
 	sb.WriteString("\nlitellm_settings:\n")
 	sb.WriteString("  drop_params: true\n")
 
+	// Add OTel callback when telemetry is enabled
+	if telemetry != nil && telemetry.Enabled {
+		sb.WriteString("  success_callback: [\"otel\"]\n")
+		sb.WriteString("  failure_callback: [\"otel\"]\n")
+	}
+
 	return sb.String()
 }
 
diff --git a/operator/pkg/util/telemetry.go b/operator/pkg/util/telemetry.go
index 9409c27..e87dd46 100644
--- a/operator/pkg/util/telemetry.go
+++ b/operator/pkg/util/telemetry.go
@@ -2,7 +2,6 @@ package util
 
 import (
 	"os"
-	"strings"
 
 	corev1 "k8s.io/api/core/v1"
 
@@ -35,7 +34,9 @@ func MergeTelemetryConfig(componentConfig *kaosv1alpha1.TelemetryConfig) *kaosv1
 // BuildTelemetryEnvVars creates environment variables for OpenTelemetry configuration.
 // Uses standard OTEL_* env vars so the SDK auto-configures.
 // serviceName is used as OTEL_SERVICE_NAME (typically the CR name).
-// namespace is added to OTEL_RESOURCE_ATTRIBUTES (appended to existing user values).
+// namespace is added to OTEL_RESOURCE_ATTRIBUTES as KAOS-specific attributes.
+// Note: If user sets OTEL_RESOURCE_ATTRIBUTES in spec.config.env, both will be present
+// and the user value takes precedence when they appear later in the env list.
 func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, namespace string) []corev1.EnvVar {
 	if tel == nil || !tel.Enabled {
 		return nil
@@ -59,22 +60,13 @@ func BuildTelemetryEnvVars(tel *kaosv1alpha1.TelemetryConfig, serviceName, names
 		})
 	}
 
-	// Build resource attributes - append to existing user values if OTEL_RESOURCE_ATTRIBUTES is set
+	// Add KAOS-specific resource attributes
+	// These are added as a baseline; if user also sets OTEL_RESOURCE_ATTRIBUTES
+	// in spec.config.env, the container runtime merges them (later values win)
 	kaosAttrs := "service.namespace=" + namespace + ",kaos.resource.name=" + serviceName
-	existingAttrs := os.Getenv("OTEL_RESOURCE_ATTRIBUTES")
-	var finalAttrs string
-	if existingAttrs != "" {
-		// Append KAOS attrs to user attrs (user attrs take precedence for same keys)
-		finalAttrs = existingAttrs + "," + kaosAttrs
-	} else {
-		finalAttrs = kaosAttrs
-	}
-	// Remove any duplicate trailing/leading commas
-	finalAttrs = strings.Trim(finalAttrs, ",")
-
 	envVars = append(envVars, corev1.EnvVar{
 		Name:  "OTEL_RESOURCE_ATTRIBUTES",
-		Value: finalAttrs,
+		Value: kaosAttrs,
 	})
 
 	return envVars
diff --git a/python/agent/client.py b/python/agent/client.py
index b5cab6b..6aae4fc 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -390,7 +390,7 @@ class Agent:
             error_event = self.memory.create_event("error", error_msg)
             await self.memory.add_event(session_id, error_event)
             yield f"Sorry, I encountered an error: {str(e)}"
-        finally:
+        else:
             self._otel.span_success()
 
     async def _agentic_loop(
@@ -491,7 +491,7 @@ class Agent:
             except Exception as e:
                 self._otel.span_failure(e)
                 raise
-            finally:
+            else:
                 self._otel.span_success()
 
         # Max steps reached
@@ -510,12 +510,12 @@ class Agent:
         )
         try:
             content = cast(str, await self.model_api.process_message(messages, stream=False))
-            return content
         except Exception as e:
             self._otel.span_failure(e)
             raise
-        finally:
+        else:
             self._otel.span_success()
+            return content
 
     async def _execute_tool(self, tool_name: str, tool_args: Dict[str, Any]) -> Any:
         """Execute a tool with tracing."""
@@ -535,13 +535,12 @@ class Agent:
 
             if tool_result is None:
                 raise ValueError(f"Tool '{tool_name}' not found")
-
-            return tool_result
         except Exception as e:
             self._otel.span_failure(e)
             raise
-        finally:
+        else:
             self._otel.span_success()
+            return tool_result
 
     async def _execute_delegation(
         self,
@@ -562,12 +561,12 @@ class Agent:
             result = await self.delegate_to_sub_agent(
                 agent_name, task, context_messages, session_id
             )
-            return result
         except Exception as e:
             self._otel.span_failure(e)
             raise
-        finally:
+        else:
             self._otel.span_success()
+            return result
 
     async def delegate_to_sub_agent(
         self,
diff --git a/python/agent/server.py b/python/agent/server.py
index 45ee04f..e3964af 100644
--- a/python/agent/server.py
+++ b/python/agent/server.py
@@ -23,7 +23,7 @@ from modelapi.client import ModelAPI
 from agent.client import Agent, RemoteAgent
 from agent.memory import LocalMemory
 from mcptools.client import MCPClient
-from telemetry.manager import init_otel, is_otel_enabled
+from telemetry.manager import init_otel, is_otel_enabled, should_enable_otel
 
 
 def configure_logging(level: str = "INFO", otel_correlation: bool = False) -> None:
@@ -473,11 +473,11 @@ def create_agent_server(
         # Load from environment variables - requires AGENT_NAME and MODEL_API_URL
         settings = AgentServerSettings()  # type: ignore[call-arg]
 
-    # Check if OTel enabled via env var
-    otel_enabled = is_otel_enabled()
+    # Check if OTel should be enabled based on env vars (before init_otel)
+    otel_should_enable = should_enable_otel()
 
     # Configure logging with optional OTel correlation
-    configure_logging(settings.agent_log_level, otel_correlation=otel_enabled)
+    configure_logging(settings.agent_log_level, otel_correlation=otel_should_enable)
 
     model_api = ModelAPI(model=settings.model_name, api_base=settings.model_api_url)
 
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index bc51e81..14fbfca 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -61,7 +61,8 @@ class SpanState:
 
 
 # Async-safe span stack per context (supports nesting)
-_span_stack: ContextVar[List[SpanState]] = ContextVar("kaos_span_stack", default=[])
+# default=None to avoid shared mutable list across async contexts
+_span_stack: ContextVar[Optional[List[SpanState]]] = ContextVar("kaos_span_stack", default=None)
 
 
 class OtelConfig(BaseSettings):
@@ -97,6 +98,25 @@ def is_otel_enabled() -> bool:
     return _initialized
 
 
+def should_enable_otel() -> bool:
+    """Check if OTel should be enabled based on environment variables.
+
+    This checks env vars BEFORE init_otel() is called, useful for deciding
+    whether to enable log correlation before the SDK is initialized.
+
+    Returns True if OTEL_SDK_DISABLED is not set to true AND required env vars
+    (OTEL_SERVICE_NAME, OTEL_EXPORTER_OTLP_ENDPOINT) are configured.
+    """
+    disabled = os.getenv("OTEL_SDK_DISABLED", "false").lower() in ("true", "1", "yes")
+    if disabled:
+        return False
+
+    # Check if required env vars are set
+    service_name = os.getenv("OTEL_SERVICE_NAME", "")
+    endpoint = os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "")
+    return bool(service_name and endpoint)
+
+
 def init_otel(service_name: Optional[str] = None) -> bool:
     """Initialize OpenTelemetry with standard OTEL_* env vars.
 
@@ -146,14 +166,15 @@ def init_otel(service_name: Optional[str] = None) -> bool:
         CompositePropagator([TraceContextTextMapPropagator(), W3CBaggagePropagator()])
     )
 
-    # Initialize tracing - let SDK use OTEL_EXPORTER_OTLP_* env vars for advanced config
+    # Initialize tracing - let SDK use OTEL_EXPORTER_OTLP_* env vars for TLS, headers, etc.
+    # By not passing endpoint explicitly, SDK will read from OTEL_EXPORTER_OTLP_ENDPOINT
     tracer_provider = TracerProvider(resource=resource)
-    otlp_span_exporter = OTLPSpanExporter(endpoint=config.otel_exporter_otlp_endpoint)
+    otlp_span_exporter = OTLPSpanExporter()  # Uses OTEL_EXPORTER_OTLP_* env vars
     tracer_provider.add_span_processor(BatchSpanProcessor(otlp_span_exporter))
     trace.set_tracer_provider(tracer_provider)
 
-    # Initialize metrics
-    otlp_metric_exporter = OTLPMetricExporter(endpoint=config.otel_exporter_otlp_endpoint)
+    # Initialize metrics - also uses env vars for endpoint, TLS config, etc.
+    otlp_metric_exporter = OTLPMetricExporter()  # Uses OTEL_EXPORTER_OTLP_* env vars
     metric_reader = PeriodicExportingMetricReader(otlp_metric_exporter)
     meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
     metrics.set_meter_provider(meter_provider)
@@ -181,7 +202,7 @@ class KaosOtelManager:
         except Exception as e:
             otel.span_failure(e)
             raise
-        finally:
+        else:
             otel.span_success()
     """
 
@@ -236,13 +257,15 @@ class KaosOtelManager:
         )
 
     def _get_stack(self) -> List[SpanState]:
-        """Get or create the span stack for current async context."""
-        try:
-            return _span_stack.get()
-        except LookupError:
-            stack: List[SpanState] = []
+        """Get or create the span stack for current async context.
+
+        Allocates a new list per-context to avoid sharing mutable state.
+        """
+        stack = _span_stack.get()
+        if stack is None:
+            stack = []
             _span_stack.set(stack)
-            return stack
+        return stack
 
     def span_begin(
         self,
diff --git a/python/tests/test_telemetry.py b/python/tests/test_telemetry.py
index 81ccf42..5c79905 100644
--- a/python/tests/test_telemetry.py
+++ b/python/tests/test_telemetry.py
@@ -24,6 +24,57 @@ class TestIsOtelEnabled:
             tm._initialized = original
 
 
+class TestShouldEnableOtel:
+    """Tests for should_enable_otel utility."""
+
+    def test_returns_false_when_disabled(self):
+        """Test should_enable_otel returns False when OTEL_SDK_DISABLED=true."""
+        with patch.dict(
+            os.environ,
+            {
+                "OTEL_SDK_DISABLED": "true",
+                "OTEL_SERVICE_NAME": "test-agent",
+                "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
+            },
+            clear=True,
+        ):
+            from telemetry.manager import should_enable_otel
+
+            assert should_enable_otel() is False
+
+    def test_returns_false_without_service_name(self):
+        """Test should_enable_otel returns False without OTEL_SERVICE_NAME."""
+        with patch.dict(
+            os.environ,
+            {"OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317"},
+            clear=True,
+        ):
+            from telemetry.manager import should_enable_otel
+
+            assert should_enable_otel() is False
+
+    def test_returns_false_without_endpoint(self):
+        """Test should_enable_otel returns False without OTEL_EXPORTER_OTLP_ENDPOINT."""
+        with patch.dict(os.environ, {"OTEL_SERVICE_NAME": "test-agent"}, clear=True):
+            from telemetry.manager import should_enable_otel
+
+            assert should_enable_otel() is False
+
+    def test_returns_true_with_required_vars(self):
+        """Test should_enable_otel returns True with required env vars."""
+        with patch.dict(
+            os.environ,
+            {
+                "OTEL_SERVICE_NAME": "test-agent",
+                "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
+            },
+            clear=True,
+        ):
+            from telemetry.manager import should_enable_otel
+
+            assert should_enable_otel() is True
+
+
 class TestOtelConfig:
     """Tests for OtelConfig pydantic BaseSettings."""
 
@@ -102,7 +153,10 @@ class TestKaosOtelManager:
         manager.span_begin("test-operation")
         try:
             pass  # do work
-        finally:
+        except Exception as e:
+            manager.span_failure(e)
+            raise
+        else:
             manager.span_success()
 
     def test_span_begin_failure_pattern(self):
@@ -115,8 +169,8 @@ class TestKaosOtelManager:
             raise ValueError("test error")
         except ValueError as e:
             manager.span_failure(e)
-        finally:
-            manager.span_success()  # Should be no-op after failure
+        else:
+            manager.span_success()
 
     def test_nested_spans(self):
         """Test nested span_begin calls."""
@@ -128,9 +182,15 @@ class TestKaosOtelManager:
             manager.span_begin("inner")
             try:
                 pass
-            finally:
+            except Exception as e:
+                manager.span_failure(e)
+                raise
+            else:
                 manager.span_success()
-        finally:
+        except Exception as e:
+            manager.span_failure(e)
+            raise
+        else:
             manager.span_success()
 
     def test_span_with_metric_kind(self):
@@ -145,7 +205,10 @@ class TestKaosOtelManager:
         )
         try:
             pass
-        finally:
+        except Exception as e:
+            manager.span_failure(e)
+            raise
+        else:
             manager.span_success()
 
 
@@ -172,13 +235,19 @@ class TestContextPropagation:
 class TestMCPServerTelemetrySimplified:
     """Tests for MCPServer simplified telemetry settings."""
 
-    def test_otel_enabled_by_default(self):
-        """Test that OTel is enabled by default (OTEL_SDK_DISABLED not set)."""
+    def test_otel_not_disabled_by_default(self):
+        """Test that OTel is not disabled by default (OTEL_SDK_DISABLED not set).
+
+        Note: _otel_enabled=True means "not disabled", not "fully configured".
+        Actual telemetry requires OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT.
+        """
         with patch.dict(os.environ, {}, clear=True):
             from mcptools.server import MCPServer, MCPServerSettings
 
             settings = MCPServerSettings()
             server = MCPServer(settings)
+            # _otel_enabled means "not disabled", telemetry will be a no-op without
+            # OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT
             assert server._otel_enabled is True
 
     def test_otel_disabled_from_env(self):
```

---

### Commit: f6ca856
**Subject:** docs(telemetry): clarify OTEL_RESOURCE_ATTRIBUTES behavior
**Date:** 2026-01-26 21:05:07 +0100
**Author:** Alejandro Saucedo

- Clarify that operator sets attributes directly (not appends)
- Note that user values in spec.config.env take precedence


```diff
 docs/operator/telemetry.md | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index 903b325..81ff8a3 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -392,7 +392,7 @@ The operator automatically sets these environment variables when telemetry is en
 | `OTEL_SDK_DISABLED` | "false" when telemetry is enabled (standard OTel env var) |
 | `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP endpoint URL from `telemetry.endpoint` |
 | `OTEL_SERVICE_NAME` | Defaults to CR name (agent or MCP server name) |
-| `OTEL_RESOURCE_ATTRIBUTES` | Appends `service.namespace` and `kaos.resource.name` to user values |
+| `OTEL_RESOURCE_ATTRIBUTES` | Sets `service.namespace` and `kaos.resource.name`; if user sets same var in spec.config.env, their value takes precedence |
 
 For additional configuration, use standard [OpenTelemetry environment variables](https://opentelemetry-python.readthedocs.io/en/latest/sdk/environment_variables.html) via `spec.config.env`.
 
```

---

### Commit: 775ad49
**Subject:** fix(telemetry): prevent span leakage with try/except/finally pattern
**Date:** 2026-01-27 06:53:56 +0100
**Author:** Alejandro Saucedo

Replace try/except/else with try/except/finally using failed flag to ensure
spans always close, even on early return, yield, or continue statements.

Affected functions:
- process_message(): generator may not reach else on early exit
- _agentic_loop(): contains return/continue that bypass else block
- _call_model, _execute_tool, _execute_delegation: updated for consistency

The pattern ensures span_success() runs in finally unless span_failure()
already handled the span, preventing span leakage in streaming scenarios.


```diff
 python/agent/client.py | 43 ++++++++++++++++++++++++++++++-------------
 1 file changed, 30 insertions(+), 13 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/agent/client.py b/python/agent/client.py
index 6aae4fc..21f3b16 100644
--- a/python/agent/client.py
+++ b/python/agent/client.py
@@ -342,6 +342,8 @@ class Agent:
             attrs=span_attrs,
             metric_kind="request",
         )
+        # Use failed flag pattern to ensure spans close on return/yield/early exit
+        span_failed = False
         try:
             # Extract user-provided system prompt (if any) from message array
             user_system_prompt: Optional[str] = None
@@ -384,14 +386,16 @@ class Agent:
                 yield chunk
 
         except Exception as e:
+            span_failed = True
             self._otel.span_failure(e)
             error_msg = f"Error processing message: {str(e)}"
             logger.error(error_msg)
             error_event = self.memory.create_event("error", error_msg)
             await self.memory.add_event(session_id, error_event)
             yield f"Sorry, I encountered an error: {str(e)}"
-        else:
-            self._otel.span_success()
+        finally:
+            if not span_failed:
+                self._otel.span_success()
 
     async def _agentic_loop(
         self,
@@ -406,6 +410,8 @@ class Agent:
             # Start step span
             step_attrs = {"step": step + 1, "max_steps": self.max_steps}
             self._otel.span_begin(f"agent.step.{step + 1}", attrs=step_attrs)
+            # Use failed flag pattern to ensure spans close on continue/return/yield
+            step_failed = False
             try:
                 # Get model response
                 model_name = self.model_api.model if self.model_api else "unknown"
@@ -489,10 +495,12 @@ class Agent:
                 return
 
             except Exception as e:
+                step_failed = True
                 self._otel.span_failure(e)
                 raise
-            else:
-                self._otel.span_success()
+            finally:
+                if not step_failed:
+                    self._otel.span_success()
 
         # Max steps reached
         max_steps_msg = f"Reached maximum reasoning steps ({self.max_steps})"
@@ -508,14 +516,17 @@ class Agent:
             metric_kind="model",
             metric_attrs={"model": model_name},
         )
+        failed = False
         try:
             content = cast(str, await self.model_api.process_message(messages, stream=False))
+            return content
         except Exception as e:
+            failed = True
             self._otel.span_failure(e)
             raise
-        else:
-            self._otel.span_success()
-            return content
+        finally:
+            if not failed:
+                self._otel.span_success()
 
     async def _execute_tool(self, tool_name: str, tool_args: Dict[str, Any]) -> Any:
         """Execute a tool with tracing."""
@@ -526,6 +537,7 @@ class Agent:
             metric_kind="tool",
             metric_attrs={"tool": tool_name},
         )
+        failed = False
         try:
             tool_result = None
             for mcp_client in self.mcp_clients:
@@ -535,12 +547,14 @@ class Agent:
 
             if tool_result is None:
                 raise ValueError(f"Tool '{tool_name}' not found")
+            return tool_result
         except Exception as e:
+            failed = True
             self._otel.span_failure(e)
             raise
-        else:
-            self._otel.span_success()
-            return tool_result
+        finally:
+            if not failed:
+                self._otel.span_success()
 
     async def _execute_delegation(
         self,
@@ -557,16 +571,19 @@ class Agent:
             metric_kind="delegation",
             metric_attrs={"target": agent_name},
         )
+        failed = False
         try:
             result = await self.delegate_to_sub_agent(
                 agent_name, task, context_messages, session_id
             )
+            return result
         except Exception as e:
+            failed = True
             self._otel.span_failure(e)
             raise
-        else:
-            self._otel.span_success()
-            return result
+        finally:
+            if not failed:
+                self._otel.span_success()
 
     async def delegate_to_sub_agent(
         self,
```

---

### Commit: f4922f7
**Subject:** fix(telemetry): MCPServer uses should_enable_otel() for proper detection
**Date:** 2026-01-27 06:55:14 +0100
**Author:** Alejandro Saucedo

MCPServer was treating 'not disabled' as 'enabled', which enabled OTel
instrumentation/log correlation without required env vars, resulting in
a misconfigured state.

Now uses should_enable_otel() which requires:
- OTEL_SDK_DISABLED != 'true'
- OTEL_SERVICE_NAME is set
- OTEL_EXPORTER_OTLP_ENDPOINT is set

This aligns MCPServer behavior with AgentServer and ensures OTel is only
enabled when properly configured.


```diff
 python/mcptools/server.py      |  8 +++++---
 python/tests/test_telemetry.py | 39 +++++++++++++++++++++++++++++++--------
 2 files changed, 36 insertions(+), 11 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/mcptools/server.py b/python/mcptools/server.py
index 41c6cd8..e447af7 100644
--- a/python/mcptools/server.py
+++ b/python/mcptools/server.py
@@ -84,9 +84,11 @@ class MCPServer:
 
     def __init__(self, settings: MCPServerSettings):
         """Initialize MCP server."""
-        # Check if OTel enabled via env var (OTEL_SDK_DISABLED=true disables)
-        otel_disabled = os.getenv("OTEL_SDK_DISABLED", "false").lower() in ("true", "1", "yes")
-        otel_enabled = not otel_disabled
+        # Check if OTel should be enabled using the shared utility
+        # This requires both OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT
+        from telemetry.manager import should_enable_otel
+
+        otel_enabled = should_enable_otel()
 
         # Configure logging with optional OTel correlation
         configure_logging(settings.mcp_log_level, otel_correlation=otel_enabled)
diff --git a/python/tests/test_telemetry.py b/python/tests/test_telemetry.py
index 5c79905..3f2882d 100644
--- a/python/tests/test_telemetry.py
+++ b/python/tests/test_telemetry.py
@@ -233,26 +233,49 @@ class TestContextPropagation:
 
 
 class TestMCPServerTelemetrySimplified:
-    """Tests for MCPServer simplified telemetry settings."""
+    """Tests for MCPServer telemetry detection using should_enable_otel()."""
 
-    def test_otel_not_disabled_by_default(self):
-        """Test that OTel is not disabled by default (OTEL_SDK_DISABLED not set).
+    def test_otel_disabled_without_required_env_vars(self):
+        """Test that OTel is disabled when required env vars are missing.
 
-        Note: _otel_enabled=True means "not disabled", not "fully configured".
-        Actual telemetry requires OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT.
+        MCPServer now uses should_enable_otel() which requires both
+        OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT.
         """
         with patch.dict(os.environ, {}, clear=True):
             from mcptools.server import MCPServer, MCPServerSettings
 
             settings = MCPServerSettings()
             server = MCPServer(settings)
-            # _otel_enabled means "not disabled", telemetry will be a no-op without
-            # OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT
+            # Without required env vars, telemetry should be disabled
+            assert server._otel_enabled is False
+
+    def test_otel_enabled_with_required_env_vars(self):
+        """Test that OTel is enabled when required env vars are set."""
+        with patch.dict(
+            os.environ,
+            {
+                "OTEL_SERVICE_NAME": "test-mcp",
+                "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
+            },
+            clear=True,
+        ):
+            from mcptools.server import MCPServer, MCPServerSettings
+
+            settings = MCPServerSettings()
+            server = MCPServer(settings)
             assert server._otel_enabled is True
 
     def test_otel_disabled_from_env(self):
         """Test that OTel can be disabled via OTEL_SDK_DISABLED env var."""
-        with patch.dict(os.environ, {"OTEL_SDK_DISABLED": "true"}, clear=True):
+        with patch.dict(
+            os.environ,
+            {
+                "OTEL_SDK_DISABLED": "true",
+                "OTEL_SERVICE_NAME": "test-mcp",
+                "OTEL_EXPORTER_OTLP_ENDPOINT": "http://collector:4317",
+            },
+            clear=True,
+        ):
             from mcptools.server import MCPServer, MCPServerSettings
 
             settings = MCPServerSettings()
```

---

### Commit: 210dc69
**Subject:** fix(operator): use standard OTEL_EXPORTER_OTLP_ENDPOINT for LiteLLM
**Date:** 2026-01-27 06:56:27 +0100
**Author:** Alejandro Saucedo

Changed OTEL_ENDPOINT to OTEL_EXPORTER_OTLP_ENDPOINT in ModelAPI controller
to align with OpenTelemetry standard environment variables.

Also documented that custom configYaml requires manual OTel callback setup.


```diff
 docs/operator/telemetry.md                  | 14 +++++++++++++-
 operator/controllers/modelapi_controller.go |  3 ++-
 2 files changed, 15 insertions(+), 2 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index 81ff8a3..d68177c 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -121,7 +121,7 @@ ModelAPI supports telemetry for the LiteLLM Proxy mode. For Ollama Hosted mode,
 
 When `spec.telemetry.enabled: true` on a Proxy-mode ModelAPI:
 - LiteLLM config is automatically updated with `success_callback: ["otel"]` and `failure_callback: ["otel"]`
-- Environment variables `OTEL_EXPORTER`, `OTEL_ENDPOINT`, and `OTEL_SERVICE_NAME` are set
+- Environment variables `OTEL_EXPORTER`, `OTEL_EXPORTER_OTLP_ENDPOINT`, and `OTEL_SERVICE_NAME` are set
 - LiteLLM will send traces for each model call to the configured OTLP endpoint
 
 ```yaml
@@ -140,6 +140,18 @@ spec:
     apiBase: "https://api.openai.com/v1"
 ```
 
+::: warning Custom configYaml
+If you provide a custom LiteLLM configuration via `proxyConfig.configYaml`, you must manually add the OTel callbacks:
+
+```yaml
+litellm_settings:
+  success_callback: ["otel"]
+  failure_callback: ["otel"]
+```
+
+The operator only auto-injects callbacks when using the default generated config.
+:::
+
 ### Ollama Hosted Mode
 
 If telemetry is enabled for a Hosted-mode ModelAPI, the operator emits a warning in the status message. Ollama does not have native OpenTelemetry support, so traces and metrics will not be collected from the model server itself.
diff --git a/operator/controllers/modelapi_controller.go b/operator/controllers/modelapi_controller.go
index c395d0b..a086559 100644
--- a/operator/controllers/modelapi_controller.go
+++ b/operator/controllers/modelapi_controller.go
@@ -479,8 +479,9 @@ func (r *ModelAPIReconciler) constructContainer(modelapi *kaosv1alpha1.ModelAPI)
 				Value: "otlp",
 			})
 			if telemetry.Endpoint != "" {
+				// Use standard OTEL_EXPORTER_OTLP_ENDPOINT env var
 				env = append(env, corev1.EnvVar{
-					Name:  "OTEL_ENDPOINT",
+					Name:  "OTEL_EXPORTER_OTLP_ENDPOINT",
 					Value: telemetry.Endpoint,
 				})
 			}
```

---

### Commit: 2eacb36
**Subject:** fix(operator): implement field-wise telemetry config merge with validation
**Date:** 2026-01-27 07:00:55 +0100
**Author:** Alejandro Saucedo

MergeTelemetryConfig now performs field-wise merge instead of all-or-nothing:
- Component can set enabled=true and inherit global endpoint
- Endpoint field: component wins if set, otherwise inherits global

Added IsTelemetryConfigValid() to check for enabled=true with empty endpoint.
Controllers now emit warnings when telemetry config is invalid.


```diff
 operator/controllers/agent_controller.go     | 10 +++++
 operator/controllers/mcpserver_controller.go |  6 +++
 operator/controllers/modelapi_controller.go  |  7 +++-
 operator/pkg/util/telemetry.go               | 41 +++++++++++++++++---
 operator/pkg/util/telemetry_test.go          | 56 +++++++++++++++++++++++++++-
 5 files changed, 112 insertions(+), 8 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/operator/controllers/agent_controller.go b/operator/controllers/agent_controller.go
index 33fbeea..68aea46 100644
--- a/operator/controllers/agent_controller.go
+++ b/operator/controllers/agent_controller.go
@@ -88,6 +88,16 @@ func (r *AgentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl
 		}
 	}
 
+	// Validate telemetry config
+	var componentTelemetry *kaosv1alpha1.TelemetryConfig
+	if agent.Spec.Config != nil {
+		componentTelemetry = agent.Spec.Config.Telemetry
+	}
+	telemetryConfig := util.MergeTelemetryConfig(componentTelemetry)
+	if !util.IsTelemetryConfigValid(telemetryConfig) {
+		log.Info("WARNING: telemetry.enabled=true but endpoint is empty; telemetry will not function", "agent", agent.Name)
+	}
+
 	// Resolve ModelAPI reference
 	modelapi := &kaosv1alpha1.ModelAPI{}
 	err := r.Get(ctx, types.NamespacedName{Name: agent.Spec.ModelAPI, Namespace: agent.Namespace}, modelapi)
diff --git a/operator/controllers/mcpserver_controller.go b/operator/controllers/mcpserver_controller.go
index 3ea7314..c97634f 100644
--- a/operator/controllers/mcpserver_controller.go
+++ b/operator/controllers/mcpserver_controller.go
@@ -83,6 +83,12 @@ func (r *MCPServerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (
 		}
 	}
 
+	// Validate telemetry config
+	telemetryConfig := util.MergeTelemetryConfig(mcpserver.Spec.Config.Telemetry)
+	if !util.IsTelemetryConfigValid(telemetryConfig) {
+		log.Info("WARNING: telemetry.enabled=true but endpoint is empty; telemetry will not function", "mcpserver", mcpserver.Name)
+	}
+
 	// Create or update Deployment
 	deployment := &appsv1.Deployment{}
 	deploymentName := fmt.Sprintf("mcpserver-%s", mcpserver.Name)
diff --git a/operator/controllers/modelapi_controller.go b/operator/controllers/modelapi_controller.go
index a086559..35aa3b1 100644
--- a/operator/controllers/modelapi_controller.go
+++ b/operator/controllers/modelapi_controller.go
@@ -90,9 +90,14 @@ func (r *ModelAPIReconciler) Reconcile(ctx context.Context, req ctrl.Request) (c
 	needsConfigMap := modelapi.Spec.Mode == kaosv1alpha1.ModelAPIModeProxy &&
 		modelapi.Spec.ProxyConfig != nil
 
+	// Validate telemetry config
+	telemetry := util.MergeTelemetryConfig(modelapi.Spec.Telemetry)
+	if !util.IsTelemetryConfigValid(telemetry) {
+		log.Info("WARNING: telemetry.enabled=true but endpoint is empty; telemetry will not function", "modelapi", modelapi.Name)
+	}
+
 	// Warn if telemetry is enabled for Ollama (Hosted mode) - OTel not supported
 	if modelapi.Spec.Mode == kaosv1alpha1.ModelAPIModeHosted {
-		telemetry := util.MergeTelemetryConfig(modelapi.Spec.Telemetry)
 		if telemetry != nil && telemetry.Enabled {
 			log.Info("WARNING: OpenTelemetry telemetry is not supported for Ollama (Hosted mode). "+
 				"Traces and metrics will not be collected.", "modelapi", modelapi.Name)
diff --git a/operator/pkg/util/telemetry.go b/operator/pkg/util/telemetry.go
index e87dd46..0b55109 100644
--- a/operator/pkg/util/telemetry.go
+++ b/operator/pkg/util/telemetry.go
@@ -20,15 +20,44 @@ func GetDefaultTelemetryConfig() *kaosv1alpha1.TelemetryConfig {
 	}
 }
 
-// MergeTelemetryConfig merges component-level telemetry config with global defaults.
-// Component-level config takes precedence over global defaults.
+// MergeTelemetryConfig performs field-wise merge of component-level telemetry config
+// with global defaults. Component-level fields take precedence when set.
+// This allows a component to set enabled=true and inherit the global endpoint.
 func MergeTelemetryConfig(componentConfig *kaosv1alpha1.TelemetryConfig) *kaosv1alpha1.TelemetryConfig {
-	// If component has explicit config, use it
-	if componentConfig != nil {
+	globalConfig := GetDefaultTelemetryConfig()
+
+	// If no component config, use global (may be nil)
+	if componentConfig == nil {
+		return globalConfig
+	}
+
+	// If no global config, use component as-is
+	if globalConfig == nil {
 		return componentConfig
 	}
-	// Otherwise fall back to global defaults
-	return GetDefaultTelemetryConfig()
+
+	// Field-wise merge: component fields take precedence if set
+	merged := &kaosv1alpha1.TelemetryConfig{
+		Enabled: componentConfig.Enabled, // Component controls enabled state
+	}
+
+	// Endpoint: use component if set, otherwise inherit global
+	if componentConfig.Endpoint != "" {
+		merged.Endpoint = componentConfig.Endpoint
+	} else {
+		merged.Endpoint = globalConfig.Endpoint
+	}
+
+	return merged
+}
+
+// IsTelemetryConfigValid returns true if the telemetry config is valid.
+// A valid config has enabled=true and non-empty endpoint.
+func IsTelemetryConfigValid(tel *kaosv1alpha1.TelemetryConfig) bool {
+	if tel == nil || !tel.Enabled {
+		return true // disabled is valid (just means no telemetry)
+	}
+	return tel.Endpoint != ""
 }
 
 // BuildTelemetryEnvVars creates environment variables for OpenTelemetry configuration.
diff --git a/operator/pkg/util/telemetry_test.go b/operator/pkg/util/telemetry_test.go
index 03a09dd..f60d9e2 100644
--- a/operator/pkg/util/telemetry_test.go
+++ b/operator/pkg/util/telemetry_test.go
@@ -104,7 +104,17 @@ func TestMergeTelemetryConfig(t *testing.T) {
 			componentConfig: &kaosv1alpha1.TelemetryConfig{
 				Enabled: false,
 			},
-			expectEnabled: false,
+			expectEnabled:  false,
+			expectEndpoint: "http://global:4317", // inherits global endpoint
+		},
+		{
+			name: "inherits global endpoint when component sets enabled but no endpoint",
+			componentConfig: &kaosv1alpha1.TelemetryConfig{
+				Enabled: true,
+				// Endpoint not set
+			},
+			expectEnabled:  true,
+			expectEndpoint: "http://global:4317", // inherited from global
 		},
 	}
 
@@ -125,6 +135,50 @@ func TestMergeTelemetryConfig(t *testing.T) {
 	}
 }
 
+func TestIsTelemetryConfigValid(t *testing.T) {
+	tests := []struct {
+		name   string
+		tel    *kaosv1alpha1.TelemetryConfig
+		expect bool
+	}{
+		{
+			name:   "nil is valid",
+			tel:    nil,
+			expect: true,
+		},
+		{
+			name:   "disabled is valid",
+			tel:    &kaosv1alpha1.TelemetryConfig{Enabled: false},
+			expect: true,
+		},
+		{
+			name: "enabled with endpoint is valid",
+			tel: &kaosv1alpha1.TelemetryConfig{
+				Enabled:  true,
+				Endpoint: "http://collector:4317",
+			},
+			expect: true,
+		},
+		{
+			name: "enabled without endpoint is invalid",
+			tel: &kaosv1alpha1.TelemetryConfig{
+				Enabled:  true,
+				Endpoint: "",
+			},
+			expect: false,
+		},
+	}
+
+	for _, tt := range tests {
+		t.Run(tt.name, func(t *testing.T) {
+			result := IsTelemetryConfigValid(tt.tel)
+			if result != tt.expect {
+				t.Errorf("expected %v, got %v", tt.expect, result)
+			}
+		})
+	}
+}
+
 func TestBuildTelemetryEnvVars(t *testing.T) {
 	// Clear any existing env
 	os.Unsetenv("OTEL_RESOURCE_ATTRIBUTES")
```

---

### Commit: 4102830
**Subject:** fix(operator): remove localhost:4317 default from telemetry endpoint CRD
**Date:** 2026-01-27 07:02:07 +0100
**Author:** Alejandro Saucedo

Regenerated Helm chart CRDs to remove incorrect default for endpoint.
Endpoint is now truly required when telemetry is enabled, matching docs.


```diff
 operator/chart/crds/agent-crd.yaml     |  5 +++--
 operator/chart/crds/mcpserver-crd.yaml |  5 +++--
 operator/chart/crds/modelapi-crd.yaml  | 18 ++++++++++++++++++
 operator/chart/values.yaml             | 34 ----------------------------------
 4 files changed, 24 insertions(+), 38 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/operator/chart/crds/agent-crd.yaml b/operator/chart/crds/agent-crd.yaml
index 14f6408..25f7330 100644
--- a/operator/chart/crds/agent-crd.yaml
+++ b/operator/chart/crds/agent-crd.yaml
@@ -296,8 +296,9 @@ spec:
                           When enabled, traces, metrics, and log correlation are all active.
                         type: boolean
                       endpoint:
-                        default: http://localhost:4317
-                        description: 'Endpoint is the OTLP endpoint URL (default: http://localhost:4317)'
+                        description: |-
+                          Endpoint is the OTLP gRPC endpoint URL (required when enabled).
+                          Example: "http://otel-collector.observability:4317"
                         type: string
                     type: object
                 type: object
diff --git a/operator/chart/crds/mcpserver-crd.yaml b/operator/chart/crds/mcpserver-crd.yaml
index 50b9cb2..d74d22c 100644
--- a/operator/chart/crds/mcpserver-crd.yaml
+++ b/operator/chart/crds/mcpserver-crd.yaml
@@ -224,8 +224,9 @@ spec:
                           When enabled, traces, metrics, and log correlation are all active.
                         type: boolean
                       endpoint:
-                        default: http://localhost:4317
-                        description: 'Endpoint is the OTLP endpoint URL (default: http://localhost:4317)'
+                        description: |-
+                          Endpoint is the OTLP gRPC endpoint URL (required when enabled).
+                          Example: "http://otel-collector.observability:4317"
                         type: string
                     type: object
                   tools:
diff --git a/operator/chart/crds/modelapi-crd.yaml b/operator/chart/crds/modelapi-crd.yaml
index fb7fe4f..f2148a9 100644
--- a/operator/chart/crds/modelapi-crd.yaml
+++ b/operator/chart/crds/modelapi-crd.yaml
@@ -8892,6 +8892,24 @@ spec:
                 required:
                 - models
                 type: object
+              telemetry:
+                description: |-
+                  Telemetry configures OpenTelemetry instrumentation.
+                  For Proxy mode (LiteLLM): Enables OTel callbacks for traces/metrics.
+                  For Hosted mode (Ollama): Not supported; a warning is emitted if enabled.
+                properties:
+                  enabled:
+                    default: false
+                    description: |-
+                      Enabled controls whether OpenTelemetry is enabled (default: false)
+                      When enabled, traces, metrics, and log correlation are all active.
+                    type: boolean
+                  endpoint:
+                    description: |-
+                      Endpoint is the OTLP gRPC endpoint URL (required when enabled).
+                      Example: "http://otel-collector.observability:4317"
+                    type: string
+                type: object
             required:
             - mode
             type: object
diff --git a/operator/chart/values.yaml b/operator/chart/values.yaml
index 64bf387..82be84a 100644
--- a/operator/chart/values.yaml
+++ b/operator/chart/values.yaml
@@ -27,42 +27,8 @@ controllerManager:
   tolerations: []
   topologySpreadConstraints: []
 kubernetesClusterDomain: cluster.local
-
-# Default images for agent runtime and MCP servers
-defaultImages:
-  agentRuntime: axsauze/kaos-agent:latest
-  mcpServer: axsauze/kaos-mcp-server:latest
-  litellm: ghcr.io/berriai/litellm:main-stable
-  ollama: ollama/ollama:latest
-
-# Gateway API configuration
-gatewayAPI:
-  enabled: false
-  createGateway: false
-  gatewayName: kaos-gateway
-  gatewayClassName: ""
-  gatewayNamespace: ""
-  listenerPort: 80
-  listenerProtocol: HTTP
-
-# Gateway timeout settings
-gateway:
-  defaultTimeouts:
-    agent: "300s"
-    modelAPI: "120s"
-    mcp: "60s"
-
 serviceAccount:
   annotations: {}
   automount: true
   create: true
   name: ""
-
-# Global telemetry configuration (applies to all components)
-# Component-level telemetry config overrides these defaults
-telemetry:
-  # enabled controls whether telemetry is enabled by default for all components
-  enabled: false
-  # endpoint is the OTLP endpoint URL (required when enabled)
-  # Example: "http://otel-collector.observability:4317"
-  endpoint: ""
```

---

### Commit: 25f25b8
**Subject:** docs(telemetry): fix metric names and labels to match implementation
**Date:** 2026-01-27 07:03:05 +0100
**Author:** Alejandro Saucedo

Updated docs to reflect actual metric names and labels:
- Metric names: kaos.requests (not kaos.agent.requests)
- Duration unit: milliseconds (not seconds)
- Labels: success=true/false (not status=success/error)
- Labels: agent.name, tool, target (not agent_name, tool_name, etc.)
- Removed mcp_server label (not implemented)


```diff
 docs/operator/telemetry.md | 22 ++++++++++++----------
 1 file changed, 12 insertions(+), 10 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/docs/operator/telemetry.md b/docs/operator/telemetry.md
index d68177c..37d7b24 100644
--- a/docs/operator/telemetry.md
+++ b/docs/operator/telemetry.md
@@ -216,25 +216,27 @@ The following metrics are collected:
 
 | Metric | Type | Description |
 |--------|------|-------------|
-| `kaos.agent.requests` | Counter | Total requests processed |
-| `kaos.agent.request.duration` | Histogram | Request duration in seconds |
+| `kaos.requests` | Counter | Total requests processed |
+| `kaos.request.duration` | Histogram | Request duration in milliseconds |
 | `kaos.model.calls` | Counter | Total model API calls |
-| `kaos.model.duration` | Histogram | Model call duration in seconds |
+| `kaos.model.duration` | Histogram | Model call duration in milliseconds |
 | `kaos.tool.calls` | Counter | Total tool executions |
-| `kaos.tool.duration` | Histogram | Tool execution duration in seconds |
+| `kaos.tool.duration` | Histogram | Tool execution duration in milliseconds |
 | `kaos.delegations` | Counter | Total agent delegations |
-| `kaos.delegation.duration` | Histogram | Delegation duration in seconds |
+| `kaos.delegation.duration` | Histogram | Delegation duration in milliseconds |
 
 All metrics include labels:
-- `agent_name`: Name of the agent
-- `status`: "success" or "error"
+- `agent.name`: Name of the agent
+- `success`: "true" or "false"
+
+Model metrics also include:
+- `model`: Model identifier
 
 Tool metrics also include:
-- `tool_name`: Name of the tool
-- `mcp_server`: Name of the MCP server
+- `tool`: Name of the tool
 
 Delegation metrics also include:
-- `target_agent`: Name of the target agent
+- `target`: Name of the target agent
 
 ## Log Correlation
 
```

---

### Commit: 353052f
**Subject:** fix(telemetry): add proper type annotation for SpanState.token
**Date:** 2026-01-27 07:06:18 +0100
**Author:** Alejandro Saucedo

Changed token field from 'object' to 'Token[Context]' to satisfy type checker.
Added import for Token from contextvars and Context from opentelemetry.context.


```diff
 python/telemetry/manager.py | 5 +++--
 1 file changed, 3 insertions(+), 2 deletions(-)
```

#### Detailed Changes

```diff
diff --git a/python/telemetry/manager.py b/python/telemetry/manager.py
index 14fbfca..543c75c 100644
--- a/python/telemetry/manager.py
+++ b/python/telemetry/manager.py
@@ -15,12 +15,13 @@ Key design:
 import logging
 import os
 import time
-from contextvars import ContextVar
+from contextvars import ContextVar, Token
 from dataclasses import dataclass, field
 from typing import Any, Dict, List, Optional
 
 from pydantic_settings import BaseSettings, SettingsConfigDict
 from opentelemetry import trace, metrics, context as otel_context
+from opentelemetry.context import Context
 from opentelemetry.sdk.trace import TracerProvider
 from opentelemetry.sdk.trace.export import BatchSpanProcessor
 from opentelemetry.sdk.metrics import MeterProvider
@@ -53,7 +54,7 @@ class SpanState:
     """State for an active span on the stack."""
 
     span: Span
-    token: object  # Context token for detaching
+    token: Token[Context]  # Context token for detaching
     start_time: float
     metric_kind: Optional[str] = None  # "request", "model", "tool", "delegation"
     metric_attrs: Dict[str, Any] = field(default_factory=dict)
```

---
