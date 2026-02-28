# Implementation Plan: Modular MCPServer Framework Extension

**Status:** Ready for Implementation  
**Reference:** DESIGN_PROPOSAL.md  
**Created:** 2026-01-31

---

## Overview

This plan implements the Modular MCPServer Framework as specified in DESIGN_PROPOSAL.md. The implementation is structured into numbered tasks that must be executed sequentially, with tests validated and changes committed after each task.

**Core Principle:** KEEP IT SIMPLE - Avoid over-engineering; each commit should be minimal and focused.

---

## Codebase Analysis & Caveats

### Current State Analysis

#### CRD Types (`operator/api/v1alpha1/`)

| File | Current State | Changes Required |
|------|---------------|------------------|
| `mcpserver_types.go` | Has `MCPToolsConfig` with `fromString/fromPackage/fromSecretKeyRef`, `Type` field | Remove `MCPToolsConfig`, replace `Type` with `Runtime`, add `Params`, add `ContainerOverride` |
| `agent_types.go` | Has `Env` in `AgentConfig` (line 102) | Add `ContainerOverride`, remove `Env` from `AgentConfig` |
| `modelapi_types.go` | Has `Env` in `ProxyConfig` (line 92) and `HostedConfig` (line 104) | Add `ContainerOverride`, remove `Env` from both configs |

#### Controllers (`operator/controllers/`)

| File | Current State | Changes Required |
|------|---------------|------------------|
| `mcpserver_controller.go` | Reads `Spec.Config.Env` (line 285), has `constructPythonContainer` with fromString/fromPackage logic (lines 283-383) | Add runtime registry resolution, read from `Spec.Container.Env`, remove old tool loading logic |
| `agent_controller.go` | Reads `Spec.Config.Env` (line 429) | Read from `Spec.Container.Env` |
| `modelapi_controller.go` | Reads `ProxyConfig.Env` (line 476), `HostedConfig.Env` (line 542) | Read from `Spec.Container.Env` |

#### Integration Tests (`operator/controllers/integration/`)

| File | Current State | Tests Using Old Patterns |
|------|---------------|--------------------------|
| `mcpserver_test.go` | Uses `spec.type: python-runtime`, `spec.config.tools.fromString`, `spec.config.env` | Must update all test specs |
| `agent_test.go` | Uses `spec.config.env` | Must update all test specs |
| `modelapi_test.go` | Uses `spec.proxyConfig.env`, `spec.hostedConfig.env` | Must update all test specs |

#### E2E Tests (`operator/tests/e2e/`)

| File | Current Patterns | Changes Required |
|------|------------------|------------------|
| `conftest.py` | `create_mcpserver_resource` uses `type: python-runtime`, `config.tools.fromPackage` (lines 406-421) | Update to use `runtime: custom` pattern |
| `conftest.py` | `create_modelapi_resource` uses `proxyConfig.env` (lines 347-350) | Move env to container.env |
| `conftest.py` | `create_agent_resource` uses `config.env` (lines 443-445) | Move env to container.env |
| All test files | Use conftest helpers | Automatically fixed via conftest updates |

#### Python Framework (`python/`)

| File | Current State | Changes Required |
|------|---------------|------------------|
| `mcptools/server.py` | MCPServer with OTEL, logging (277 lines) | DELETE - replaced by simpler rawpython |
| `mcptools/client.py` | MCPClient for Agent (183 lines) | KEEP - used by Agent |
| `mcptools/__init__.py` | Empty | No changes needed |

#### CLI (`kaos-cli/kaos_cli/`)

| File | Current State | Changes Required |
|------|---------------|------------------|
| `main.py` | Has `install`, `uninstall`, `ui`, `version` commands | Add `system`, `mcp`, `agent`, `modelapi` subcommands |
| `install.py` | Install/uninstall logic | Move to `system/install.py` |

### Key Caveats & Risks

1. **Breaking Changes Everywhere**: All YAML samples in tests, docs, and config/samples use old patterns
2. **Go Generate Required**: After type changes, must run `make generate manifests`
3. **Helm Chart Regeneration**: After controller changes, must run `make helm`
4. **E2E Tests Use Old Patterns**: `conftest.py` helper functions create resources with old schema
5. **No `mcptools/__init__.py` imports**: Can safely delete `server.py` without breaking imports

### Critical Design Issues Identified

#### Issue 1: Transport Compatibility

**Research Findings:**
| Runtime | Transport Support | HTTP Support |
|---------|------------------|--------------|
| **rawpython** (ours) | Streamable HTTP | ✅ Native |
| **kubernetes** (containers/kubernetes-mcp-server) | Streamable HTTP | ✅ `--port` flag |
| **slack** (zencoderai/slack-mcp-server) | stdio AND HTTP | ✅ `--transport http` |
| **discord** (hanweg/mcp-discord) | **stdio only** | ❌ No |
| **fetch** (modelcontextprotocol/servers) | **stdio only** | ❌ No |

**Decision:** 
- **Include rawpython, kubernetes, and slack** in initial implementation (all support HTTP)
- **Defer discord, fetch** until we implement stdio-to-HTTP adapter
- Document deferred runtimes as "coming soon" in the registry

#### Issue 2: Params Env Var Naming

**Contradiction:** DESIGN_PROPOSAL.md uses both `MCP_PARAMS` and `MCP_TOOLS_STRING`.

**Decision:** Use consistent naming:
- `MCP_PARAMS` - Generic params env var for all runtimes (new standard)
- Runtime-specific override via `paramsEnvVar` in registry (for backwards compatibility)
- rawpython uses `MCP_TOOLS_STRING` internally (set via `paramsEnvVar: MCP_TOOLS_STRING`)

#### Issue 3: MCPServerConfig Struct Incomplete

**Current Issue:** Task 1.4 says "Update MCPServerConfig to remove Tools and Env" but doesn't specify what remains.

**Decision:** Remove `MCPServerConfig` entirely. New MCPServerSpec structure:
```go
type MCPServerSpec struct {
    Runtime      string              `json:"runtime"`
    Params       string              `json:"params,omitempty"`
    Telemetry    *TelemetryConfig    `json:"telemetry,omitempty"`
    GatewayRoute *GatewayRoute       `json:"gatewayRoute,omitempty"`
    Container    *ContainerOverride  `json:"container,omitempty"`
    PodSpec      *corev1.PodSpec     `json:"podSpec,omitempty"`
    ServiceAccountName string        `json:"serviceAccountName,omitempty"`  // For RBAC
}
```

#### Issue 4: ServiceAccount Wiring for RBAC

**Problem:** `kaos system create-rbac` creates ServiceAccount, but MCPServer pods can't use it without explicit wiring.

**Solution:** Add `serviceAccountName` field to MCPServerSpec:
```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: custom-tools
spec:
  runtime: custom
  serviceAccountName: mcpserver-custom  # Created by kaos system create-rbac
  container:
    image: myregistry/my-mcp:v1
```

**Controller behavior:**
- If `serviceAccountName` is set, use it in the pod spec
- If not set, use default service account
- Useful for custom runtimes that need Kubernetes API access

---

## Implementation Tasks

### Phase 1: CRD & Controller Updates

#### Task 1.1: Add ContainerOverride Type

**Files to modify:**
- `operator/api/v1alpha1/agent_types.go`

**Changes:**
1. Add `ContainerOverride` struct after `TelemetryConfig`
2. Add `Container *ContainerOverride` field to `AgentSpec`

**Validation:**
```bash
cd operator && make generate manifests && make test-unit
```

**Commit:** `feat(operator): add ContainerOverride type for container-level overrides`

---

#### Task 1.2: Move env from AgentConfig to ContainerOverride

**Files to modify:**
- `operator/api/v1alpha1/agent_types.go`
- `operator/controllers/agent_controller.go`
- `operator/controllers/integration/agent_test.go`

**Changes:**
1. Remove `Env []corev1.EnvVar` from `AgentConfig` struct
2. Update `constructEnvVars` to read from `spec.Container.Env` instead of `spec.Config.Env`
3. Update all test specs to use new pattern

**Validation:**
```bash
cd operator && make generate manifests && make test-unit
```

**Commit:** `refactor(operator): move Agent env from config to container`

---

#### Task 1.3: Add ContainerOverride to ModelAPI and migrate env

**Files to modify:**
- `operator/api/v1alpha1/modelapi_types.go`
- `operator/controllers/modelapi_controller.go`
- `operator/controllers/integration/modelapi_test.go`

**Changes:**
1. Add `Container *ContainerOverride` field to `ModelAPISpec`
2. Remove `Env []corev1.EnvVar` from `ProxyConfig` and `HostedConfig`
3. Update controller to read from `spec.Container.Env`
4. Update all test specs

**Validation:**
```bash
cd operator && make generate manifests && make test-unit
```

**Commit:** `refactor(operator): move ModelAPI env from proxyConfig/hostedConfig to container`

---

#### Task 1.4: Redesign MCPServer CRD with runtime field

**Files to modify:**
- `operator/api/v1alpha1/mcpserver_types.go`

**Changes:**
1. Remove `MCPServerType` const and type (lines 9-16)
2. Remove `MCPToolsConfig` struct entirely (lines 21-36)
3. Remove `MCPServerConfig` struct entirely (lines 40-53)
4. Redesign `MCPServerSpec` to:
   ```go
   type MCPServerSpec struct {
       // Runtime identifier from ConfigMap or "custom"
       // +kubebuilder:validation:Required
       Runtime string `json:"runtime"`
       
       // Params is runtime-specific configuration (YAML string)
       // Passed to container via runtime's paramsEnvVar (e.g., MCP_TOOLS_STRING for rawpython)
       // +kubebuilder:validation:Optional
       Params string `json:"params,omitempty"`
       
       // ServiceAccountName for RBAC (e.g., for kubernetes runtime)
       // Created via `kaos system create-rbac`
       // +kubebuilder:validation:Optional
       ServiceAccountName string `json:"serviceAccountName,omitempty"`
       
       // Telemetry configures OpenTelemetry instrumentation
       // +kubebuilder:validation:Optional
       Telemetry *TelemetryConfig `json:"telemetry,omitempty"`
       
       // GatewayRoute configures Gateway API routing
       // +kubebuilder:validation:Optional
       GatewayRoute *GatewayRoute `json:"gatewayRoute,omitempty"`
       
       // Container provides shorthand container overrides
       // +kubebuilder:validation:Optional
       Container *ContainerOverride `json:"container,omitempty"`
       
       // PodSpec allows full pod spec override
       // +kubebuilder:validation:Optional
       PodSpec *corev1.PodSpec `json:"podSpec,omitempty"`
   }
   ```
5. Update print columns to show `Runtime` instead of `Type`

**Validation:**
```bash
cd operator && make generate manifests
```

**Commit:** `feat(operator): redesign MCPServer CRD with runtime field`

---

#### Task 1.5: Add runtime registry ConfigMap to Helm chart

**Files to create/modify:**
- `operator/chart/templates/runtime-configmap.yaml` (NEW)
- `operator/chart/values.yaml`

**Changes:**
1. Create runtime registry ConfigMap with HTTP-compatible runtimes:
   ```yaml
   runtimes:
     rawpython:
       type: python
       image: {{ .Values.defaultImages.mcpRawPython }}
       description: "Execute Python code strings as MCP tools"
       paramsEnvVar: MCP_TOOLS_STRING
       
     kubernetes:
       type: go
       image: {{ .Values.defaultImages.mcpKubernetes }}
       description: "Kubernetes API operations"
       args: ["--port", "8000"]
       # Requires serviceAccountName in MCPServer spec
       
     slack:
       type: nodejs
       image: {{ .Values.defaultImages.mcpSlack }}
       description: "Slack messaging integration"
       command: ["--transport", "http", "--port", "8000"]
       requiredEnv:
       - SLACK_BOT_TOKEN
       - SLACK_TEAM_ID
       
     # discord, fetch deferred - require stdio-to-HTTP adapter
   ```
2. Add `defaultImages.mcpRawPython`, `mcpKubernetes`, `mcpSlack` to values.yaml
3. Document that discord/fetch require future stdio adapter

**Validation:**
```bash
cd operator && make helm && helm template chart
```

**Commit:** `feat(helm): add runtime registry ConfigMap`

---

#### Task 1.6: Implement runtime resolution in MCPServer controller

**Files to modify:**
- `operator/controllers/mcpserver_controller.go`

**Changes:**
1. Add `RuntimeConfig` struct and `RuntimeRegistry` struct:
   ```go
   type RuntimeConfig struct {
       Type         string   `yaml:"type"`          // python, nodejs, go
       Image        string   `yaml:"image"`
       Description  string   `yaml:"description"`
       Command      []string `yaml:"command,omitempty"`
       Args         []string `yaml:"args,omitempty"`
       ParamsEnvVar string   `yaml:"paramsEnvVar,omitempty"` // e.g., MCP_TOOLS_STRING
       RequiredEnv  []string `yaml:"requiredEnv,omitempty"`
   }
   ```
2. Add `resolveRuntime` function to read from ConfigMap
3. Replace `constructPythonContainer` with `constructContainerFromRuntime`
4. Handle `custom` runtime (requires container.image)
5. Handle `serviceAccountName` - set in pod spec if provided:
   ```go
   if mcpserver.Spec.ServiceAccountName != "" {
       basePodSpec.ServiceAccountName = mcpserver.Spec.ServiceAccountName
   }
   ```
6. Pass `params` to container via runtime's `paramsEnvVar`:
   ```go
   if mcpserver.Spec.Params != "" && runtime.ParamsEnvVar != "" {
       env = append(env, corev1.EnvVar{
           Name:  runtime.ParamsEnvVar,
           Value: mcpserver.Spec.Params,
       })
   }
   ```
7. Add `SystemNamespace` field to reconciler for ConfigMap lookup
8. Read env from `spec.Container.Env`
9. Warn if kubernetes runtime used without serviceAccountName

**Validation:**
```bash
cd operator && make test-unit
```

**Commit:** `feat(operator): implement runtime registry resolution for MCPServer`

---

#### Task 1.7: Update MCPServer integration tests

**Files to modify:**
- `operator/controllers/integration/mcpserver_test.go`

**Changes:**
1. Update all test specs to use new `runtime` field instead of `type`
2. Remove tests for `fromString`, `fromPackage`, `fromSecretKeyRef`
3. Add tests for `runtime: rawpython` with params
4. Add tests for `runtime: kubernetes` with serviceAccountName
5. Add tests for `runtime: custom` with container.image
6. Add tests for `runtime: slack` with container.env
7. Move env to `spec.container.env`
8. Add test for `serviceAccountName` wiring

**Validation:**
```bash
cd operator && make test-unit
```

**Commit:** `test(operator): update MCPServer tests for runtime-based CRD`

---

#### Task 1.8: Create mcpserver_runtime_test.go for runtime registry tests

**Files to create:**
- `operator/controllers/integration/mcpserver_runtime_test.go`

**Changes:**
1. Add tests for runtime registry ConfigMap parsing
2. Add tests for unknown runtime error handling
3. Add tests for custom runtime validation (missing image error)
4. Add tests for params env var injection
5. Add tests for serviceAccountName propagation
6. Add tests for kubernetes runtime with serviceAccountName

**Validation:**
```bash
cd operator && make test-unit
```

**Commit:** `test(operator): add runtime registry integration tests`

---

### Phase 2: Project Restructure

#### Task 2.1: Rename python/ to data-plane/kaos-framework/

**Files to modify:**
- Move entire `python/` directory to `data-plane/kaos-framework/`
- `.github/workflows/*.yaml` - update paths
- `.github/instructions/python.instructions.md` - update paths
- `operator/Makefile` - update any python references

**Validation:**
```bash
cd data-plane/kaos-framework && make lint && python -m pytest tests/ -v
```

**Commit:** `refactor: rename python/ to data-plane/kaos-framework/`

---

#### Task 2.2: Remove mcptools/server.py

**Files to delete:**
- `data-plane/kaos-framework/mcptools/server.py`

**Files to modify:**
- `data-plane/kaos-framework/tests/` - remove any server.py tests

**Validation:**
```bash
cd data-plane/kaos-framework && make lint && python -m pytest tests/ -v
```

**Commit:** `refactor(framework): remove mcptools/server.py (replaced by standalone rawpython)`

---

#### Task 2.3: Create data-plane/mcp-servers/rawpython/

**Files to create:**
- `data-plane/mcp-servers/rawpython/server.py`
- `data-plane/mcp-servers/rawpython/pyproject.toml`
- `data-plane/mcp-servers/rawpython/Dockerfile`
- `data-plane/mcp-servers/rawpython/tests/test_rawpython.py`

**server.py content:**
```python
import os
from types import FunctionType
from fastmcp import FastMCP

mcp = FastMCP("RawPython MCP Server")

tools_string = os.getenv("MCP_TOOLS_STRING", "")
if tools_string:
    namespace = {}
    exec(tools_string, {}, namespace)
    for name, func in namespace.items():
        if isinstance(func, FunctionType):
            mcp.tool(name)(func)
```

**Validation:**
```bash
cd data-plane/mcp-servers/rawpython && pip install -e . && python -m pytest tests/ -v
```

**Commit:** `feat(mcp): create standalone rawpython MCP server`

---

#### Task 2.4: Create build-mcp-rawpython.yaml workflow

**Files to create:**
- `.github/workflows/build-mcp-rawpython.yaml`

**Changes:**
1. Trigger only on `data-plane/mcp-servers/rawpython/**` changes
2. Build and push `axsauze/kaos-mcp-rawpython` image

**Validation:**
- Check workflow YAML syntax

**Commit:** `ci: add workflow for rawpython MCP server image`

---

#### Task 2.5: Update CI workflows for new paths

**Files to modify:**
- `.github/workflows/python-tests.yaml` - update to `data-plane/kaos-framework/`
- `.github/workflows/reusable-build-images.yaml` - update paths
- `.github/instructions/python.instructions.md` - update references

**Validation:**
- Check workflow YAML syntax

**Commit:** `ci: update workflows for data-plane/ restructure`

---

### Phase 3: CLI System Commands

#### Task 3.1: Create kaos system subcommand structure

**Files to create:**
- `kaos-cli/kaos_cli/system/__init__.py`
- `kaos-cli/kaos_cli/system/install.py`

**Files to modify:**
- `kaos-cli/kaos_cli/main.py` - add system subcommand, move install/uninstall

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos system --help
```

**Commit:** `feat(cli): add kaos system subcommand with install/uninstall`

---

#### Task 3.2: Implement kaos system create-rbac

**Files to create:**
- `kaos-cli/kaos_cli/system/create_rbac.py`

**Changes:**
1. Add `--name`, `--namespace`, `--namespaces`, `--resources`, `--verbs` flags
2. Add `--read-only` shorthand
3. Add `--cluster-wide` for ClusterRole/ClusterRoleBinding
4. Generate ServiceAccount, Role/ClusterRole, RoleBinding/ClusterRoleBinding YAML
5. Apply via kubectl

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos system create-rbac --help
```

**Commit:** `feat(cli): add kaos system create-rbac for MCPServer RBAC`

---

#### Task 3.3: Implement kaos system status and runtimes

**Files to create:**
- `kaos-cli/kaos_cli/system/status.py`
- `kaos-cli/kaos_cli/system/runtimes.py`

**Changes:**
1. `status` - show operator deployment status, gateway status, version
2. `runtimes` - list runtimes from ConfigMap

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos system status --help && kaos system runtimes --help
```

**Commit:** `feat(cli): add kaos system status and runtimes commands`

---

### Phase 4: CLI Resource Commands

#### Task 4.1: Create kaos mcp CRUD commands (kubectl wrappers)

**Files to create:**
- `kaos-cli/kaos_cli/mcp/__init__.py`
- `kaos-cli/kaos_cli/mcp/crud.py`

**Changes:**
1. `kaos mcp list` -> `kubectl get mcpservers`
2. `kaos mcp get <name>` -> `kubectl get mcpserver <name> -o yaml`
3. `kaos mcp logs <name>` -> `kubectl logs -l mcpserver=<name>`
4. `kaos mcp delete <name>` -> `kubectl delete mcpserver <name>`

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos mcp list --help
```

**Commit:** `feat(cli): add kaos mcp list/get/logs/delete commands`

---

#### Task 4.2: Implement kaos mcp invoke

**Files to create:**
- `kaos-cli/kaos_cli/mcp/invoke.py`

**Changes:**
1. Add `--tool` and `--args` flags
2. Port-forward to MCPServer service
3. Call MCP tool via HTTP
4. Output result

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos mcp invoke --help
```

**Commit:** `feat(cli): add kaos mcp invoke command`

---

#### Task 4.3: Create kaos agent CRUD and invoke commands

**Files to create:**
- `kaos-cli/kaos_cli/agent/__init__.py`
- `kaos-cli/kaos_cli/agent/crud.py`
- `kaos-cli/kaos_cli/agent/deploy.py`
- `kaos-cli/kaos_cli/agent/invoke.py`

**Changes:**
1. Same CRUD pattern as mcp
2. `deploy` - apply YAML file
3. `invoke` - send chat completion request

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos agent --help
```

**Commit:** `feat(cli): add kaos agent deploy/list/get/logs/delete/invoke commands`

---

#### Task 4.4: Create kaos modelapi CRUD and invoke commands

**Files to create:**
- `kaos-cli/kaos_cli/modelapi/__init__.py`
- `kaos-cli/kaos_cli/modelapi/crud.py`
- `kaos-cli/kaos_cli/modelapi/deploy.py`
- `kaos-cli/kaos_cli/modelapi/invoke.py`

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos modelapi --help
```

**Commit:** `feat(cli): add kaos modelapi deploy/list/get/logs/delete/invoke commands`

---

### Phase 5: CLI Build Workflow

#### Task 5.1: Implement kaos mcp init

**Files to create:**
- `kaos-cli/kaos_cli/mcp/init.py`
- `kaos-cli/kaos_cli/utils/templates/fastmcp/server.py.tmpl`
- `kaos-cli/kaos_cli/utils/templates/fastmcp/pyproject.toml.tmpl`

**Changes:**
1. `--name` flag (required)
2. `--create-dockerfile` flag (optional)
3. `--create-yaml` flag (optional)
4. Generate server.py and pyproject.toml templates

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos mcp init --name test-mcp && ls test-mcp/
```

**Commit:** `feat(cli): add kaos mcp init command`

---

#### Task 5.2: Implement kaos mcp build

**Files to create:**
- `kaos-cli/kaos_cli/mcp/build.py`
- `kaos-cli/kaos_cli/utils/docker.py`

**Changes:**
1. `--name`, `--tag`, `--registry` flags
2. `--path` for FastMCP project path
3. `--fastmcp-path` for FastMCP object (default: server:mcp)
4. `--push` to push to registry
5. `--kind-load` to load into KIND
6. `--create-dockerfile` to save generated Dockerfile
7. Auto-generate Dockerfile if not exists

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos mcp build --help
```

**Commit:** `feat(cli): add kaos mcp build command`

---

#### Task 5.3: Implement kaos mcp deploy

**Files to create:**
- `kaos-cli/kaos_cli/mcp/deploy.py`

**Changes:**
1. `--name`, `--namespace` flags
2. `--runtime` flag (default: custom)
3. `--image` flag (required for custom)
4. `--file` for MCPServer YAML
5. `--create-yaml` to save generated YAML
6. `--wait` to wait for ready
7. `--dry-run` to show YAML without applying

**Validation:**
```bash
cd kaos-cli && source .venv/bin/activate && kaos mcp deploy --help
```

**Commit:** `feat(cli): add kaos mcp deploy command`

---

### Phase 6: E2E Test Updates

#### Task 6.1: Update E2E conftest.py helpers

**Files to modify:**
- `operator/tests/e2e/conftest.py`

**Changes:**
1. Update `create_mcpserver_resource` to use `runtime: custom` + `container.image`
2. Update `create_modelapi_resource` to use `container.env` instead of `proxyConfig.env`
3. Update `create_modelapi_hosted_resource` to use `container.env`
4. Update `create_agent_resource` to use `container.env` instead of `config.env`

**Validation:**
```bash
cd operator/tests && source .venv/bin/activate && python -m pytest e2e/ -v -k "test_" --collect-only
```

**Commit:** `test(e2e): update conftest helpers for new CRD structure`

---

#### Task 6.2: Update all E2E test files

**Files to modify:**
- `operator/tests/e2e/test_base_func_e2e.py`
- `operator/tests/e2e/test_mcp_tools_e2e.py`
- `operator/tests/e2e/test_modelapi_e2e.py`
- `operator/tests/e2e/test_agentic_loop_e2e.py`
- `operator/tests/e2e/test_multi_agent_e2e.py`

**Changes:**
1. Update any inline YAML specs to use new schema
2. Ensure all tests use conftest helpers

**Validation:**
Run E2E tests in CI (push PR)

**Commit:** `test(e2e): update all E2E tests for new CRD schema`

---

#### Task 6.3: Add test_mcp_runtimes_e2e.py

**Files to create:**
- `operator/tests/e2e/test_mcp_runtimes_e2e.py`

**Changes:**
1. Test `runtime: rawpython` with params
2. Test `runtime: kubernetes` with serviceAccountName (read-only operations)
3. Test `runtime: custom` with custom image
4. Skip slack tests (require credentials via secrets)

**Note:** discord/fetch runtimes are deferred (stdio-only transport).

**Validation:**
Run E2E tests in CI (push PR)

**Commit:** `test(e2e): add runtime integration E2E tests`

---

### Phase 7: Documentation & Samples

This phase significantly expands documentation to support the modular MCPServer framework.

#### Task 7.1: Restructure docs for modularity

**Files to create:**
- `docs/operator/mcpserver/index.md` (overview, links to sub-pages)
- `docs/operator/mcpserver/runtime-registry.md` (how runtimes work)
- `docs/operator/mcpserver/built-in-runtimes.md` (rawpython, kubernetes, slack)
- `docs/operator/mcpserver/custom-runtimes.md` (building your own)
- `docs/operator/mcpserver/rbac-setup.md` (serviceAccountName, permissions)

**Files to modify:**
- `docs/operator/mcpserver-crd.md` → redirect to `mcpserver/index.md`
- `docs/.vitepress/config.ts` → update sidebar for new structure

**Changes:**
1. Create MCPServer documentation hub with sub-pages
2. Document each built-in runtime with examples
3. Document custom runtime workflow (init → build → deploy)
4. Document RBAC setup for kubernetes runtime

**Commit:** `docs: restructure MCPServer documentation for modularity`

---

#### Task 7.2: Update Agent and ModelAPI CRD documentation

**Files to modify:**
- `docs/operator/agent-crd.md`
- `docs/operator/modelapi-crd.md`
- `docs/operator/overview.md`

**Changes:**
1. Update all YAML examples to use `container.env` instead of `config.env`
2. Document ContainerOverride fields
3. Add migration note (breaking change from alpha)
2. Document `runtime` field and runtime registry
3. Document `container` field and `env` location
4. Document ContainerOverride fields

**Commit:** `docs: update Agent and ModelAPI CRD documentation`

---

#### Task 7.3: Update sample YAML files

**Files to modify:**
- `operator/config/samples/kaos_v1alpha1_mcpserver.yaml`
- `operator/config/samples/kaos_v1alpha1_agent.yaml`
- `operator/config/samples/kaos_v1alpha1_modelapi.yaml`

**Files to create:**
- `operator/config/samples/kaos_v1alpha1_mcpserver_kubernetes.yaml`
- `operator/config/samples/kaos_v1alpha1_mcpserver_slack.yaml`
- `operator/config/samples/kaos_v1alpha1_mcpserver_custom.yaml`

**Changes:**
1. Update existing samples to new schema
2. Add samples for each built-in runtime
3. Add sample with serviceAccountName for kubernetes runtime

**Commit:** `docs: update sample YAML files for v2 schema`

---

#### Task 7.4: Create CLI reference documentation

**Files to create:**
- `docs/cli/index.md` (CLI overview and installation)
- `docs/cli/system.md` (kaos system commands)
- `docs/cli/mcp.md` (kaos mcp commands - init, build, deploy, list, logs, invoke)
- `docs/cli/agent.md` (kaos agent commands)
- `docs/cli/modelapi.md` (kaos modelapi commands)

**Files to modify:**
- `docs/.vitepress/config.ts` → add CLI section to sidebar

**Changes:**
1. Document all CLI commands with examples
2. Document init → build → deploy workflow
3. Document invoke command for each resource type
4. Add quickstart examples

**Commit:** `docs: add comprehensive CLI reference documentation`

---

#### Task 7.5: Update copilot instructions and project structure docs

**Files to modify:**
- `.github/copilot-instructions.md`
- `.github/instructions/python.instructions.md` → rename to `kaos-framework.instructions.md`
- `.github/instructions/e2e.instructions.md`
- `.github/instructions/operator.instructions.md`

**Changes:**
1. Update paths for data-plane/ restructure
2. Document new CLI commands
3. Update CRD references
4. Document runtime registry
5. Add mcp-servers development instructions

**Commit:** `docs: update copilot instructions for new structure`

---

### Phase 8: Integration Tests Split

#### Task 8.1: Split agent_test.go

**Files to create:**
- `operator/controllers/integration/agent_network_test.go`

**Changes:**
1. Move A2A delegation tests from agent_test.go to agent_network_test.go
2. Keep basic agent tests in agent_test.go

**Validation:**
```bash
cd operator && make test-unit
```

**Commit:** `test(operator): split agent tests for A2A delegation`

---

#### Task 8.2: Split modelapi_test.go

**Files to create:**
- `operator/controllers/integration/modelapi_types_test.go`

**Changes:**
1. Move Proxy vs Hosted tests to modelapi_types_test.go
2. Keep basic modelapi tests in modelapi_test.go

**Validation:**
```bash
cd operator && make test-unit
```

**Commit:** `test(operator): split modelapi tests by mode type`

---

### Phase 9: Final Polish

#### Task 9.1: Update Helm chart with all runtime images

**Files to modify:**
- `operator/chart/values.yaml`

**Changes:**
1. Add all ecosystem images with pinned versions
2. Verify runtime ConfigMap renders correctly

**Validation:**
```bash
cd operator && make helm && helm template chart
```

**Commit:** `feat(helm): add ecosystem MCP server images to values`

---

#### Task 9.2: Run full E2E test suite and fix issues

**Validation:**
Push PR and run full CI pipeline

**Commit:** `fix: address E2E test failures` (if needed)

---

## Task Checklist

### Phase 1: CRD & Controller Updates
- [ ] 1.1: Add ContainerOverride Type
- [ ] 1.2: Move env from AgentConfig to ContainerOverride
- [ ] 1.3: Add ContainerOverride to ModelAPI and migrate env
- [ ] 1.4: Redesign MCPServer CRD with runtime field
- [ ] 1.5: Add runtime registry ConfigMap to Helm chart
- [ ] 1.6: Implement runtime resolution in MCPServer controller
- [ ] 1.7: Update MCPServer integration tests
- [ ] 1.8: Create mcpserver_runtime_test.go for runtime registry tests

### Phase 2: Project Restructure
- [ ] 2.1: Rename python/ to data-plane/kaos-framework/
- [ ] 2.2: Remove mcptools/server.py
- [ ] 2.3: Create data-plane/mcp-servers/rawpython/
- [ ] 2.4: Create build-mcp-rawpython.yaml workflow
- [ ] 2.5: Update CI workflows for new paths

### Phase 3: CLI System Commands
- [ ] 3.1: Create kaos system subcommand structure
- [ ] 3.2: Implement kaos system create-rbac
- [ ] 3.3: Implement kaos system status and runtimes

### Phase 4: CLI Resource Commands
- [ ] 4.1: Create kaos mcp CRUD commands (kubectl wrappers)
- [ ] 4.2: Implement kaos mcp invoke
- [ ] 4.3: Create kaos agent CRUD and invoke commands
- [ ] 4.4: Create kaos modelapi CRUD and invoke commands

### Phase 5: CLI Build Workflow
- [ ] 5.1: Implement kaos mcp init
- [ ] 5.2: Implement kaos mcp build
- [ ] 5.3: Implement kaos mcp deploy

### Phase 6: E2E Test Updates
- [ ] 6.1: Update E2E conftest.py helpers
- [ ] 6.2: Update all E2E test files
- [ ] 6.3: Add test_mcp_runtimes_e2e.py

### Phase 7: Documentation & Samples
- [ ] 7.1: Restructure docs for modularity
- [ ] 7.2: Update Agent and ModelAPI CRD documentation
- [ ] 7.3: Update sample YAML files
- [ ] 7.4: Create CLI reference documentation
- [ ] 7.5: Update copilot instructions and project structure docs

### Phase 8: Integration Tests Split
- [ ] 8.1: Split agent_test.go
- [ ] 8.2: Split modelapi_test.go

### Phase 9: Final Polish
- [ ] 9.1: Update Helm chart with all runtime images
- [ ] 9.2: Run full E2E test suite and fix issues

---

## Commit Guidelines

Each task should be committed with a conventional commit message:

```
<type>(<scope>): <description>

<body - optional>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `ci`

Scopes: `operator`, `cli`, `framework`, `helm`, `e2e`, `mcp`

---

## Validation Commands Summary

```bash
# Go operator
cd operator && make generate manifests && make test-unit

# Helm chart
cd operator && make helm && helm template chart

# Python framework
cd data-plane/kaos-framework && make lint && python -m pytest tests/ -v

# CLI
cd kaos-cli && source .venv/bin/activate && kaos --help

# E2E (via PR)
git push && # check CI
```

---

*End of Implementation Plan*
