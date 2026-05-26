# ToolHive MCP Optimizer — VAI Fork

> **Fork maintained by Gerald Staruiala / [Value Added International](https://www.value-added-international.com)**
>
> This is a fork of the original [stackloklabs/mcp-optimizer](https://github.com/stackloklabs/mcp-optimizer)
> (archived upstream). The upstream project has been folded into ToolHive's vMCP feature.
> This fork continues standalone development to support Podman-based local deployments
> where the upstream container image no longer works correctly.
>
> All original code is copyright © Stacklok, Inc. and contributors, licensed under the
> Apache License 2.0. New contributions in this fork are copyright © 2026 Gerald Staruiala /
> Value Added International, also under the Apache License 2.0.

---

## What was changed and why

The upstream `ghcr.io/stackloklabs/mcp-optimizer:0.3.0` image has two bugs that prevent
it from working in a standard Podman-based ToolHive setup:

### Bug 1 — ToolHive API server not being reached

The container was scanning proxy ports (50000–50100) for `GET /api/v1beta/version`, but
those ports belong to MCP proxy processes, not the ToolHive API server. The ToolHive API
server (`thv serve`) must be running separately and the optimizer must be pointed at it
via `TOOLHIVE_HOST` + `TOOLHIVE_PORT`.

**Fix:** `setup-mcp-optimizer.sh` now starts `thv serve --host 0.0.0.0 --port 8080` before
launching the container, and passes `TOOLHIVE_HOST=127.0.0.1` and `TOOLHIVE_PORT=8080`
as explicit environment variables.

### Bug 2 — `host.docker.internal` does not resolve under Podman

The Dockerfile hardcoded `ENV TOOLHIVE_HOST=host.docker.internal`. This hostname is
Docker-specific. Under Podman the correct hostname is `host.containers.internal`.
Additionally, the MCP proxy ports are bound to `127.0.0.1` (loopback), so they are only
reachable from inside the container when using `--network host`.

**Fix:**
- `Dockerfile`: changed default `TOOLHIVE_HOST` to `host.containers.internal`
- `toolhive_client.py`: added a fallback host list in `_discover_port_async` — tries
  the configured host first, then `host.containers.internal`, then `host.docker.internal`
- Container is run with `--network host` so it shares the host network namespace and
  can reach all loopback-bound proxy ports directly

---

## Original project description

An intelligent intermediary MCP server that provides semantic tool discovery,
caching, and unified access to multiple MCP servers through a single endpoint.

### Features

- **Semantic Tool Discovery**: Intelligently discover and route requests to appropriate MCP tools
- **Unified Access**: Single endpoint to access multiple MCP servers
- **Tool Management**: Manage large numbers of MCP tools seamlessly
- **Group Filtering**: Filter tool discovery by ToolHive groups for multi-environment support

---

## Requirements

- Python 3.13+
- uv package manager
- ToolHive CLI (`thv`) v0.12.2+
- Podman (Docker also supported)

---

## Quickstart (Podman + local build)

### 1. Download models and build the local image

```bash
# From the repo root
uv sync --group offline-models
uv run --group offline-models python scripts/download_models.py
podman build -t localhost/vai-mcp-optimizer:local .
```

### 2. Start ToolHive API server

The optimizer needs the ToolHive REST API to be reachable. Start it bound to all
interfaces so the container can connect via the host network:

```bash
thv serve --host 0.0.0.0 --port 8080
```

### 3. Run the optimizer

```bash
thv group create optim

thv run \
  --name vai-mcp-optimizer \
  --group optim \
  --proxy-port 50000 \
  --network host \
  --env ALLOWED_GROUPS=code \
  --env TOOLHIVE_HOST=127.0.0.1 \
  --env TOOLHIVE_PORT=8080 \
  localhost/vai-mcp-optimizer:local
```

Or use the provided script which handles all steps:

```bash
./setup-mcp-optimizer.sh
```

### 4. Configure your AI client

```bash
thv client register cursor --group optim
# or
thv client register claude --group optim
```

---

## Configuration

### Key environment variables

| Variable | Default | Description |
|---|---|---|
| `TOOLHIVE_HOST` | `host.containers.internal` | Host where `thv serve` is reachable |
| `TOOLHIVE_PORT` | *(scan 50000–50100)* | Port for ToolHive API server |
| `ALLOWED_GROUPS` | *(all groups)* | Comma-separated ToolHive group names to discover tools from |
| `RUNTIME_MODE` | `docker` | `docker` or `k8s` |
| `TOOLHIVE_MAX_RETRIES` | `100` | Max connection retry attempts |
| `TOOLHIVE_INITIAL_BACKOFF` | `1.0` | Initial retry delay in seconds |
| `TOOLHIVE_MAX_BACKOFF` | `60.0` | Maximum retry delay in seconds |

### Runtime modes

- **docker** (default): MCP servers run as containers via Docker/Podman
- **k8s**: MCP servers run as Kubernetes workloads

### Connection resilience

If `thv serve` restarts, the optimizer will automatically rescan for ToolHive and
retry with exponential backoff (1s → 2s → 4s → … up to 60s), up to 100 attempts.

---

## Development

```bash
# Install dependencies
uv sync

# Format
task format

# Lint
task lint

# Type check
task typecheck

# Test
task test
```

---

## License

Original code: Apache License 2.0 — Copyright © Stacklok, Inc. and contributors.
Fork modifications: Apache License 2.0 — Copyright © 2026 Gerald Staruiala / Value Added International.

See [LICENSE](LICENSE) for the full license text.

---

## Upstream references

- Original repo (archived): https://github.com/stackloklabs/mcp-optimizer
- ToolHive docs: https://docs.stacklok.com/toolhive
- ToolHive vMCP (upstream path forward): https://docs.stacklok.com/toolhive/guides-vmcp/optimizer
