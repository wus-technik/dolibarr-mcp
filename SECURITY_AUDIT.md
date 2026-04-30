# Security Audit Report — Dolibarr MCP Server

**Date:** 2026-04-30  
**Scope:** Full codebase review (`src/dolibarr_mcp/`, `docker/`, configuration files)  
**Methodology:** Manual static analysis against OWASP Top 10 (2021)

---

## HIGH Severity

### 1. No Authentication on MCP HTTP Transport (CWE-306)

**Location:** `src/dolibarr_mcp/dolibarr_mcp_server.py` — `_build_http_app()` / `_run_http_server()`

The HTTP server exposes all MCP tools (full CRUD over ERP data) without any authentication. Anyone who can reach the port can create/delete invoices, users, customers, etc.

**Recommendation:** Add an authentication middleware (e.g., bearer token check, mTLS, or API key header) before the session manager handles requests.

---

### 2. Wildcard CORS — `allow_origins=["*"]` (CWE-942)

**Location:** `src/dolibarr_mcp/dolibarr_mcp_server.py` — `_build_http_app()`

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Any website can make cross-origin requests. Combined with finding #1, a malicious web page visited by an operator on the same network could invoke destructive tools.

**Recommendation:** Restrict `allow_origins` to known MCP client origins, or make it configurable via an env var with a safe default.

---

### 3. SQL Filter Injection via Insufficient Escaping (CWE-943)

**Location:** `src/dolibarr_mcp/dolibarr_mcp_server.py` — `_escape_sqlfilter()`

```python
def _escape_sqlfilter(value: str) -> str:
    """Escape single quotes for SQL filters."""
    return value.replace("'", "''")
```

This only escapes single quotes. Dolibarr's `sqlfilters` parameter uses a structural syntax like `(t.ref:like:'%value%')`. An attacker can inject:
- `)` to break out of the filter clause
- Additional `OR`/`AND` conditions
- `%` and `_` wildcard abuse for information disclosure

The injected values flow directly into search tool handlers (search_products_by_ref, search_customers, search_products_by_label, search_projects).

**Recommendation:** Validate input against an allowlist pattern (alphanumeric + limited punctuation), or use a proper parameterized approach. At minimum, also escape `)`, `(`, `:`, and `'`.

---

### 4. Unrestricted `dolibarr_raw_api` Tool (CWE-284)

**Location:** `src/dolibarr_mcp/dolibarr_client.py` — `dolibarr_raw_api()`

This tool allows any MCP client to call *arbitrary* Dolibarr REST endpoints with any HTTP method. No endpoint allowlist, no method restrictions beyond GET/POST/PUT/DELETE.

A compromised or rogue AI agent could delete all records, access admin endpoints, or exfiltrate data.

**Recommendation:** Either remove this tool in production, add an endpoint allowlist, or gate it behind an explicit operator confirmation flag.

---

## MEDIUM Severity

### 5. Binding to `0.0.0.0` by Default (CWE-668)

**Location:** `src/dolibarr_mcp/config.py`

```python
mcp_http_host: str = Field(default="0.0.0.0")
```

Exposes the unauthenticated server to all network interfaces by default.

**Recommendation:** Default to `127.0.0.1` for local-only access. Operators who need network exposure can override explicitly.

---

### 6. API Key Accepted via CLI Argument (CWE-214)

**Location:** `src/dolibarr_mcp/cli.py`

```python
@click.option("--api-key", help="Dolibarr API key")
```

Command-line arguments appear in process listings (`ps aux`), shell history, and audit logs.

**Recommendation:** Accept the key only via environment variable or file. If CLI support is kept, document the risk and consider reading from stdin.

---

### 7. No TLS — Cleartext API Key Transmission (CWE-319)

**Locations:**
- `src/dolibarr_mcp/dolibarr_client.py` — `DOLAPIKEY` header sent over HTTP
- `src/dolibarr_mcp/config.py` — accepts `http://` Dolibarr URLs
- `src/dolibarr_mcp/dolibarr_mcp_server.py` — uvicorn runs without SSL context

The MCP HTTP server runs plain HTTP. The config validator also accepts `http://` Dolibarr URLs, meaning the `DOLAPIKEY` header is sent in cleartext.

**Recommendation:**
- Warn or reject `http://` URLs for the upstream Dolibarr API in production.
- Support TLS termination (either via uvicorn ssl params or document a reverse proxy requirement).

---

### 8. Exception Details Leaked to Client (CWE-209)

**Location:** `src/dolibarr_mcp/dolibarr_mcp_server.py` — `handle_call_tool()` exception handler

```python
error_result = {
    "message": f"Tool execution failed: {str(e)}",
    ...
}
```

Internal exception messages (which may contain file paths, connection strings, or stack details) are returned verbatim to the MCP client.

**Recommendation:** Return a generic message to the client; log the full exception server-side only.

---

### 9. No Rate Limiting (CWE-770)

No request throttling on the HTTP transport or tool dispatch. A compromised client can flood the Dolibarr API with unlimited requests.

**Recommendation:** Add request-rate middleware (e.g., `slowapi`, or a simple token-bucket per session).

---

### 10. Placeholder Credentials Silently Accepted (CWE-1188)

**Location:** `src/dolibarr_mcp/config.py` — `validate_api_key()`

```python
return "placeholder_api_key"
```

The server starts and accepts connections even with dummy credentials. An operator may not realize the system is effectively non-functional but still network-reachable.

**Recommendation:** In HTTP mode, refuse to start if credentials are placeholder values.

---

## LOW Severity

### 11. No Input Length Validation

Tool arguments (e.g., search queries, descriptions, line items) have no `maxLength` constraints in their JSON schemas. Very large inputs could cause resource exhaustion downstream.

**Recommendation:** Add `maxLength` constraints to string-type tool parameters.

---

### 12. Dependency Version Ranges Too Loose

**Location:** `requirements.txt` / `pyproject.toml`

```
aiohttp>=3.9.0
pydantic>=2.5.0
```

No upper bounds means a future vulnerable release could be installed automatically.

**Recommendation:** Pin dependencies to known-good ranges (e.g., `aiohttp>=3.9.0,<4.0`) or use a lockfile.

---

### 13. Docker — Broad `/usr/local/bin/` Copy

**Location:** `docker/Dockerfile`

```dockerfile
COPY --from=builder /usr/local/bin/ /usr/local/bin/
```

Copies the entire bin directory from builder stage, which may include unnecessary tools that increase the attack surface.

**Recommendation:** Copy only the specific entrypoints needed.

---

### 14. Deprecated `datetime.utcnow()` Usage

**Location:** `src/dolibarr_mcp/dolibarr_client.py` — `_now_iso()`, `_generate_reference()`

Deprecated since Python 3.12. While not a security vulnerability per se, deprecated APIs may be removed and could lead to unexpected behavior in future runtimes.

**Recommendation:** Use `datetime.now(datetime.UTC)` instead.

---

## Summary Table

| # | Severity | Issue | OWASP Category |
|---|----------|-------|----------------|
| 1 | **HIGH** | No auth on HTTP transport | A07 - Authentication Failures |
| 2 | **HIGH** | Wildcard CORS | A01 - Broken Access Control |
| 3 | **HIGH** | SQL filter injection | A03 - Injection |
| 4 | **HIGH** | Unrestricted raw API tool | A01 - Broken Access Control |
| 5 | MEDIUM | Bind `0.0.0.0` by default | A05 - Security Misconfiguration |
| 6 | MEDIUM | API key via CLI argument | A07 - Authentication Failures |
| 7 | MEDIUM | No TLS / cleartext API key | A02 - Cryptographic Failures |
| 8 | MEDIUM | Exception details leaked | A04 - Insecure Design |
| 9 | MEDIUM | No rate limiting | A04 - Insecure Design |
| 10 | MEDIUM | Placeholder creds accepted | A05 - Security Misconfiguration |
| 11 | LOW | No input length validation | A04 - Insecure Design |
| 12 | LOW | Loose dependency pinning | A06 - Vulnerable Components |
| 13 | LOW | Broad Docker bin copy | A05 - Security Misconfiguration |
| 14 | LOW | Deprecated datetime usage | — |

---

## Priority Remediation Plan

1. **Add authentication middleware** to the HTTP transport (bearer token or similar).
2. **Restrict CORS origins** to configured allowlist.
3. **Harden `_escape_sqlfilter`** to reject structural characters or use allowlist validation.
4. **Gate or remove `dolibarr_raw_api`** in production deployments.
5. **Default bind to `127.0.0.1`** and require explicit opt-in for network exposure.
