# dcc-mcp-facade

Reverse-proxy facade that exposes MCP servers behind a single host. Request flow:

```
client ──HTTPS──> nginx (TLS termination for *.mcp.data.bs.ch)
                       │ proxy_pass (preserves full Host header)
                       ▼
                   Traefik (in-cluster reverse proxy)
                       │ routes by Host header
          ┌────────────┼──────────────────────────┐
          ▼            ▼                          ▼
   ogd.mcp.data.bs.ch  grossrat.mcp.data.bs.ch    statistik.mcp.data.bs.ch
   ogd container       grossrat container         statistik container
   (mcp-data-bs)       (grossrat-bs-mcp)          (mcp-statistikportal)
```

- **nginx** is external to this project: it terminates TLS and forwards every
  `*.mcp.data.bs.ch` request to Traefik. See `nginx-example.conf`.
- **Traefik** (in this compose) only routes the `ogd.mcp.data.bs.ch`,
  `grossrat.mcp.data.bs.ch` and `statistik.mcp.data.bs.ch` hostnames, so other
  subdomains forwarded by nginx simply get no route (404).

## Services

| Service        | Container      | Image                          | Role                                             |
|----------------|----------------|--------------------------------|--------------------------------------------------|
| `ogd`          | `mcp-ogd`      | `ghcr.io/dcc-bs/mcp-data-bs`   | MCP server for the data.bs.ch open-data portal    |
| `grossrat`     | `mcp-grossrat` | `ghcr.io/dcc-bs/grossrat-bs-mcp` | MCP server for the Grosser Rat Basel-Stadt: every business item, document and transcript since 1973 |
| `statistik`    | `mcp-statistik` | `ghcr.io/dcc-bs/mcp-statistikportal` | MCP server for the statistics portal statistik.bs.ch: indicator search, metadata and time series |
| `reverse-proxy`| `mcp-reverse-proxy` | `traefik:v3.6.1`          | Routes each hostname to its container |

## Files

| File | Purpose |
|------|---------|
| `compose.yml` | Defines the Traefik + ogd + grossrat + statistik stack |
| `nginx-example.conf` | Example nginx server blocks routing `*.mcp.data.bs.ch` to Traefik |
| `.env.example` | Template for environment overrides |
| `.env` | Local overrides (gitignored) |

## Prerequisites

- Docker Engine **v29+**, which dropped legacy Docker API versions (min 1.44).
  This requires **Traefik v3.6.1+** — earlier Traefik versions hardcode the
  Docker API at v1.24 and fail with `client version 1.24 is too old`.
- Docker Compose v2.

## Configuration

Copy the example and adjust for your host:

```bash
cp .env.example .env
```

| Variable | Description | Example |
|----------|-------------|---------|
| `DOCKER_SOCKET` | Path to the Docker daemon socket, used by Traefik's Docker provider. | `=/run/user/1000/docker.sock` (rootless) or `=/var/run/docker.sock` (rootful) |
| `TRAEFIK_PORT` | Host port Traefik publishes on (used when reaching it via a host port). | `8001` |

## Run

```bash
docker compose up -d
```

Verify:

```bash
docker compose ps            # all services Up, ogd, grossrat and statistik healthy
docker compose logs reverse-proxy
```

## Testing locally

There is no local DNS/nginx, so emulate nginx by sending the `Host` header
manually to Traefik's published port (`localhost:8001`):

```bash
# 1. Health check
curl -H "Host: ogd.mcp.data.bs.ch" http://localhost:8001/healthz
# {"status":"ok"}

# 2. MCP initialize — the real test
curl -N \
  -H "Host: ogd.mcp.data.bs.ch" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"curl","version":"1.0"}}}' \
  http://localhost:8001/mcp
# SSE response with serverInfo (name: data.bs.ch)

# 3. The same for the Grosser Rat server
curl -H "Host: grossrat.mcp.data.bs.ch" http://localhost:8001/healthz
# {"status":"ok"}
curl -H "Host: grossrat.mcp.data.bs.ch" http://localhost:8001/health
# documents, search mode (hybrid) and vector coverage

# 4. The same for the statistics portal server
curl -H "Host: statistik.mcp.data.bs.ch" http://localhost:8001/healthz
# {"status":"ok"}
```

## The grossrat image

`ghcr.io/dcc-bs/grossrat-bs-mcp` carries its databases inside (~3.3 GB compressed,
~10 GB unpacked), so it needs no volume and no model server; the search model runs
on the CPU in the container. It is built and pushed from the machine that holds the
data, not by CI (see the grossrat-bs-mcp repository, `mise run deploy:mcp`); a new
version therefore brings new data. The package is in the dcc-bs organisation, so the
host needs `docker login ghcr.io` with `read:packages` unless the package is public.
Update with `docker compose pull grossrat && docker compose up -d grossrat`.

## Production hostname / allowed hosts

`MCP_ALLOWED_HOSTS` is the app's DNS-rebinding `Host`-header allowlist.
nginx forwards a **bare** Host header (e.g. `Host: ogd.mcp.data.bs.ch`, no
port). It must therefore be an **exact match**:

```yaml
MCP_ALLOWED_HOSTS: "ogd.mcp.data.bs.ch"   # correct for nginx in front
```

Do **not** use the `*:port` wildcard form (`ogd.mcp.data.bs.ch:*`) behind
nginx: that pattern only matches a Host header that includes a port, so a bare
host would be rejected with `421 Invalid Host header`.

## Endpoints

- `https://ogd.mcp.data.bs.ch/mcp` → MCP streamable HTTP endpoint
- `https://ogd.mcp.data.bs.ch/healthz` → liveness check (`{"status":"ok"}`)
- `https://grossrat.mcp.data.bs.ch/mcp` → MCP streamable HTTP endpoint
- `https://grossrat.mcp.data.bs.ch/healthz` → liveness check (`{"status":"ok"}`)
- `https://grossrat.mcp.data.bs.ch/health` → state of the corpus (documents, search mode)
- `https://statistik.mcp.data.bs.ch/mcp` → MCP streamable HTTP endpoint
- `https://statistik.mcp.data.bs.ch/healthz` → liveness check (`{"status":"ok"}`)
- `http://localhost:<TRAEFIK_PORT>` → Traefik HTTP entrypoint (for local tests)

## Troubleshooting

**`client version 1.24 is too old`** — Docker v29 removed old API versions.
Use Traefik `v3.6.1`+ (already pinned in this project).

**`421 Invalid Host header`** — the `Host` header didn't match
`MCP_ALLOWED_HOSTS`. Verify nginx forwards the bare host and that
`MCP_ALLOWED_HOSTS` is the exact hostname (see above).

**`404 page not found`** from Traefik — the hostname isn't routed. Only the
hostnames listed under Endpoints are routed; confirm the `Host` header matches and that
Traefik has picked up the router (`curl -H "Host: ogd.mcp.data.bs.ch" http://localhost:8001/healthz`).

**Traefik cannot connect to the Docker daemon** — the `DOCKER_SOCKET` in
`.env` is wrong. Run `docker context ls` to find the daemon socket path.
