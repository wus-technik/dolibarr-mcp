# TODO: Per-User Credential Authentication

## Goal

Replace the current shared/server-level `DOLIBARR_API_KEY` with per-user credentials so that each MCP session uses the connecting user's Dolibarr API key, inheriting their Dolibarr permissions.

## Current State

- A single `DOLIBARR_API_KEY` env var is used for all requests (configured in `config.py`)
- `DolibarrClient` is instantiated per tool call in the dispatcher with the shared config (`dolibarr_mcp_server.py:1146`)
- The HTTP session header `DOLAPIKEY` is set once at client session creation (`dolibarr_client.py:65`)
- No authentication layer exists on the MCP HTTP transport itself
- Dolibarr natively supports per-user API keys — each user has their own `DOLAPIKEY` that enforces their permissions server-side

## Required Changes

### 1. MCP HTTP Transport Authentication Layer
- [ ] Add middleware to the Starlette app that extracts the user's Dolibarr API key from the incoming MCP HTTP request
- [ ] Decide on credential passing mechanism:
  - **Option A: HTTP Header** — Client sends `Authorization: Bearer <dolibarr_api_key>` or custom header (e.g. `X-Dolibarr-API-Key`)
  - **Option B: MCP initialization params** — Pass API key during MCP `initialize` handshake via `clientInfo` or custom metadata
  - **Option C: OAuth2 proxy** — External OAuth2 flow that maps tokens to Dolibarr API keys (more complex, better for web UIs)
- [ ] Reject unauthenticated requests with appropriate error response

### 2. Per-Session Client Instantiation
- [ ] Store the user's API key in the MCP session context (available to tool handlers)
- [ ] Modify `handle_call_tool` dispatcher to create `DolibarrClient` with the session-specific API key instead of the global config
- [ ] Allow `DolibarrClient.__init__` to accept an explicit `api_key` parameter override (independent of `Config`)

### 3. Config Changes
- [ ] Make `DOLIBARR_API_KEY` optional (no longer required for hosted multi-user mode)
- [ ] Add config flag: `MCP_AUTH_MODE` = `shared` (current behavior) | `per_user` (new behavior)
- [ ] When `per_user`, server starts without a Dolibarr API key and expects each session to provide one
- [ ] Keep `shared` mode as default for backward compatibility

### 4. Session Context Plumbing
- [ ] Investigate MCP SDK's `RequestContext` / session metadata — determine how to thread per-request data into `call_tool` handlers
- [ ] If SDK doesn't natively support request context in tool handlers, use Starlette middleware + contextvars to propagate the API key from HTTP request → tool handler
- [ ] Ensure session isolation: one user's key never leaks to another session

### 5. DolibarrClient Refactoring
- [ ] Decouple `DolibarrClient` from `Config` for the API key — accept `base_url` + `api_key` as constructor params
- [ ] Keep config for non-credential settings (timeouts, retries, debug mode, ref autogen)
- [ ] Consider connection pooling: reuse `aiohttp.ClientSession` per unique API key within a session, close on session end

### 6. Security Considerations
- [ ] Never log user API keys (already partially handled by `_mask_api_key`)
- [ ] Validate API key format before forwarding to Dolibarr
- [ ] Rate-limit authentication attempts
- [ ] Consider TLS requirement enforcement for HTTP transport when `per_user` auth is active
- [ ] Add CORS restrictions (currently `allow_origins=["*"]`) — tighten for production
- [ ] Session timeout / key expiry handling

### 7. Testing
- [ ] Unit tests: mock per-session client creation, verify correct API key is passed
- [ ] Integration test: two concurrent sessions with different API keys get different permissions
- [ ] Test fallback: `shared` mode still works unchanged
- [ ] Test error paths: missing key, invalid key, expired key

### 8. Documentation
- [ ] Update `docs/configuration.md` with new `MCP_AUTH_MODE` variable
- [ ] Document client-side setup: how MCP clients pass their Dolibarr API key
- [ ] Add security recommendations section (TLS, key rotation, least-privilege Dolibarr users)

## Implementation Order (Suggested)

1. **Config changes** — Add `MCP_AUTH_MODE`, make API key optional
2. **DolibarrClient refactor** — Accept explicit api_key param
3. **Session context** — Implement contextvars-based key propagation
4. **HTTP middleware** — Extract and validate user credentials
5. **Dispatcher update** — Use session API key in `handle_call_tool`
6. **Tests** — Cover both modes
7. **Docs** — Update configuration and security docs

## Notes

- Dolibarr enforces permissions server-side per API key, so no additional permission logic needed in the MCP server
- The `login` API endpoint (`POST /api/index.php/login`) can exchange username+password for an API token — could be used for initial key retrieval
- `DOLAPIENTITY` header can also be forwarded per-user for multi-company setups
- Consider whether STDIO mode needs per-user support (probably not — it's single-user by nature)
