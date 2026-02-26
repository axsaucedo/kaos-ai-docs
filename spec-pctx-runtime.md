# Spec: Add “pctx code-mode” MCP runtime to KAOS

## Version A (concise)

### Scope
Add a built-in MCP runtime based on **pctx (Port of Context)** to provide a “code-mode” MCP server that can sit in front of other MCP servers (KAOS-managed and external) and expose a unified endpoint. Follow the upstream docs and examples (pctx “Unified MCP” workflows such as `pctx mcp init/add/dev`).  
Reference: https://github.com/portofcontext/pctx?tab=readme-ov-file

### KAOS codebase touchpoints
- CRD/runtime/params contract: `docs/operator/mcpserver-crd.md`
- Runtime registry: `operator/chart/templates/mcp-runtimes-configmap.yaml` (`kaos-mcp-runtimes`)
- Default images: `operator/chart/values.yaml` (`defaultImages.*`)
- Controller wiring: `operator/controllers/mcpserver_controller.go`
- CLI runtime listing: `kaos-cli/kaos_cli/system/runtimes.py`
- Examples + CI E2E pattern: `docs/examples/kaos-monkey.*`, `.github/workflows/reusable-tests.yaml`, `operator/tests/e2e/test_examples_e2e.py`

### Deliverables
1) KAOS runtime Docker image for pctx (HTTP, binds `0.0.0.0:8000`, configured from `spec.params`).  
2) Register runtime `pctx` in `kaos-mcp-runtimes` (image/transport/paramsEnvVar).  
3) Minimal `spec.params` mapping for pctx to accept upstream MCP servers as either KAOS refs (cluster-local) or external URLs (per pctx docs).  
4) End-to-end example (KAOS Monkey style) included in CI E2E examples shard.  
5) Must fit current `MCPServer` CRD (no CRD changes; use `spec.runtime`, `spec.params`, optional `spec.container.env`).

---

## Version B (more compact)

Add a new built-in MCP runtime **`pctx`** (Port of Context) as a KAOS-managed MCPServer runtime, per upstream docs: https://github.com/portofcontext/pctx?tab=readme-ov-file. Ship a KAOS Docker image that runs pctx as an HTTP MCP server on `0.0.0.0:8000`, configured via `MCPServer.spec.params`. The params must support upstream MCP servers both as **KAOS refs** (other MCPServers in-cluster) and **external URLs** (matching pctx’s unified MCP usage). Wire this into the existing runtime registry and defaults (`operator/chart/templates/mcp-runtimes-configmap.yaml`, `operator/chart/values.yaml`) with controller support already handled by `operator/controllers/mcpserver_controller.go`, and ensure the runtime shows up in the CLI (`kaos-cli/kaos_cli/system/runtimes.py`). Add a KAOS Monkey–style end-to-end example under `docs/examples/` and run it in the existing examples E2E CI path (`operator/tests/e2e/test_examples_e2e.py`, workflow in `.github/workflows/reusable-tests.yaml`). Must work with the current `MCPServer` CRD as-is (`docs/operator/mcpserver-crd.md`).
