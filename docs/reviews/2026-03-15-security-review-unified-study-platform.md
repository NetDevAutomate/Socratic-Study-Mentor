# Security Review: Unified Study Platform Plan

**Date:** 2026-03-15
**Reviewer:** Security Sentinel (Application Security Audit)
**Target:** `docs/plans/2026-03-15-feat-unified-study-platform-plan.md`
**Scope:** Pre-implementation security review of planned architecture and identified risk areas

---

## Executive Summary

The plan transforms a localhost-only study tool into a **LAN-accessible web application** with file serving, config editing, credential storage, and subprocess execution. This fundamentally changes the threat model from "single-user local tool" to "multi-client network service." The plan demonstrates good security awareness in some areas (directory traversal tests are planned, WAL mode for SQLite, bcrypt for password hashing) but has **critical gaps** in credential management, transport security, and input validation that need addressing before implementation.

**Overall Risk Rating: MEDIUM-HIGH** (for a LAN-only personal tool)

If this were internet-facing, it would be CRITICAL. The LAN context reduces but does not eliminate risk -- any compromised device on the WiFi becomes an attacker.

---

## Risk Matrix

| ID | Finding | Severity | OWASP Category | Exploitability |
|----|---------|----------|----------------|----------------|
| S-01 | GitHub PAT in plaintext config.yaml | **CRITICAL** | A02:2021 Cryptographic Failures | Trivial |
| S-02 | HTTP Basic Auth over cleartext HTTP | **HIGH** | A02:2021 Cryptographic Failures | Easy (WiFi sniffing) |
| S-03 | Config editor allows path modification over LAN | **HIGH** | A01:2021 Broken Access Control | Easy |
| S-04 | Directory traversal in artefact serving | **MEDIUM** | A01:2021 Broken Access Control | Medium (plan has mitigation) |
| S-05 | Subprocess calls with user-influenced input | **MEDIUM** | A03:2021 Injection | Medium |
| S-06 | CORS wildcard `Access-Control-Allow-Origin: *` | **MEDIUM** | A05:2021 Security Misconfiguration | Easy |
| S-07 | No rate limiting on auth endpoints | **MEDIUM** | A07:2021 Identification Failures | Easy |
| S-08 | Service worker cache staleness | **LOW** | A05:2021 Security Misconfiguration | Low |
| S-09 | SQLite concurrent access from LAN | **LOW** | A04:2021 Insecure Design | Low (WAL helps) |
| S-10 | MCP server transport security | **LOW** | A05:2021 Security Misconfiguration | Low (stdio) |
| S-11 | Error messages leak internal state | **MEDIUM** | A04:2021 Insecure Design | Easy |
| S-12 | No input validation on API POST bodies | **MEDIUM** | A03:2021 Injection | Medium |
| S-13 | Feedback endpoint as open relay to GitHub API | **MEDIUM** | A01:2021 Broken Access Control | Easy |

---

## Detailed Findings

### S-01: GitHub Personal Access Token Stored in Plaintext Config (CRITICAL)

**Location in plan:** Phase 7 -- `feedback.github_token: ghp_...` in `config.yaml`

**The problem:** The plan stores a GitHub PAT directly in `~/.config/studyctl/config.yaml`. This token has repository write access (creating issues). The same config file is:
- Readable by any process running as the user
- Served via the config GET endpoint (`/api/config`)
- Editable via the config PUT endpoint
- Potentially backed up, synced, or committed accidentally

**Exploitation:** Any LAN client hitting `GET /api/config` retrieves the token. Even if you filter "safe fields," a bug in the allowlist exposes it. The token can then create issues, read private repos, or worse depending on scope.

**Remediation:**

1. **Never store the PAT in config.yaml.** Use an environment variable (`STUDYCTL_GITHUB_TOKEN`) or a system keyring:
   ```python
   import os

   def get_github_token() -> str | None:
       # Priority: env var > keyring > None
       token = os.environ.get("STUDYCTL_GITHUB_TOKEN")
       if token:
           return token
       try:
           import keyring
           return keyring.get_password("studyctl", "github_token")
       except ImportError:
           return None
   ```

2. **If env var is the only option**, document it clearly and ensure the config GET endpoint NEVER returns any key containing `token`, `secret`, `key`, or `password`.

3. **Add a `.gitignore` pattern** for `config.yaml` if not already present.

4. **Scope the PAT minimally:** `public_repo` only (issues on public repos). Document this requirement.

---

### S-02: HTTP Basic Auth Credentials Sent in Cleartext (HIGH)

**Location in plan:** Phase 7 -- `--password` flag, HTTP Basic Auth via `Depends(verify_password)`

**The problem:** HTTP Basic Auth sends credentials as base64-encoded (NOT encrypted) text in every request. On a WiFi LAN, any device running Wireshark or tcpdump captures the password trivially. The plan acknowledges this with a warning banner, but a warning is not a control.

**Exploitation:** Passive WiFi sniffing captures the base64 `Authorization` header. Decode it, access the app as the user.

**Remediation:**

1. **Add self-signed TLS as the default for LAN mode:**
   ```python
   from pathlib import Path
   import ssl

   def get_or_create_self_signed_cert(cert_dir: Path) -> tuple[Path, Path]:
       """Generate self-signed cert if none exists. Returns (cert_path, key_path)."""
       cert_path = cert_dir / "studyctl.pem"
       key_path = cert_dir / "studyctl-key.pem"
       if not cert_path.exists():
           # Use cryptography library or subprocess openssl
           ...
       return cert_path, key_path

   # In uvicorn startup:
   uvicorn.run(app, host=host, port=port,
               ssl_certfile=str(cert_path),
               ssl_keyfile=str(key_path))
   ```

2. **At minimum, when `--password` is set, refuse to start without `--https` or `--accept-http-risk`:**
   ```python
   if password and not https:
       click.echo("ERROR: --password requires --https (or --accept-http-risk to override)")
       raise SystemExit(1)
   ```

3. **Consider token-based auth instead of Basic Auth.** Issue a session cookie after initial auth so the password is only sent once, not on every request.

4. **Bind to localhost by default, require explicit `--host 0.0.0.0` for LAN.** The current plan defaults to `0.0.0.0` which exposes the server to the entire network by default.

---

### S-03: Config Editor Accessible Over LAN (HIGH)

**Location in plan:** Phase 2/3 -- `PUT /api/config`, Settings page with HTMX

**The problem:** The config editor allows modifying `content.base_path`, `review.directories`, and other path settings. An attacker on the LAN (or anyone who guesses/sniffs the password) can:
- Change `content.base_path` to point at sensitive directories (e.g., `~/.ssh/`)
- Then use the artefact serving endpoint to read arbitrary files
- Modify `review.directories` to inject paths
- If config includes the `feedback` section, potentially exfiltrate or modify the GitHub PAT

**The plan mentions "safe fields only" but does not define the allowlist or validation rules.**

**Remediation:**

1. **Define a strict allowlist with type validation in code, not just documentation:**
   ```python
   CONFIG_SAFE_FIELDS = {
       "review.default_mode": {"type": str, "allowed": ["flashcards", "quiz"]},
       "review.shuffle": {"type": bool},
       "review.voice_enabled": {"type": bool},
       "tui.theme": {"type": str, "max_length": 50, "pattern": r"^[a-z_]+$"},
       "tui.dyslexic_friendly": {"type": bool},
   }
   # EXCLUDED from safe fields: ALL path fields, ALL credential fields
   ```

2. **Path fields (`content.base_path`, `review.directories`) must NOT be editable via the web API.** These should only be configurable via CLI (`studyctl setup`) running as the local user. The risk of a LAN client pointing the artefact server at `/etc/` or `~/.ssh/` is too high.

3. **If path editing is truly needed via web UI**, validate against an allowlist of parent directories and require re-authentication.

4. **Never expose credential fields** (`feedback.github_token`, `web.password_hash`) via GET or PUT.

---

### S-04: Directory Traversal in Artefact File Serving (MEDIUM)

**Location in plan:** Phase 2 -- `GET /api/artefacts/{course}/{type}/{filename}`

**The good news:** The plan includes a `validate_artefact_path()` function with `resolve()` and `is_relative_to()` checks, and plans dedicated tests (`test_web_artefacts.py`). This is the right approach.

**Remaining risks:**

1. **Symlink following:** `resolve()` follows symlinks. If an attacker can create a symlink inside the content directory (e.g., via the content pipeline or a crafted PDF filename), they can escape the base path. The `is_relative_to()` check on the resolved path mitigates this, but only if checked AFTER resolve.

2. **URL encoding bypass:** Ensure the framework decodes `%2e%2e%2f` (URL-encoded `../`) before path construction. FastAPI's path parameters generally handle this, but double-encoding (`%252e%252e%252f`) can bypass naive checks.

3. **Null byte injection:** Python 3 Path handles null bytes safely (raises ValueError), but verify this is caught.

4. **The `course` and `type` path parameters are not validated.** An attacker could send `course=../../etc&type=passwd&filename=shadow`.

**Remediation:**

```python
import re

SAFE_PATH_COMPONENT = re.compile(r'^[a-zA-Z0-9][a-zA-Z0-9._\- ]{0,200}$')

def validate_artefact_path(course: str, artefact_type: str, filename: str) -> Path:
    # Step 1: Validate each component individually (reject traversal attempts early)
    for component, name in [(course, "course"), (artefact_type, "type"), (filename, "filename")]:
        if not SAFE_PATH_COMPONENT.match(component):
            raise HTTPException(status_code=400, detail=f"Invalid {name}")

    # Step 2: Construct and resolve
    base = settings.content.base_path.resolve()
    target = (base / course / artefact_type / filename).resolve()

    # Step 3: Verify containment (defense in depth)
    if not target.is_relative_to(base):
        raise HTTPException(status_code=404)

    # Step 4: Verify it's a regular file (not a directory, device, etc.)
    if not target.is_file():
        raise HTTPException(status_code=404)

    # Step 5: Validate file extension against allowed types
    ALLOWED_EXTENSIONS = {'.mp3', '.wav', '.ogg', '.mp4', '.webm', '.pdf', '.png', '.jpg', '.svg'}
    if target.suffix.lower() not in ALLOWED_EXTENSIONS:
        raise HTTPException(status_code=403, detail="File type not allowed")

    return target
```

Add test cases:
- `../../../etc/passwd`
- `....//....//etc/passwd` (double dot-slash)
- URL-encoded variants
- Null bytes: `file%00.pdf`
- Symlinks pointing outside base
- Filenames with spaces, unicode, and special characters

---

### S-05: Subprocess Command Injection Risk (MEDIUM)

**Location in plan:** Phase 1 -- `check_content_dependencies()` for pandoc, mmdc, typst; content pipeline execution

**Existing code (study-speak-server.py):**
```python
subprocess.run([str(_SPEAK_BIN), text], check=True, timeout=120, capture_output=True)
```

**The good:** The existing MCP server uses list-form `subprocess.run()` (not `shell=True`), which prevents shell injection. The `text` parameter is passed as a single argument.

**The risks:**
1. The plan absorbs `pdf-by-chapters` which likely calls `pandoc`, `mmdc`, and `typst` as subprocesses. If filenames from PDF TOC entries are used as command arguments without sanitization, injection is possible.
2. The `--ranges` flag for `content split` takes user input that could be malformed.
3. If any subprocess call uses `shell=True` or string formatting for commands, it becomes injectable.

**Remediation:**

1. **Never use `shell=True` with user-influenced arguments.** Always use list-form:
   ```python
   # GOOD
   subprocess.run(["pandoc", input_file, "-o", output_file], check=True)

   # BAD - shell injection via filename
   subprocess.run(f"pandoc {input_file} -o {output_file}", shell=True)
   ```

2. **Validate the `--ranges` flag input:**
   ```python
   RANGE_PATTERN = re.compile(r'^(\d+-\d+)(,\d+-\d+)*$')
   if not RANGE_PATTERN.match(ranges):
       raise click.BadParameter("Ranges must be like '1-30,31-60'")
   ```

3. **Sanitize filenames derived from PDF TOC entries** before using them as filesystem paths or subprocess arguments. Strip or replace characters outside `[a-zA-Z0-9._\- ]`.

4. **Set `timeout` on all subprocess calls** (already done in speak server -- maintain this pattern).

---

### S-06: CORS Wildcard Header (MEDIUM)

**Location in existing code:** `server.py` line in `_json_response()`:
```python
self.send_header("Access-Control-Allow-Origin", "*")
```

**The problem:** `Access-Control-Allow-Origin: *` allows any website to make requests to the study API from a user's browser. If a user visits a malicious website while the study server is running, that site can:
- Read all flashcard content
- Submit fake review data
- Access the config API (if authenticated via cookies/basic auth stored in browser)

**Remediation:**

1. **Remove the wildcard CORS header.** For a LAN app where the client and server are same-origin, CORS headers are not needed at all.

2. **If cross-origin access is needed** (e.g., PWA on a different port), restrict to the specific origin:
   ```python
   from fastapi.middleware.cors import CORSMiddleware

   app.add_middleware(
       CORSMiddleware,
       allow_origins=[f"http://localhost:{port}", f"http://{hostname}:{port}"],
       allow_methods=["GET", "POST", "PUT"],
       allow_headers=["Authorization", "Content-Type"],
   )
   ```

---

### S-07: No Rate Limiting on Auth or API Endpoints (MEDIUM)

**Location in plan:** Phase 7 -- auth middleware, Phase 2 -- all API endpoints

**The problem:** Without rate limiting, an attacker on the LAN can:
- Brute-force the HTTP Basic Auth password
- Flood the SQLite database with fake review data
- Spam the GitHub Issues API via the feedback endpoint
- DoS the server with rapid requests

**Remediation:**

1. **Add rate limiting middleware for FastAPI:**
   ```python
   from slowapi import Limiter
   from slowapi.util import get_remote_address

   limiter = Limiter(key_func=get_remote_address)

   @app.post("/api/review")
   @limiter.limit("60/minute")
   async def review(...): ...

   @app.post("/api/feedback")
   @limiter.limit("5/hour")
   async def feedback(...): ...
   ```

2. **For auth specifically, implement exponential backoff or account lockout** after N failed attempts:
   ```python
   # In-memory failure tracking (resets on server restart -- acceptable for LAN tool)
   _auth_failures: dict[str, list[float]] = {}
   MAX_FAILURES = 5
   LOCKOUT_SECONDS = 300
   ```

---

### S-08: Service Worker Cache Staleness (LOW)

**Location in plan:** Phase 2 -- "Update service worker cache version in sw.js on each release (manual bump)"

**Existing code (`sw.js`):**
```javascript
const CACHE = "studyctl-v1";
```

**The problem:** The current service worker uses a cache-first strategy for all non-API requests. If the cache version is not bumped on update, users will see stale HTML/JS/CSS indefinitely. The "manual bump" approach is error-prone.

**Security implications are low** (stale UI, not data corruption), but stale JavaScript could mean:
- Missing security fixes in client-side code
- Incompatible API calls if endpoints change

**Remediation:**

1. **Auto-generate cache version from package version:**
   ```javascript
   // Injected by Jinja2 template
   const CACHE = "studyctl-{{ version }}";
   ```

2. **Add a `stale-while-revalidate` strategy** for HTML pages, cache-first only for immutable assets:
   ```javascript
   self.addEventListener("fetch", (e) => {
     if (e.request.url.includes("/api/")) return; // Network-only for API
     if (e.request.url.match(/\.(css|js|woff2|png|svg)$/)) {
       // Cache-first for static assets
       e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
     } else {
       // Network-first for HTML (stale-while-revalidate)
       e.respondWith(
         fetch(e.request)
           .then(r => { caches.open(CACHE).then(c => c.put(e.request, r.clone())); return r; })
           .catch(() => caches.match(e.request))
       );
     }
   });
   ```

3. **Add a `activate` event handler** that deletes old caches:
   ```javascript
   self.addEventListener("activate", (e) => {
     e.waitUntil(
       caches.keys().then(keys =>
         Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)))
       )
     );
   });
   ```

---

### S-09: SQLite Concurrent Access from Multiple LAN Clients (LOW)

**Location in plan:** Phase 2 -- WAL mode for concurrent access

**The good:** The plan explicitly enables WAL mode, which is the correct approach for concurrent readers with occasional writers.

**Remaining risks:**

1. **WAL mode does not support concurrent writes from different machines accessing the same file over NFS/SMB.** SQLite's locking relies on POSIX file locks which are unreliable over network filesystems. If the database is on a NAS, this will corrupt data.

2. **Connection handling:** Each API request should get its own connection or use a connection pool with proper cleanup.

**Remediation:**

1. **Document that the SQLite database must be on a local filesystem**, not a network share:
   ```python
   def validate_db_path(db_path: Path) -> None:
       """Warn if database appears to be on a network filesystem."""
       import shutil
       # Basic heuristic: check if path is on a known network mount
       # More robust: check filesystem type via os.statvfs
   ```

2. **Use a connection-per-request pattern with proper cleanup:**
   ```python
   from contextlib import contextmanager

   @contextmanager
   def get_db_connection(db_path: Path):
       conn = sqlite3.connect(db_path, timeout=10)
       conn.execute("PRAGMA journal_mode=WAL")
       conn.execute("PRAGMA busy_timeout=5000")  # Wait up to 5s for locks
       try:
           yield conn
       finally:
           conn.close()
   ```

3. **Set `busy_timeout`** to prevent immediate `SQLITE_BUSY` errors under concurrent access.

---

### S-10: MCP Server Transport Security (LOW)

**Location in plan:** Phase 4 -- MCP server via `studyctl-mcp`

**The good:** The existing `study-speak-server.py` and the planned `studyctl-mcp` both use **stdio transport** (not HTTP/WebSocket). This means the MCP server communicates only via stdin/stdout with its parent process. There is no network listener.

**The risk is minimal** because:
- Stdio transport cannot be accessed from the network
- The MCP SDK's `mcp.run()` defaults to stdio
- Only the parent process (Claude Code, etc.) can invoke tools

**Residual risk:** If someone adds HTTP/SSE transport in the future, the MCP tools (which can read files, modify config, generate content) would be network-accessible.

**Remediation:**

1. **Add a comment/guard in the MCP server code:**
   ```python
   # SECURITY: This server MUST use stdio transport only.
   # Never expose via HTTP/SSE -- tools have filesystem access.
   if __name__ == "__main__":
       mcp.run(transport="stdio")
   ```

2. **If HTTP transport is ever needed**, add authentication and restrict to localhost.

---

### S-11: Error Messages Leak Internal State (MEDIUM)

**Location in existing code:** `server.py`:
```python
except (KeyError, Exception) as exc:
    self._json_response({"error": str(exc)}, 400)
```

**The problem:** Catching broad `Exception` and returning `str(exc)` can leak:
- File paths (FileNotFoundError)
- Database schema details (sqlite3.OperationalError)
- Stack traces or internal variable names
- Configuration values

**Remediation:**

```python
import logging

logger = logging.getLogger(__name__)

# In error handlers:
except KeyError as exc:
    self._json_response({"error": f"Missing required field: {exc}"}, 400)
except Exception:
    logger.exception("Unexpected error in review handler")
    self._json_response({"error": "Internal server error"}, 500)
```

Log the full exception server-side, return only generic messages to clients.

---

### S-12: No Input Validation on API POST Bodies (MEDIUM)

**Location in existing code:** `_handle_review()`, `_handle_session()` -- direct `body["key"]` access

**The problem:** The existing code reads POST body fields directly without validation:
- No type checking (what if `correct` is a string instead of bool?)
- No length limits (what if `course` is a 10MB string?)
- No format validation (what if `card_hash` contains SQL?)

The plan mentions FastAPI + Pydantic but does not show request models.

**Remediation:**

Define Pydantic models for every POST endpoint:

```python
from pydantic import BaseModel, Field, constr

class ReviewRequest(BaseModel):
    course: constr(min_length=1, max_length=200, pattern=r'^[a-zA-Z0-9._\- ]+$')
    card_type: Literal["flashcard", "quiz"] = "flashcard"
    card_hash: constr(min_length=1, max_length=64, pattern=r'^[a-f0-9]+$')
    correct: bool
    response_time_ms: int | None = Field(None, ge=0, le=3600000)

class SessionRequest(BaseModel):
    course: constr(min_length=1, max_length=200)
    mode: Literal["flashcards", "quiz"] = "flashcards"
    total: int = Field(..., ge=0, le=10000)
    correct: int = Field(..., ge=0, le=10000)
    duration_seconds: int | None = Field(None, ge=0, le=86400)

class FeedbackRequest(BaseModel):
    category: Literal["bug", "feature-request", "ux-feedback"]
    description: constr(min_length=10, max_length=5000)
    # No screenshot upload -- see S-13
```

---

### S-13: Feedback Endpoint as Open Relay to GitHub API (MEDIUM)

**Location in plan:** Phase 7 -- `POST /api/feedback` creates GitHub Issues

**The problem:** If the feedback endpoint is accessible without auth (or with shared LAN auth), any LAN client can:
- Create unlimited GitHub Issues on the configured repo
- Fill them with spam, offensive content, or phishing links
- The issues are created under the PAT owner's identity

**Remediation:**

1. **Rate limit aggressively:** 3 issues per hour maximum.
2. **Require auth even if no `--password` is set** (separate feedback auth or CAPTCHA).
3. **Input sanitization:** Strip HTML/markdown links from description, limit length.
4. **Consider a queue/review model:** Save feedback locally first, let the user push to GitHub via CLI.
5. **Add a client-side confirmation step** before submission.

---

## Security Requirements Checklist

- [ ] **All inputs validated and sanitized** -- FAIL: No Pydantic models defined (S-12). Plan mentions FastAPI but shows no request validation.
- [ ] **No hardcoded secrets or credentials** -- FAIL: GitHub PAT in config.yaml (S-01). Password hash in config is acceptable if permissions are correct.
- [ ] **Proper authentication on all endpoints** -- PARTIAL: Auth planned but optional, no rate limiting (S-07).
- [ ] **SQL queries use parameterization** -- PASS: Existing code uses parameterized queries. Some f-string SQL in sync.py uses table names (not user input).
- [ ] **XSS protection implemented** -- NOT ASSESSED: Jinja2 auto-escapes by default, but HTMX fragments need manual review at implementation time.
- [ ] **HTTPS enforced where needed** -- FAIL: No TLS support planned (S-02).
- [ ] **CSRF protection enabled** -- PARTIAL: FastAPI does not include CSRF middleware by default. HTMX requests need CSRF tokens.
- [ ] **Security headers properly configured** -- NOT PLANNED: No mention of `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`, `Strict-Transport-Security`.
- [ ] **Error messages don't leak sensitive information** -- FAIL: Broad exception stringification (S-11).
- [ ] **Dependencies are up-to-date and vulnerability-free** -- NOT ASSESSED: No `pip-audit` or `safety` in CI pipeline.

---

## Remediation Roadmap (Priority Order)

### Immediate (Before Phase 2 Implementation)

1. **S-01:** Remove GitHub PAT from config.yaml plan. Use environment variable or keyring. Update Phase 7 design.
2. **S-02:** Change default bind from `0.0.0.0` to `127.0.0.1`. Add `--lan` flag for explicit network exposure. Plan TLS support.
3. **S-03:** Define the config safe-fields allowlist in code. Exclude ALL path and credential fields from web editing.
4. **S-06:** Remove `Access-Control-Allow-Origin: *` from existing code immediately.

### Before LAN Deployment (Phase 2-3)

5. **S-04:** Implement the enhanced path validation with component-level regex checks, extension allowlist, and comprehensive test suite.
6. **S-12:** Define Pydantic request models for every POST/PUT endpoint.
7. **S-11:** Replace broad exception catching with typed error handling and generic client messages.
8. **S-07:** Add rate limiting middleware (`slowapi` or custom).
9. **Add security headers middleware:**
   ```python
   @app.middleware("http")
   async def security_headers(request, call_next):
       response = await call_next(request)
       response.headers["X-Content-Type-Options"] = "nosniff"
       response.headers["X-Frame-Options"] = "DENY"
       response.headers["Content-Security-Policy"] = "default-src 'self'; script-src 'self' 'unsafe-inline'"
       response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
       return response
   ```

### Before Phase 7 (Auth & Feedback)

10. **S-02 (continued):** Implement self-signed TLS generation. Refuse `--password` without `--https`.
11. **S-13:** Rate-limit feedback endpoint. Consider local-first queue model.
12. **Add CSRF protection** for state-changing HTMX requests.
13. **Add `pip-audit`** to CI pipeline for dependency vulnerability scanning.

### Ongoing

14. **S-05:** Audit all subprocess calls in absorbed pdf-by-chapters code. Ensure list-form, no `shell=True`.
15. **S-09:** Add `busy_timeout` PRAGMA alongside WAL. Document local-filesystem requirement.
16. **S-08:** Automate service worker cache version from package version.
17. **S-10:** Add transport guard comment to MCP server code.

---

## Additional Recommendations Not in Original Concerns List

### CSRF Protection for HTMX

HTMX makes `PUT`/`POST`/`DELETE` requests from `hx-*` attributes. Without CSRF tokens, any page the user visits can trigger these. FastAPI does not include CSRF middleware by default.

**Recommendation:** Use a CSRF token pattern:
```python
# Generate token on page load, include in meta tag
# HTMX auto-includes headers from meta tags:
# <meta name="csrf-token" content="{{ csrf_token }}">
# hx-headers='{"X-CSRF-Token": document.querySelector("meta[name=csrf-token]").content}'
```

### Dependency Auditing

The plan adds significant new dependencies (`fastapi`, `uvicorn`, `jinja2`, `python-multipart`, `pymupdf`, `httpx`, `notebooklm-py`). None are audited.

**Recommendation:** Add `pip-audit` or `safety` to CI:
```yaml
# In pre-commit or CI:
- uv pip install pip-audit
- pip-audit --require-hashes --strict
```

### XSS in Jinja2 Templates

While Jinja2 auto-escapes HTML by default, the plan includes:
- HTMX fragment responses (partial HTML)
- `|safe` filter usage (common mistake)
- Flashcard content that may contain markdown/HTML from user-generated JSON files

**Recommendation:**
- Never use `|safe` on user content
- Sanitize flashcard front/back content on load (strip HTML tags or use bleach/nh3)
- Set `Content-Security-Policy` to block inline scripts

### File Upload (Screenshot in Feedback)

The plan mentions "optional screenshot upload" in the feedback form. File upload introduces:
- Storage exhaustion (large files)
- Malicious file content (polyglot files)
- Path traversal in filenames

**Recommendation:** If keeping screenshot upload:
- Limit file size (2MB max)
- Accept only `.png`, `.jpg` extensions AND verify magic bytes
- Generate a random filename (never use client-provided name)
- Store in a dedicated temp directory, not in the content tree

---

## Sources

- Plan file: `docs/plans/2026-03-15-feat-unified-study-platform-plan.md`
- Existing web server: `packages/studyctl/src/studyctl/web/server.py`
- Existing service worker: `packages/studyctl/src/studyctl/web/static/sw.js`
- Existing MCP server: `agents/mcp/study-speak-server.py`
- Existing config loader: `packages/agent-session-tools/src/agent_session_tools/config_loader.py`
- OWASP Top 10 (2021): https://owasp.org/Top10/
