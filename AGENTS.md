# AGENTS.md

## Project Overview

Dolibarr MCP Server — a hosted, remote Model Context Protocol server providing AI agents with full CRUD access to a Dolibarr ERP/CRM instance over streamable HTTP transport.

- **Language**: Python 3.11+, async-first (aiohttp, asyncio)
- **Framework**: MCP SDK with Starlette + Uvicorn for HTTP transport
- **Config**: Pydantic Settings, loads from `.env` and environment variables

## Quick Commands

| Task | Command |
|------|---------|
| Install (dev) | `pip install -e ".[dev]"` |
| Run server (stdio) | `python -m dolibarr_mcp.dolibarr_mcp_server` |
| Run server (HTTP/remote) | `MCP_TRANSPORT=http MCP_HTTP_PORT=8080 python -m dolibarr_mcp.dolibarr_mcp_server` |
| Test connection | `python -m dolibarr_mcp.test_connection` |
| Run tests | `pytest` |
| Tests with coverage | `pytest --cov=src/dolibarr_mcp --cov-report=term-missing` |
| Integration tests only | `pytest -m integration` |
| Docker build & run | `docker compose up -d` |
| Docker test profile | `docker compose --profile test up dolibarr-mcp-test` |

## Architecture

```
src/dolibarr_mcp/
├── dolibarr_mcp_server.py  # MCP tool declarations + dispatcher + transport setup
├── dolibarr_client.py      # Async Dolibarr REST API client (aiohttp)
├── config.py               # Pydantic Settings config with env/alias support
├── cli.py                  # CLI entrypoint (dolibarr-mcp command)
├── test_connection.py      # Standalone connection validator
└── testing.py              # Test utilities
```

- **Transport**: STDIO (local) or streamable HTTP (remote/hosted). Selected via `MCP_TRANSPORT` env var.
- **HTTP stack**: Starlette app → StreamableHTTPSessionManager → Uvicorn. CORS enabled.
- **All handlers are async**. The client uses `aiohttp.ClientSession` with `DOLAPIKEY` header.

## Required Environment Variables

| Variable | Purpose |
|----------|---------|
| `DOLIBARR_URL` | Dolibarr API base URL (e.g. `https://erp.example.com/api/index.php`) |
| `DOLIBARR_API_KEY` | API authentication key |
| `MCP_TRANSPORT` | `stdio` (default) or `http` for remote hosting |
| `MCP_HTTP_PORT` | HTTP listener port (default 8080) |

See [docs/configuration.md](docs/configuration.md) for all optional variables (LOG_LEVEL, DEBUG_MODE, MAX_RETRIES, etc.).

## Key Conventions

1. **Logging to stderr only** — stdout is reserved for MCP protocol messages in stdio mode.
2. **Custom exceptions** — `DolibarrAPIError` and `DolibarrValidationError` in `dolibarr_client.py`. Always use these for API error paths.
3. **Field mapping** — The client normalizes friendly field names to Dolibarr's internal names (e.g. `customer_id` → `socid`, `product_id` → `fk_product`).
4. **Search-first design** — Prefer search tools with `sqlfilter` over listing all records. Escape filter values.
5. **Config resilience** — URL normalization, fallback aliases (`DOLIBARR_SHOP_URL`, `DOLIBARR_BASE_URL`), non-fatal warnings on missing config.
6. **Tool dispatcher pattern** — All MCP tools are declared in `list_tools`, routed through a single `call_tool` dispatcher function.

## Testing Patterns

- **Unit tests**: Mock `aiohttp.ClientSession.request` or patch `DolibarrClient` methods. Use `@pytest.mark.asyncio`.
- **Integration tests**: Marked with `@pytest.mark.integration`. Some use a local fake Dolibarr API via `aiohttp.web`.
- **Coverage**: Enforced by default in pytest config (see `pyproject.toml` addopts).

## Deployment (Remote/Hosted)

The server runs as a Docker container exposing HTTP MCP transport:

- Docker Compose maps host port (default `18004`) → container port `8080`
- Healthcheck uses `python -m dolibarr_mcp.test_connection`
- Set `MCP_TRANSPORT=http` in container environment

See [docker-compose.yml](docker-compose.yml) and [docker/Dockerfile](docker/Dockerfile) for full setup.

## Documentation

- [docs/quickstart.md](docs/quickstart.md) — Getting started
- [docs/configuration.md](docs/configuration.md) — All config options
- [docs/api-reference.md](docs/api-reference.md) — MCP tool reference
- [docs/development.md](docs/development.md) — Development workflow
