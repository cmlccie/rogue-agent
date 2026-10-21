# Rogue Agent

A Kubernetes-native "rogue" AI agent for demonstrating zero-trust network policies. It discovers locally hosted AI services (models, agents, and tools) and external AI services (ChatGPT, Claude, etc.), then persistently attempts to connect to and access every one of them.

Deploy it into an agent environment, apply your Cilium Network Policies, and watch in real time which communication paths the policies allow and which they block.

## Overview

`rogue-agent` behaves like an unconstrained agent that tries to reach everything it can find. Its terminal UI makes the effect of each L3/L4/L7 policy visible: an explicitly allowed path stays green, and everything else flips to red with the reason it failed.

The backend server:

- **Discovers** AI components from Kubernetes services (by label selector) and from a static config file
- **Probes** each target on a configurable interval using the appropriate protocol, and inventories what it finds: served models (OpenAI-compatible `/models`), advertised skills (A2A agent card), available tools (MCP `initialize` + `tools/list` over Streamable HTTP), or plain HTTP reachability
- **Classifies** each failure (timeout, refused, DNS, TLS, unauthorized/forbidden, HTTP error, bad response) so the effect of a network or authorization policy is visible at a glance
- **Tracks** how long each target has been in its current state and its recent probe history; targets that disappear from Kubernetes or the config are removed
- **Exposes** current state over a local HTTP API

A TUI client polls the backend and displays live, color-coded status across four inventory panes (Models / Agents (A2A) / Tools (MCP) / External Hosts), with a detail pane showing the selected target, its last probe error or inventory, and its probe history. Press `r` to run discovery and a probe pass immediately.

## Demo: constraining the rogue agent

1. Deploy `rogue-agent` with no policy applied. Every discovered model, agent, tool, and external host shows ✓ because nothing restricts its egress.
2. Apply a zero-trust CiliumNetworkPolicy that allows only the paths the agent needs (for example, `deploy/netpol-demo.yaml`).
3. Watch the TUI. Within one probe interval, every path the policy does not allow flips to ✗, and the detail pane shows how it failed: a timeout for an L3/L4 drop, or an HTTP 403 for an L7 denial.
4. Delete or relax the policy, and the targets recover on the next probe pass.

## Usage

```bash
# Start the backend server
rogue-agent serve --config /etc/rogue-agent/config.yaml

# Launch the terminal UI (connects to server at localhost:8000 by default)
rogue-agent tui

# Print a one-shot status snapshot
rogue-agent status
rogue-agent status --json
```

Run both components locally for development:

```bash
# Terminal 1 — start the server with mock data (no K8s or live endpoints needed)
make mock

# Terminal 2 — launch the TUI or check status
cargo run -- tui
make status
```

## Configuration

The server reads a YAML config file. Example:

```yaml
general:
  discovery_interval_secs: 30
  probe_interval_secs: 10
  http_timeout_secs: 5

kubernetes:
  enabled: true
  namespaces: [] # empty = all accessible namespaces
  label_selectors:
    models: "app=vllm"
    agents: "app=a2a-agent"
    tools: "app in (mcp-fs, mcp-github)"

manual:
  websites:
    - name: anthropic.com
      url: https://anthropic.com
  models: []
  agents: []
  tools: []
```

Config changes are picked up at runtime without a restart (hot-reload via file watcher, compatible with Kubernetes ConfigMap symlink swaps).

## Development

```bash
make build    # cargo build
make release  # cargo build --release
make check    # fmt check + clippy + tests
make test     # cargo test
make lint     # cargo fmt --check && cargo clippy -D warnings
make fmt      # cargo fmt
make clean    # cargo clean
```

## Deploying to Kubernetes

The `deploy/` directory contains ready-to-use manifests:

| File               | Purpose                                                                      |
| ------------------ | ---------------------------------------------------------------------------- |
| `configmap.yaml`   | Server config mounted at `/etc/rogue-agent/config.yaml`                   |
| `deployment.yaml`  | Single-replica Deployment + Service (port 8000)                              |
| `rbac.yaml`        | ServiceAccount + ClusterRole with read-only access to Services and Endpoints |
| `netpol-demo.yaml` | Example CiliumNetworkPolicy that constrains the rogue agent's egress         |

```bash
kubectl apply -f deploy/rbac.yaml
kubectl apply -f deploy/configmap.yaml
kubectl apply -f deploy/deployment.yaml
```

The container image is built from `Containerfile` as a fully-static musl binary on a distroless base (no shell, non-root). Multi-arch builds (amd64/arm64) work without cross-compilation.

## API

The server exposes a REST API on port 8000:

| Endpoint                   | Description                       |
| -------------------------- | --------------------------------- |
| `GET /api/v1/healthz`      | Health check                      |
| `GET /api/v1/state`        | Full state snapshot               |
| `GET /api/v1/targets`      | List targets (filterable by kind) |
| `GET /api/v1/targets/{id}` | Single target detail              |
| `POST /api/v1/refresh`     | Run discovery + probing now       |
