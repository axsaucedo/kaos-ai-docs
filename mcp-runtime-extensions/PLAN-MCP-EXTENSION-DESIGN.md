# Design Proposal: Modular MCPServer Framework for KAOS

**Status:** Draft  
**Author:** KAOS Team  
**Created:** 2026-01-31  

---

## Executive Summary

This proposal outlines a redesign of the KAOS MCPServer subsystem to support:
1. **Runtime Registry** - ConfigMap-based registry for reusable MCP server runtimes (Python, Node.js, Go)
2. **CLI Workflow** - `kaos mcp/agent/modelapi` commands for deploy, list, logs, invoke
3. **System Commands** - `kaos system` for RBAC setup, install, and cluster configuration
4. **Built-in MCP Servers** - RawPython, Kubernetes, Slack (all HTTP-compatible)
5. **Consistent CRD Pattern** - `container` with `env` moved from config to container across all CRDs
6. **Comprehensive Documentation** - Modular docs structure for MCPServer and CLI

**Core Principle: KEEP IT SIMPLE** - Avoid over-engineering; sacrifice features for simplicity when needed.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [CRD Redesign](#3-crd-redesign)
4. [Runtime Registry Architecture](#4-runtime-registry-architecture)
5. [Built-in MCP Servers & Ecosystem Analysis](#5-built-in-mcp-servers--ecosystem-analysis)
6. [CLI Design](#6-cli-design)
7. [Project Structure](#7-project-structure)
8. [Build & Release Integration](#8-build--release-integration)
9. [Testing Strategy](#9-testing-strategy)
10. [Refactoring Scope](#10-refactoring-scope)
11. [Documentation Structure](#11-documentation-structure)
12. [Decisions](#12-decisions)
13. [Future Enhancements](#13-future-enhancements)

---

## 1. Problem Statement

### Current State

The existing MCPServer CRD supports three tool loading methods:
- `fromPackage`: Install PyPI package and run
- `fromString`: Dynamic Python code via `exec()`
- `fromSecretKeyRef`: Load tools from Kubernetes Secret

**Limitations:**
1. No reusable runtime pattern
2. No CLI workflow for building/deploying custom MCPServers
3. No Kubernetes or messaging channel tools built-in
4. Inconsistent CLI experience across resource types
5. `env` is scattered across different config locations (config.env, proxyConfig.env, etc.)

---

## 2. Goals & Non-Goals

### Goals

| Goal | Priority |
|------|----------|
| Introduce runtime registry for reusable MCPServer types | P0 |
| Support both Python and Node.js runtimes (ecosystem servers use both) | P0 |
| Provide `kaos mcp init/build/deploy` CLI workflow | P0 |
| Provide `kaos system` commands for RBAC and cluster setup | P0 |
| Ship rawpython, kubernetes, and slack runtimes (HTTP transport support) | P0 |
| Move `env` to `container.env` across all CRDs (breaking change) | P0 |
| Remove `tools` list from MCPServer spec (simplify - not used currently) | P0 |
| Add `serviceAccountName` to MCPServer for RBAC wiring | P0 |
| Extend CLI pattern to agents/modelapis (deploy, list, logs, invoke) | P0 |
| Consistent `container` shorthand across all CRDs | P1 |

### Non-Goals

- Backwards compatibility (alpha repo - breaking changes OK)
- Migration tooling
- Schema validation for runtime params
- Auto-versioning of runtimes
- stdio-only runtimes (discord, fetch - deferred until adapter exists)

### Transport Compatibility Requirement

**Critical:** KAOS uses streamable-http (HTTP/JSON-RPC) to communicate with MCP servers. Runtimes that only support stdio transport cannot be used directly and require a stdio-to-HTTP adapter.

| Runtime | Transport | Status |
|---------|-----------|--------|
| rawpython | HTTP ✅ | Included |
| kubernetes | HTTP ✅ (`--port` flag) | Included |
| slack | HTTP ✅ (`--transport http`) | Included |
| discord | stdio ❌ | Deferred |
| fetch | stdio ❌ | Deferred |

**Deferred stdio runtimes** may be supported in future via:
1. Building a stdio-to-HTTP adapter container
2. Contributing HTTP support upstream to those projects

---

## 3. CRD Redesign

### 3.1 MCPServer CRD v2

**Key Simplification:** Remove `tools` list entirely. Currently the Agent does not use tool information from the MCPServer spec - it discovers tools dynamically via the MCP protocol. This simplifies the CRD significantly.

**Key Addition:** Add `serviceAccountName` for RBAC wiring (required for kubernetes runtime).

```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: my-mcp
  namespace: default
spec:
  # Required: Runtime identifier from ConfigMap or "custom"
  runtime: rawpython  # or: slack, custom (kubernetes/discord/fetch deferred - require stdio adapter)

  # Runtime-specific parameters (YAML string)
  # Passed to container via runtime's paramsEnvVar (e.g., MCP_TOOLS_STRING for rawpython)
  params: |
    code: |
      def echo(text: str) -> str:
          return text

  # ServiceAccount for RBAC (created via `kaos system create-rbac`)
  # Required for kubernetes runtime, optional for others
  serviceAccountName: mcpserver-k8s-tools

  # Telemetry configuration
  telemetry:
    enabled: true
    endpoint: "http://otel-collector:4317"

  # Gateway routing
  gatewayRoute:
    timeout: "60s"

  # Container-level overrides (works with ANY runtime)
  # NOTE: env moved here from config.env
  container:
    image: myregistry/my-mcp:v1.0.0  # Override image (required for custom)
    env:                              # All env vars now here
    - name: LOG_LEVEL
      value: "INFO"
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: api-key
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
    command: ["python", "-m", "my_server"]
    args: ["--port", "8000"]

  # Full PodSpec override for advanced use cases (sidecars, volumes, etc.)
  podSpec:
    containers:
    - name: sidecar
      image: datadog/agent:latest
```

### 3.2 Breaking Changes from Current CRD

The following fields are **removed** (no migration path - alpha repo):

| Removed Field | Replacement |
|---------------|-------------|
| `spec.type` | `spec.runtime` |
| `spec.config.tools.fromString` | `spec.runtime: rawpython` + `spec.params` |
| `spec.config.tools.fromPackage` | `spec.runtime: custom` + `spec.container.image` |
| `spec.config.tools.fromSecretKeyRef` | Use `spec.container.env` with `secretKeyRef` |
| `spec.config.env` | `spec.container.env` |
| `spec.config.tools` (list) | **Removed entirely** - tools discovered via MCP protocol |
| `spec.config` (struct) | **Removed entirely** - telemetry moved to top-level |

### 3.3 Agent CRD Update

Move `env` from `config.env` to `container.env`:

```yaml
apiVersion: kaos.tools/v1alpha1
kind: Agent
metadata:
  name: my-agent
spec:
  modelAPI: my-modelapi
  model: gpt-4
  mcpServers: [my-mcp]
  
  config:
    description: "My agent"
    instructions: "Be helpful"
    # NOTE: env REMOVED from here
  
  # Container-level overrides
  container:
    env:                              # Moved from config.env
    - name: DEBUG_MOCK_RESPONSES
      value: "[]"
    resources:
      requests:
        memory: "256Mi"

  # Full PodSpec override (for sidecars, volumes, etc.)
  podSpec:
    containers:
    - name: sidecar
      image: some-sidecar:latest
```

**Breaking Changes:**
| Removed Field | Replacement |
|---------------|-------------|
| `spec.config.env` | `spec.container.env` |

### 3.4 ModelAPI CRD Update

Move `env` from `proxyConfig.env` and `hostedConfig.env` to `container.env`:

```yaml
apiVersion: kaos.tools/v1alpha1
kind: ModelAPI
metadata:
  name: my-modelapi
spec:
  type: proxy
  proxyConfig:
    models: [gpt-4, gpt-3.5-turbo]
    # NOTE: env REMOVED from here
  
  # Container-level overrides
  container:
    env:                              # Moved from proxyConfig.env / hostedConfig.env
    - name: LITELLM_MASTER_KEY
      valueFrom:
        secretKeyRef:
          name: api-keys
          key: litellm
    resources:
      limits:
        memory: "512Mi"

  # Full PodSpec override
  podSpec: {}
```

**Breaking Changes:**
| Removed Field | Replacement |
|---------------|-------------|
| `spec.proxyConfig.env` | `spec.container.env` |
| `spec.hostedConfig.env` | `spec.container.env` |

### 3.5 ContainerOverride Type Definition

```go
// ContainerOverride provides shorthand container configuration
// Applied as strategic merge patch to the generated container
type ContainerOverride struct {
    // Image overrides the container image
    // +kubebuilder:validation:Optional
    Image string `json:"image,omitempty"`
    
    // Command overrides the container entrypoint
    // +kubebuilder:validation:Optional
    Command []string `json:"command,omitempty"`
    
    // Args overrides the container arguments
    // +kubebuilder:validation:Optional
    Args []string `json:"args,omitempty"`
    
    // Resources overrides compute resources
    // +kubebuilder:validation:Optional
    Resources *corev1.ResourceRequirements `json:"resources,omitempty"`
    
    // Env sets environment variables (replaces config.env)
    // +kubebuilder:validation:Optional
    Env []corev1.EnvVar `json:"env,omitempty"`
}
```

---

## 4. Runtime Registry Architecture

### 4.1 ConfigMap-Based Registry

Runtimes are defined in a ConfigMap that supports both system and user-defined runtimes.
Supports both **Python** and **Node.js** runtimes (ecosystem servers use both):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kaos-mcp-runtimes
  namespace: kaos-system
data:
  runtimes.yaml: |
    runtimes:
      # === SYSTEM RUNTIMES (built by KAOS) ===
      
      rawpython:
        type: python
        image: axsauze/kaos-mcp-rawpython:v0.2.0
        description: "Execute Python code strings as MCP tools"
        command: ["python", "-m", "mcptools.server"]
        paramsEnvVar: MCP_TOOLS_STRING   # Runtime-specific env var for params
        transport: http                   # Supports streamable-http (required)
        
      # === ECOSYSTEM RUNTIMES (external images) ===
      
      kubernetes:
        type: go
        image: ghcr.io/manusa/kubernetes-mcp-server:latest
        description: "Kubernetes API operations (from manusa/kubernetes-mcp-server)"
        args: ["--port", "8000"]          # HTTP mode via --port flag
        transport: http
        # Requires serviceAccountName in MCPServer spec for RBAC
        
      slack:
        type: nodejs
        image: zencoderai/slack-mcp:latest
        description: "Slack messaging integration (from zencoderai/slack-mcp-server)"
        command: ["--transport", "http"]  # Supports --transport http flag
        transport: http
        requiredEnv:
        - SLACK_BOT_TOKEN
        - SLACK_TEAM_ID
      
      # === DEFERRED RUNTIMES (stdio only - require adapter) ===
      # These runtimes only support stdio transport, not HTTP.
      # A stdio-to-http adapter is needed before they can work with KAOS.
      # Deferred to future implementation.
      #
      # discord:
      #   type: python
      #   image: ghcr.io/hanweg/mcp-discord:latest
      #   transport: stdio  # uv-based, no HTTP support
      #
      # fetch:
      #   type: python
      #   image: mcp/fetch:latest
      #   transport: stdio  # No HTTP support
        
      # === USER-DEFINED RUNTIMES ===
      # Users can add their own runtimes here via Helm values or kubectl edit
      # Note: All runtimes MUST support transport: http (streamable-http/JSON-RPC)
```

### 4.2 Runtime Types

Support Python, Node.js, and Go runtimes:

```go
// RuntimeType indicates the language/platform of the runtime
type RuntimeType string

const (
    RuntimeTypePython RuntimeType = "python"
    RuntimeTypeNodeJS RuntimeType = "nodejs"
    RuntimeTypeGo     RuntimeType = "go"
)
```

### 4.3 Dynamic Runtime Resolution

The controller dynamically resolves runtimes from the ConfigMap:

```go
func (r *MCPServerReconciler) resolveRuntime(ctx context.Context, mcpserver *kaosv1alpha1.MCPServer) (*RuntimeConfig, error) {
    runtimeName := mcpserver.Spec.Runtime
    
    if runtimeName == "custom" {
        // Custom runtime - container spec must provide image
        if mcpserver.Spec.Container == nil || mcpserver.Spec.Container.Image == "" {
            return nil, fmt.Errorf("custom runtime requires container.image")
        }
        return &RuntimeConfig{
            Image:   mcpserver.Spec.Container.Image,
            Command: mcpserver.Spec.Container.Command,
        }, nil
    }
    
    // Look up in ConfigMap
    runtimes := &corev1.ConfigMap{}
    if err := r.Get(ctx, types.NamespacedName{
        Name:      "kaos-mcp-runtimes",
        Namespace: r.SystemNamespace,
    }, runtimes); err != nil {
        return nil, fmt.Errorf("failed to get runtime registry: %w", err)
    }
    
    var registry RuntimeRegistry
    if err := yaml.Unmarshal([]byte(runtimes.Data["runtimes.yaml"]), &registry); err != nil {
        return nil, fmt.Errorf("failed to parse runtime registry: %w", err)
    }
    
    runtimeDef, ok := registry.Runtimes[runtimeName]
    if !ok {
        return nil, fmt.Errorf("unknown runtime: %s (available: %v)", runtimeName, maps.Keys(registry.Runtimes))
    }
    
    return &runtimeDef, nil
}
```

### 4.4 User-Defined Runtimes

Users can add their own runtimes via Helm values:

```yaml
# values.yaml
mcpRuntimes:
  additionalRuntimes:
    my-weather-api:
      type: python
      image: myregistry/weather-mcp:v1.0.0
      description: "Weather API integration"
      requiredEnv:
      - WEATHER_API_KEY
```

Or by editing the ConfigMap directly:

```bash
kubectl edit configmap kaos-mcp-runtimes -n kaos-system
```

---

## 5. Built-in MCP Servers & Ecosystem Analysis

### 5.1 Strategy: Build vs Integrate

| Runtime | Strategy | Language | Source | Stars |
|---------|----------|----------|--------|-------|
| **rawpython** | Build | Python | KAOS internal | - |
| **kubernetes** | Integrate | Go | containers/kubernetes-mcp-server | 1076 |
| **slack** | Integrate | Node.js | zencoderai/slack-mcp-server | - |
| **discord** | Integrate | Python | hanweg/mcp-discord | 141 |
| **fetch** | Integrate | Python | modelcontextprotocol/servers | Official |

### 5.2 RawPython Runtime (Build)

**Purpose:** Execute Python code strings as MCP tools (current `fromString` behavior)

**Architecture Decision:** Create a simple standalone FastMCP server in `data-plane/mcp-servers/rawpython/`. This removes the `mcptools/server.py` from kaos-framework completely.

**Rationale for removing mcptools/server.py:**
- Current implementation has OTEL and other complexity that vanilla FastMCP doesn't need
- Using `kaos mcp init` -> `kaos mcp build` should be the dogfood approach
- Simplifies kaos-framework codebase (only keep `mcptools/client.py` used by Agent)
- rawpython is only for testing; OTEL instrumentation not really used there

**Simple rawpython implementation:**

```python
# data-plane/mcp-servers/rawpython/server.py
import os
from types import FunctionType
from fastmcp import FastMCP

mcp = FastMCP("RawPython MCP Server")

# Load tools from MCP_TOOLS_STRING env var
tools_string = os.getenv("MCP_TOOLS_STRING", "")
if tools_string:
    namespace = {}
    exec(tools_string, {}, namespace)
    for name, func in namespace.items():
        if isinstance(func, FunctionType):
            mcp.tool(name)(func)
```

```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: calculator
spec:
  runtime: rawpython
  params: |
    code: |
      def add(a: int, b: int) -> int:
          """Add two numbers."""
          return a + b
      
      def subtract(a: int, b: int) -> int:
          """Subtract b from a."""
          return a - b
```

**Note:** `mcptools/client.py` remains in `kaos-framework` as it's used by the Agent to connect to MCP servers. `mcptools/server.py` will be **removed** as it's replaced by the simpler rawpython implementation.

### 5.3 Kubernetes Runtime (Integrate)

**Source:** https://github.com/containers/kubernetes-mcp-server (1076 stars)

**Why integrate vs build:**
- Go native implementation (no kubectl wrapper)
- Actively maintained by Red Hat containers team
- Supports OpenShift, Helm, and extensive K8s operations
- Available as npm, pip, Docker, and native binary

**Available Tools:**
- `configuration_view` - View kubeconfig
- `resources_list` - List any K8s resources
- `resources_get` - Get specific resource
- `resources_create_or_update` - Create/update resources
- `resources_delete` - Delete resources
- `pods_log` - Get pod logs
- `pods_exec` - Exec into pods
- `pods_run` - Run container images
- `events_list` - List events
- `helm_install/list/uninstall` - Helm operations

**Example Usage:**

```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: k8s-tools
spec:
  runtime: kubernetes
```

### 5.4 Slack Runtime (Integrate)

**Source:** https://github.com/zencoderai/slack-mcp-server (Node.js)

**Why integrate:**
- Production-ready, maintained by Zencoder
- Supports Streamable HTTP transport
- Docker image available: `zencoderai/slack-mcp:latest`

**Available Tools:**
- `slack_list_channels`, `slack_post_message`, `slack_reply_to_thread`
- `slack_add_reaction`, `slack_get_channel_history`, `slack_get_thread_replies`
- `slack_get_users`, `slack_get_user_profile`

**Example Usage:**

```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: slack-tools
spec:
  runtime: slack
  container:
    env:
    - name: SLACK_BOT_TOKEN
      valueFrom:
        secretKeyRef:
          name: slack-credentials
          key: bot-token
    - name: SLACK_TEAM_ID
      valueFrom:
        secretKeyRef:
          name: slack-credentials
          key: team-id
```

### 5.5 Discord Runtime (Integrate)

**Source:** https://github.com/hanweg/mcp-discord (141 stars, Python)

**Available Tools:**
- `list_servers`, `get_server_info`, `get_channels`, `list_members`
- `send_message`, `read_messages`, `add_reaction`, `remove_reaction`
- `create_text_channel`, `delete_channel`, `add_role`, `remove_role`

**Example Usage:**

```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: discord-tools
spec:
  runtime: discord
  container:
    env:
    - name: DISCORD_TOKEN
      valueFrom:
        secretKeyRef:
          name: discord-credentials
          key: bot-token
```

### 5.6 Fetch Runtime (Integrate)

**Source:** https://github.com/modelcontextprotocol/servers/tree/main/src/fetch (Official MCP reference)

**Available Tools:**
- `fetch` - Fetch URL and convert to markdown/text

```yaml
apiVersion: kaos.tools/v1alpha1
kind: MCPServer
metadata:
  name: fetch-tools
spec:
  runtime: fetch
```

### 5.7 Other Runtimes to Consider (Future)

| Runtime | Source | Stars | Language | Notes |
|---------|--------|-------|----------|-------|
| `git` | modelcontextprotocol/servers | Official | Python | Git operations |
| `filesystem` | modelcontextprotocol/servers | Official | Python | File operations |
| `memory` | modelcontextprotocol/servers | Official | Python | Knowledge graph |
| `argocd` | argoproj-labs/mcp-for-argocd | 312 | TypeScript | GitOps |
| `telegram` | Various | - | - | Telegram bot |

---

## 6. CLI Design

### 6.1 Command Structure

```
kaos
├── install          # (moved to system)
├── uninstall        # (moved to system)
├── ui               # Launch web UI (existing)
├── version          # Show version (existing)
│
├── system           # NEW: Cluster/operator management
│   ├── install      # Install KAOS operator
│   ├── uninstall    # Uninstall KAOS operator
│   ├── create-rbac  # Create RBAC for MCPServer
│   ├── status       # Show operator status
│   └── runtimes     # List available MCP runtimes
│
├── mcp              # MCPServer commands
│   ├── init         # Initialize new MCPServer project
│   ├── build        # Build MCPServer container image
│   ├── deploy       # Deploy MCPServer to cluster
│   ├── list         # List MCPServers (kubectl wrapper)
│   ├── get          # Get MCPServer details (kubectl wrapper)
│   ├── logs         # View MCPServer logs (kubectl wrapper)
│   ├── delete       # Delete MCPServer (kubectl wrapper)
│   └── invoke       # Invoke MCP tool directly
│
├── agent            # Agent commands
│   ├── deploy       # Deploy Agent from YAML
│   ├── list         # List Agents
│   ├── get          # Get Agent details
│   ├── logs         # View Agent logs
│   ├── delete       # Delete Agent
│   └── invoke       # Send completion request
│
└── modelapi         # ModelAPI commands
    ├── deploy       # Deploy ModelAPI from YAML
    ├── list         # List ModelAPIs
    ├── get          # Get ModelAPI details
    ├── logs         # View ModelAPI logs
    ├── delete       # Delete ModelAPI
    └── invoke       # Send completion request
```

### 6.2 System Commands

#### `kaos system install`

```bash
$ kaos system install \
    --namespace kaos-system \
    --version v0.2.0 \
    --set gatewayAPI.enabled=true \
    --wait

Installing KAOS operator...
✓ Helm repo added
✓ CRDs installed
✓ Operator deployed
✓ Runtime registry created
```

#### `kaos system uninstall`

```bash
$ kaos system uninstall --namespace kaos-system
Uninstalling KAOS operator...
✓ Operator removed
```

#### `kaos system create-rbac`

Create RBAC for MCPServer Kubernetes runtime:

```bash
$ kaos system create-rbac \
    --name k8s-tools \
    --namespace default \
    --namespaces default,staging \
    --resources pods,services,deployments \
    --verbs get,list,watch

Creating RBAC for MCPServer k8s-tools...
✓ ServiceAccount created: mcpserver-k8s-tools
✓ Role created: mcpserver-k8s-tools
✓ RoleBinding created: mcpserver-k8s-tools

# For read-only access:
$ kaos system create-rbac --name k8s-tools --namespace default --read-only

# For cluster-wide access:
$ kaos system create-rbac --name k8s-tools --namespace default --cluster-wide
```

**Generated YAML:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mcpserver-k8s-tools
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mcpserver-k8s-tools
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mcpserver-k8s-tools
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mcpserver-k8s-tools
subjects:
- kind: ServiceAccount
  name: mcpserver-k8s-tools
  namespace: default
```

#### `kaos system status`

```bash
$ kaos system status
KAOS Operator Status:
  Namespace:  kaos-system
  Version:    v0.2.0
  Ready:      True
  Pods:       1/1

Available Runtimes:
  rawpython   - Execute Python code strings as MCP tools
  kubernetes  - Kubernetes API operations
  slack       - Slack messaging integration
  discord     - Discord messaging integration
  fetch       - Web content fetching

Gateway API: Enabled
  Gateway: kaos-gateway (kaos-system)
```

#### `kaos system runtimes`

```bash
$ kaos system runtimes
NAME         TYPE      DESCRIPTION                                              SOURCE
rawpython    python    Execute Python code strings as MCP tools                 Built-in
kubernetes   go        Kubernetes API operations                                containers/kubernetes-mcp-server
slack        nodejs    Slack messaging integration                              zencoderai/slack-mcp-server
discord      python    Discord messaging integration                            hanweg/mcp-discord
fetch        python    Web content fetching                                     modelcontextprotocol/servers
custom       -         User-provided container image                            -
```

### 6.3 MCP Commands

#### `kaos mcp init`

Initialize a FastMCP project:

```bash
$ kaos mcp init --name my-tools
Created my-tools/
├── server.py         # FastMCP server template
└── pyproject.toml    # Python dependencies

Next steps:
  cd my-tools
  kaos mcp build --name my-tools --registry docker.io/myuser
  kaos mcp deploy --name my-tools --namespace default
```

**Flags:**
- `--name` (required): Project name
- `--create-dockerfile`: Also generate Dockerfile (optional, auto-generated by build otherwise)
- `--create-yaml`: Also generate mcp.yaml manifest (optional, generated by deploy otherwise)

#### `kaos mcp build`

Build MCPServer container image:

```bash
$ kaos mcp build \
    --name my-tools \
    --tag v1.0.0 \
    --registry docker.io/myuser \
    --push \
    --kind-load

Building image docker.io/myuser/my-tools:v1.0.0...
✓ Dockerfile generated
✓ Image built
✓ Pushed to registry
✓ Loaded into KIND cluster
```

**Flags:**
- `--name` (required): Image name
- `--tag` (default: latest): Version tag
- `--registry`: Registry prefix
- `--path` (default: .): Path to FastMCP project
- `--fastmcp-path` (default: server:mcp): Path to FastMCP object
- `--push`: Push to registry after build
- `--kind-load`: Load into KIND cluster
- `--dockerfile`: Custom Dockerfile path (overrides auto-generation)
- `--create-dockerfile`: Save generated Dockerfile to project

#### `kaos mcp deploy`

Deploy MCPServer to cluster:

```bash
# Deploy custom runtime (from build)
$ kaos mcp deploy \
    --name my-tools \
    --namespace agents \
    --image docker.io/myuser/my-tools:v1.0.0

Deploying MCPServer agents/my-tools...
✓ MCPServer created

# Deploy existing runtime
$ kaos mcp deploy \
    --name k8s-tools \
    --namespace agents \
    --runtime kubernetes

Deploying MCPServer agents/k8s-tools...
✓ MCPServer created
```

**Flags:**
- `--name` (required): MCPServer name
- `--namespace, -n` (default: default): Target namespace
- `--runtime` (default: custom): Runtime name
- `--image`: Container image (required for custom runtime)
- `--file, -f`: Path to MCPServer YAML (alternative to flags)
- `--create-yaml`: Save generated YAML to file
- `--wait`: Wait for ready status
- `--dry-run`: Show YAML without applying

#### `kaos mcp list/get/logs/delete`

Thin wrappers around kubectl:

```bash
$ kaos mcp list -n agents
# Runs: kubectl get mcpservers -n agents

$ kaos mcp get my-tools -n agents
# Runs: kubectl get mcpserver my-tools -n agents -o yaml

$ kaos mcp logs my-tools -n agents --follow
# Runs: kubectl logs -l mcpserver=my-tools -n agents -f

$ kaos mcp delete my-tools -n agents
# Runs: kubectl delete mcpserver my-tools -n agents
```

#### `kaos mcp invoke`

Invoke an MCP tool directly:

```bash
$ kaos mcp invoke my-tools -n agents --tool hello --args '{"name": "World"}'
{"result": "Hello, World!"}
```

### 6.4 Agent & ModelAPI Commands

Follow same pattern as MCP:

```bash
# Agent
$ kaos agent deploy -f agent.yaml -n agents
$ kaos agent list -n agents
$ kaos agent get my-agent -n agents
$ kaos agent logs my-agent -n agents --follow
$ kaos agent delete my-agent -n agents
$ kaos agent invoke my-agent -n agents --message "Hello"

# ModelAPI
$ kaos modelapi deploy -f modelapi.yaml -n agents
$ kaos modelapi list -n agents
$ kaos modelapi get my-modelapi -n agents
$ kaos modelapi logs my-modelapi -n agents --follow
$ kaos modelapi delete my-modelapi -n agents
$ kaos modelapi invoke my-modelapi -n agents --message "Hello"
```

---

## 7. Project Structure

### 7.1 New Directory Layout

Rename `python/` to `data-plane/kaos-framework/` and add `data-plane/mcp-servers/`:

```
kaos/
├── operator/                      # Control plane (Go)
│   ├── api/v1alpha1/
│   ├── controllers/
│   │   ├── agent_controller.go
│   │   ├── mcpserver_controller.go
│   │   ├── modelapi_controller.go
│   │   └── integration/
│   │       ├── agent_test.go               # Existing content, same file
│   │       ├── agent_network_test.go       # NEW FILE: A2A tests (split from agent_test.go)
│   │       ├── mcpserver_test.go           # Existing content, same file
│   │       ├── mcpserver_runtime_test.go   # NEW FILE: NEW TESTS for runtime registry
│   │       ├── modelapi_test.go            # Existing content, same file
│   │       └── modelapi_types_test.go      # NEW FILE: proxy vs hosted (split from modelapi_test.go)
│   ├── config/
│   ├── chart/
│   └── tests/e2e/
│       ├── test_agent_e2e.py
│       ├── test_mcp_e2e.py
│       ├── test_mcp_runtimes_e2e.py    # NEW FILE: NEW TESTS for ecosystem runtimes
│       ├── test_modelapi_e2e.py
│       └── test_multi_agent_e2e.py
│
├── data-plane/                    # NEW: Data plane components
│   ├── kaos-framework/            # Renamed from python/
│   │   ├── agent/
│   │   ├── mcptools/              # MODIFIED: only client.py retained
│   │   │   └── client.py          # Used by Agent to connect to MCP servers
│   │   ├── modelapi/
│   │   ├── telemetry/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   └── tests/
│   │
│   └── mcp-servers/               # NEW: Separate MCP server images
│       └── rawpython/
│           ├── server.py          # Simple FastMCP server
│           ├── Dockerfile         # Standalone Python image
│           ├── pyproject.toml
│           └── tests/
│               └── test_rawpython.py
│
├── kaos-cli/                      # CLI (Python)
│   └── kaos_cli/
│       ├── main.py
│       ├── system/                # NEW: system commands
│       │   ├── __init__.py
│       │   ├── install.py         # Moved from install.py
│       │   ├── create_rbac.py
│       │   ├── status.py
│       │   └── runtimes.py
│       ├── mcp/                   # NEW
│       │   ├── __init__.py
│       │   ├── init.py
│       │   ├── build.py
│       │   ├── deploy.py
│       │   ├── crud.py
│       │   └── invoke.py
│       ├── agent/                 # NEW
│       │   ├── __init__.py
│       │   ├── deploy.py
│       │   ├── crud.py
│       │   └── invoke.py
│       ├── modelapi/              # NEW
│       │   ├── __init__.py
│       │   ├── deploy.py
│       │   ├── crud.py
│       │   └── invoke.py
│       ├── ui.py                  # Existing
│       └── utils/
│           ├── k8s.py
│           ├── docker.py
│           └── templates/
│               └── fastmcp/
│                   ├── server.py.tmpl
│                   └── pyproject.toml.tmpl
│
├── docs/
└── .github/
    └── workflows/
        ├── reusable-build-images.yaml
        └── build-mcp-rawpython.yaml  # NEW: only for rawpython
```

### 7.2 mcptools Changes

**Keep in `kaos-framework`:**
- `mcptools/client.py` - Used by Agent to connect to any MCP server

**Remove from `kaos-framework`:**
- `mcptools/server.py` - Replaced by simpler rawpython implementation

**The rawpython Docker image** is a standalone FastMCP application:

```dockerfile
# data-plane/mcp-servers/rawpython/Dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY pyproject.toml server.py ./
RUN pip install --no-cache-dir fastmcp uvicorn

CMD ["fastmcp", "run", "server:mcp", "--transport", "streamable-http", "--host", "0.0.0.0", "--port", "8000"]
```

This is the same approach users would use with `kaos mcp init` -> `kaos mcp build`, making it a true "dogfood" example.

---

## 8. Build & Release Integration

### 8.1 Separate Workflow for RawPython

Create `build-mcp-rawpython.yaml` that only triggers on changes to `data-plane/mcp-servers/rawpython/`:

```yaml
name: Build MCP RawPython Image

on:
  push:
    paths:
    - 'data-plane/mcp-servers/rawpython/**'
    branches: [main]
  workflow_call:
    inputs:
      version:
        required: true
        type: string

env:
  MCP_RAWPYTHON_IMAGE: axsauze/kaos-mcp-rawpython

jobs:
  build-rawpython:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    - uses: docker/build-push-action@v6
      with:
        context: ./data-plane/mcp-servers/rawpython
        push: true
        tags: |
          ${{ env.MCP_RAWPYTHON_IMAGE }}:${{ inputs.version || github.sha }}
```

### 8.2 Helm Values (Single defaultImages Section)

No distinction between internal and external images:

```yaml
# operator/chart/values.yaml
defaultImages:
  agentRuntime: axsauze/kaos-agent:v0.2.0
  mcpRawPython: axsauze/kaos-mcp-rawpython:v0.2.0
  litellm: ghcr.io/berriai/litellm:main-stable
  ollama: alpine/ollama:latest
  # Ecosystem MCP servers
  mcpKubernetes: ghcr.io/containers/kubernetes-mcp-server:latest
  mcpSlack: zencoderai/slack-mcp:latest
  mcpDiscord: ghcr.io/hanweg/mcp-discord:latest
  mcpFetch: mcp/fetch:latest
```

### 8.3 Version Strategy

- **KAOS-built images** (`kaos-agent`, `kaos-operator`, `kaos-mcp-rawpython`): Use same version tag as release (e.g., `v0.2.0`)
- **Ecosystem images**: Pinned versions in Helm values, updated manually when needed

---

## 9. Testing Strategy

### 9.1 Unit Tests

```
data-plane/kaos-framework/tests/     # Existing
data-plane/mcp-servers/rawpython/tests/
  └── test_rawpython.py              # Tests specific to rawpython runtime
```

### 9.2 Integration Tests (Go)

Split existing tests into focused files:

| File | Content |
|------|---------|
| `agent_test.go` | Existing tests (no change) |
| `agent_network_test.go` | **Split from agent_test.go**: A2A delegation tests |
| `mcpserver_test.go` | Existing tests (no change) |
| `mcpserver_runtime_test.go` | **NEW TESTS**: Runtime registry, custom runtime |
| `modelapi_test.go` | Existing tests (no change) |
| `modelapi_types_test.go` | **Split from modelapi_test.go**: Proxy vs hosted tests |

### 9.3 E2E Tests

Single file for runtime tests (`test_mcp_runtimes_e2e.py`):

```python
"""E2E tests for MCP runtime integrations.

Only runs when data-plane/mcp-servers/** or operator/controllers/mcpserver* changed.
"""

import pytest

@pytest.mark.asyncio
async def test_rawpython_runtime(test_namespace, shared_modelapi):
    """Test rawpython runtime with inline code."""
    pass

@pytest.mark.asyncio  
async def test_kubernetes_runtime(test_namespace):
    """Test kubernetes runtime (read-only operations)."""
    pass

# Slack/Discord tests skipped in CI (require credentials)
@pytest.mark.skip(reason="Requires external credentials")
async def test_slack_runtime(test_namespace):
    pass
```

### 9.4 CI Triggers

Only run runtime E2E tests when relevant files change:

```yaml
# Only trigger on specific paths
on:
  push:
    paths:
    - 'data-plane/mcp-servers/rawpython/**'
    - 'operator/controllers/mcpserver*'
```

---

## 10. Refactoring Scope

### 10.1 Files to Modify for `env` Migration

Moving `env` from `config.env` to `container.env` requires updates to:

**Go Types (operator/api/v1alpha1/):**
- `agent_types.go` - Remove `Env` from `AgentConfig`, add to `ContainerOverride`
- `mcpserver_types.go` - Remove `Env` from `MCPServerConfig`, add to `ContainerOverride`
- `modelapi_types.go` - Remove `Env` from `ProxyConfig` and `HostedConfig`, add to `ContainerOverride`

**Go Controllers (operator/controllers/):**
- `agent_controller.go` - Read env from `spec.container.env` instead of `spec.config.env`
- `mcpserver_controller.go` - Read env from `spec.container.env` instead of `spec.config.env`
- `modelapi_controller.go` - Read env from `spec.container.env` instead of `spec.proxyConfig.env`/`spec.hostedConfig.env`

**Go Tests (operator/controllers/integration/):**
- `agent_test.go` - Update test specs
- `mcpserver_test.go` - Update test specs
- `modelapi_test.go` - Update test specs

**CRD YAML (operator/config/crd/bases/, operator/chart/crds/):**
- All CRD YAML files will be regenerated via `make generate manifests`

**E2E Tests (operator/tests/e2e/):**
- `test_agent_e2e.py`
- `test_mcp_e2e.py` / `test_mcp_tools_e2e.py`
- `test_modelapi_e2e.py`
- `test_multi_agent_e2e.py`
- `conftest.py` (helper functions)

**Documentation (docs/):**
- `docs/operator/agent-crd.md`
- `docs/operator/mcpserver-crd.md`
- `docs/operator/modelapi-crd.md`
- `docs/operator/overview.md`
- `docs/reference/environment-variables.md`
- `docs/tutorials/*.md`

**Samples (operator/config/samples/, docs/examples/):**
- All sample YAML files

### 10.2 Files to Modify for MCPServer `tools` Removal

**Go Types:**
- `operator/api/v1alpha1/mcpserver_types.go` - Remove `MCPToolsConfig`, add `Params` field

**Go Controllers:**
- `operator/controllers/mcpserver_controller.go` - Remove fromString/fromPackage/fromSecretKeyRef logic, add runtime resolution

**Integration Tests:**
- `operator/controllers/integration/mcpserver_test.go` - Update tests

### 10.3 Files to Remove

**kaos-framework mcptools/server.py removal:**
- `data-plane/kaos-framework/mcptools/server.py` - DELETE (replaced by standalone rawpython)
- Update `data-plane/kaos-framework/mcptools/__init__.py` - Remove server imports if any
- Update any tests referencing mcptools.server

---

## 11. Documentation Structure

### 11.1 MCPServer Documentation Hub

The MCPServer documentation is restructured into a modular format:

```
docs/operator/mcpserver/
├── index.md                    # Overview and navigation
├── runtime-registry.md         # How the runtime registry works
├── built-in-runtimes.md        # rawpython, kubernetes, slack details
├── custom-runtimes.md          # Building and deploying custom runtimes
└── rbac-setup.md               # ServiceAccountName and permissions
```

**index.md** contents:
- What is an MCPServer?
- Quick start example
- Links to sub-pages

**runtime-registry.md** contents:
- ConfigMap structure
- How resolution works
- Adding custom runtimes to registry
- paramsEnvVar explanation

**built-in-runtimes.md** contents:
- rawpython: params format, examples
- kubernetes: serviceAccountName, RBAC requirements, examples
- slack: required env vars, setup guide

**custom-runtimes.md** contents:
- `kaos mcp init` workflow
- `kaos mcp build` workflow
- `kaos mcp deploy` workflow
- Full example: init → build → deploy

**rbac-setup.md** contents:
- `kaos system create-rbac` command
- ServiceAccountName field
- Required permissions for kubernetes runtime
- Example YAML

### 11.2 CLI Documentation

```
docs/cli/
├── index.md                    # CLI overview and installation
├── system.md                   # kaos system commands
├── mcp.md                      # kaos mcp commands
├── agent.md                    # kaos agent commands
└── modelapi.md                 # kaos modelapi commands
```

**index.md** contents:
- Installation (pip install kaos-cli)
- Quick start
- Command structure overview

**system.md** contents:
- `kaos system install` - Install operator via Helm
- `kaos system uninstall` - Uninstall operator
- `kaos system create-rbac` - Create ServiceAccount with RBAC
- `kaos system status` - Check operator status
- `kaos system runtimes` - List available runtimes

**mcp.md** contents:
- `kaos mcp init` - Initialize FastMCP project
- `kaos mcp build` - Build container image
- `kaos mcp deploy` - Deploy MCPServer
- `kaos mcp list` - List MCPServers
- `kaos mcp get <name>` - Get MCPServer details
- `kaos mcp logs <name>` - View logs
- `kaos mcp delete <name>` - Delete MCPServer
- `kaos mcp invoke` - Call MCP tool

**agent.md** contents:
- `kaos agent deploy` - Deploy Agent from YAML
- `kaos agent list/get/logs/delete` - CRUD operations
- `kaos agent invoke` - Send chat completion

**modelapi.md** contents:
- `kaos modelapi deploy` - Deploy ModelAPI from YAML
- `kaos modelapi list/get/logs/delete` - CRUD operations
- `kaos modelapi invoke` - Send completion request

### 11.3 VitePress Sidebar Updates

```typescript
// docs/.vitepress/config.ts sidebar additions
{
  text: 'MCPServer',
  collapsed: false,
  items: [
    { text: 'Overview', link: '/operator/mcpserver/' },
    { text: 'Runtime Registry', link: '/operator/mcpserver/runtime-registry' },
    { text: 'Built-in Runtimes', link: '/operator/mcpserver/built-in-runtimes' },
    { text: 'Custom Runtimes', link: '/operator/mcpserver/custom-runtimes' },
    { text: 'RBAC Setup', link: '/operator/mcpserver/rbac-setup' },
  ]
},
{
  text: 'CLI Reference',
  collapsed: false,
  items: [
    { text: 'Overview', link: '/cli/' },
    { text: 'System Commands', link: '/cli/system' },
    { text: 'MCP Commands', link: '/cli/mcp' },
    { text: 'Agent Commands', link: '/cli/agent' },
    { text: 'ModelAPI Commands', link: '/cli/modelapi' },
  ]
}
```

---

## 12. Decisions

### 12.1 Resolved Questions

| Question | Decision | Rationale |
|----------|----------|-----------|
| `tools` list in MCPServer spec? | **Remove entirely** | Agent discovers tools via MCP protocol; not used |
| Runtime registry location? | **ConfigMap** | Dynamic, editable, simple |
| Node.js runtime support? | **Yes** | Ecosystem servers (Slack) use Node.js |
| Python runtime support? | **Yes** | uvx packages supported via custom runtime |
| `env` location? | **Move to `container.env`** | Consistent across all CRDs |
| Backwards compatibility? | **None** | Alpha repo, breaking changes OK |
| RBAC for kubernetes runtime? | **`kaos system create-rbac` command** | Simple CLI helper |
| CLI interactive input? | **Flags only** | Simpler implementation |
| CRUD commands? | **kubectl wrappers** | Reuse existing tools |
| Runtime versioning? | **KAOS version for built images** | External images pinned in Helm values |
| mcptools/server.py? | **Remove and replace** | Simpler standalone rawpython; dogfood `kaos mcp init/build` |

---

## 13. Future Enhancements

The following are explicitly deferred for later iterations:

### 13.1 Runtime Enhancements

- `tools` list re-introduction with `["*"]` wildcard support
- Runtime param schema validation
- Per-runtime documentation in ConfigMap

### 13.2 Additional Runtimes (stdio-to-HTTP adapter required)

- `discord` - Messaging (stdio only currently)
- `fetch` - Web content fetching (stdio only currently)
- `git` - Official MCP reference
- `filesystem` - Official MCP reference
- `telegram` - Messaging
- `argocd` - GitOps

### 13.3 CLI Enhancements

- `kaos agent chat` - Interactive chat mode (beyond invoke)
- `kaos mcp validate` - Validate FastMCP server without building

### 13.4 Observability

- Per-tool metrics (call count, latency)
- Tool call tracing dashboard

---

## Appendix A: Implementation Phases

### Phase 1: CRD & Controller Updates (Week 1-2)
- [ ] Add `ContainerOverride` type with `env` field
- [ ] Update MCPServer types (remove tools, add runtime, add params)
- [ ] Update Agent types (add container, remove config.env)
- [ ] Update ModelAPI types (add container, remove proxyConfig.env/hostedConfig.env)
- [ ] Update all controllers to read env from container.env
- [ ] Implement ConfigMap runtime registry in mcpserver_controller
- [ ] Run `make generate manifests`
- [ ] Update all integration tests

### Phase 2: Project Restructure (Week 3)
- [ ] Rename `python/` to `data-plane/kaos-framework/`
- [ ] Create `data-plane/mcp-servers/rawpython/`
- [ ] Update CI workflows for new paths
- [ ] Create `build-mcp-rawpython.yaml` workflow

### Phase 3: CLI System Commands (Week 4)
- [ ] Create `kaos_cli/system/` module
- [ ] Move install/uninstall to system
- [ ] Implement `kaos system create-rbac`
- [ ] Implement `kaos system status`
- [ ] Implement `kaos system runtimes`

### Phase 4: CLI Resource Commands (Week 5-6)
- [ ] `kaos mcp list/get/logs/delete` (kubectl wrappers)
- [ ] `kaos mcp invoke`
- [ ] `kaos agent deploy/list/get/logs/delete/invoke`
- [ ] `kaos modelapi deploy/list/get/logs/delete/invoke`

### Phase 5: CLI Build Workflow (Week 7)
- [ ] `kaos mcp init` with `--create-dockerfile` and `--create-yaml` flags
- [ ] `kaos mcp build` with `--create-dockerfile` flag
- [ ] `kaos mcp deploy` with `--create-yaml` flag

### Phase 6: Ecosystem Runtime Integration (Week 8)
- [ ] Add kubernetes runtime to registry + E2E test
- [ ] Add slack runtime to registry
- [ ] Add discord runtime to registry
- [ ] Add fetch runtime to registry
- [ ] Update Helm values with all images

### Phase 7: Documentation & Samples (Week 9)
- [ ] Update all CRD documentation
- [ ] Update all sample YAML files
- [ ] Add CLI reference docs
- [ ] Add tutorial: Building custom MCPServer
- [ ] Add tutorial: Using Kubernetes tools

### Phase 8: E2E Tests & Polish (Week 10)
- [ ] Update all E2E tests for new CRD structure
- [ ] Add `test_mcp_runtimes_e2e.py`
- [ ] Split Go integration tests into new files
- [ ] Final testing and bug fixes

---

## Appendix B: Runtime ConfigMap Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kaos-mcp-runtimes
  namespace: {{ .Release.Namespace }}
data:
  runtimes.yaml: |
    runtimes:
      rawpython:
        type: python
        image: {{ .Values.defaultImages.mcpRawPython }}
        description: "Execute Python code strings as MCP tools"
        command: ["python", "-m", "mcptools.server"]
        paramsEnvVar: MCP_TOOLS_STRING
        
      kubernetes:
        type: go
        image: {{ .Values.defaultImages.mcpKubernetes }}
        description: "Kubernetes API operations"
        
      slack:
        type: nodejs
        image: {{ .Values.defaultImages.mcpSlack }}
        description: "Slack messaging integration"
        command: ["--transport", "http"]
        requiredEnv:
        - SLACK_BOT_TOKEN
        - SLACK_TEAM_ID
        
      discord:
        type: python
        image: {{ .Values.defaultImages.mcpDiscord }}
        description: "Discord messaging integration"
        requiredEnv:
        - DISCORD_TOKEN
        
      fetch:
        type: python
        image: {{ .Values.defaultImages.mcpFetch }}
        description: "Web content fetching"
```

---

*End of Design Proposal*
