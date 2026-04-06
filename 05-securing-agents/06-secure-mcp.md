# 🔐 Secure MCP Server Design

> **Phase 5 · Article 6 of 12** | ⏱️ 19 min read | 🏷️ `#mcp` `#integration` `#server-security` `#api-hardening`

---

## TL;DR

- **MCP servers are attack surface**: Every tool the agent calls is a potential vector for compromising the agent, exfiltrating data, or injecting malicious content into the agent's context.
- **Server-side validation is critical**: Validate inputs, enforce rate limits, log all calls, and scan outputs before returning to the agent.
- **Defense layers**: Input validation → Business logic checks → Resource limits → Output scanning → Audit logging.
- **Common mistakes**: Trusting agent input implicitly, not validating before executing tool logic, returning raw system data without sanitization.

---

## MCP Security Architecture Review

```
AGENT ←→ MCP GATEWAY ←→ MCP SERVERS ←→ PROTECTED RESOURCES
         (Proxy)         (Tool Logic)    (Databases, APIs, Files)

SECURITY RESPONSIBILITIES:

Agent's responsibility:
  - Decide WHAT to call and WHY
  - Validate that tool exists and agent has permission
  - Pass parameters that comply with tool schema

Gateway responsibility (Optional but RECOMMENDED):
  - Enforce rate limits
  - Validate authentication
  - Route to appropriate server
  - Log all calls
  - Scan responses before returning

MCP Server responsibility:
  - Validate input parameters (even if agent supposedly did)
  - Check business logic constraints
  - Enforce resource limits
  - Execute tool logic safely
  - Sanitize output
  - Log execution details

Resource responsibility:
  - Implement access control (database permissions, etc.)
  - Prevent unauthorized access
  - Log access attempts
```

### Architecture Diagram

```
┌──────────────┐
│              │
│    AGENT     │
│  (Claude)    │
│              │
└──────┬───────┘
       │ "Get user data"
       │ (with validation)
       ▼
┌──────────────────────────────────────┐
│   MCP GATEWAY (SECURITY PROXY)       │
├──────────────────────────────────────┤
│ • Rate limit check                   │
│ • Auth token validation              │
│ • Tool permission check              │
│ • Parameter schema check             │
│ • Malicious output detection         │
└──────┬───────────────────────────────┘
       │ Sanitized request
       ▼
┌──────────────────────────────────────┐
│  MCP SERVER (Tool Implementation)    │
├──────────────────────────────────────┤
│ • Re-validate input (defense-in-depth)
│ • Enforce business rules             │
│ • Query database safely              │
│ • Check output for secrets           │
│ • Log execution                      │
└──────┬───────────────────────────────┘
       │ {"data": [...sanitized...]}
       ▼
┌──────────────────────────────────────┐
│  PROTECTED RESOURCE                  │
├──────────────────────────────────────┤
│ • Database (with row-level security) │
│ • API (with rate limits)             │
│ • File system (with ACLs)            │
└──────────────────────────────────────┘
```

---

## Server-Side Input Validation

**Rule: NEVER trust the agent's input. Re-validate everything server-side.**

```python
# ❌ WRONG: Trusting agent's input implicitly
class UnsecureFileServer:
    def get_file(self, filepath: str) -> str:
        """
        DANGEROUS: Agent could request /etc/passwd
        Agent might try: "../../../etc/passwd" (path traversal)
        """
        try:
            with open(filepath, 'r') as f:
                return f.read()
        except:
            return "Error reading file"

# ✅ RIGHT: Validating all inputs server-side
from pathlib import Path
import re

class SecureFileServer:
    def __init__(self, base_dir: str):
        self.base_dir = Path(base_dir).resolve()

    def get_file(self, filepath: str) -> dict:
        """
        Secure file access with multiple validation layers.
        """
        try:
            # LAYER 1: Input validation
            if not filepath:
                return {"error": "filepath cannot be empty", "success": False}

            if not isinstance(filepath, str):
                return {"error": "filepath must be string", "success": False}

            # LAYER 2: Path traversal prevention
            requested_path = (self.base_dir / filepath).resolve()

            # Ensure requested path is within base_dir
            if not str(requested_path).startswith(str(self.base_dir)):
                return {
                    "error": "Path traversal detected",
                    "success": False
                }

            # LAYER 3: File existence and type check
            if not requested_path.exists():
                return {"error": "File not found", "success": False}

            if not requested_path.is_file():
                return {"error": "Path is not a file", "success": False}

            # LAYER 4: Size limit (prevent DoS)
            max_size = 10_000_000  # 10MB
            if requested_path.stat().st_size > max_size:
                return {
                    "error": f"File exceeds size limit ({max_size} bytes)",
                    "success": False
                }

            # LAYER 5: Whitelist file types
            allowed_extensions = {".txt", ".md", ".pdf", ".json", ".csv"}
            if requested_path.suffix not in allowed_extensions:
                return {
                    "error": f"File type not allowed: {requested_path.suffix}",
                    "success": False
                }

            # LAYER 6: Read and sanitize
            with open(requested_path, 'r', encoding='utf-8', errors='ignore') as f:
                content = f.read()

            # LAYER 7: Output sanitization (see section below)
            sanitized_content = self._sanitize_content(content)

            return {
                "success": True,
                "filepath": str(requested_path),
                "size_bytes": len(sanitized_content),
                "content": sanitized_content
            }

        except Exception as e:
            # Log but don't expose details
            self._log_error(f"get_file error: {type(e).__name__}")
            return {"error": "Internal error", "success": False}

    def _sanitize_content(self, content: str) -> str:
        """Remove potentially dangerous patterns before returning to agent"""
        # Remove private keys (SSH, API keys, etc.)
        content = re.sub(
            r"-----BEGIN (PRIVATE|RSA|DSA) KEY-----[\s\S]*?-----END (PRIVATE|RSA|DSA) KEY-----",
            "[PRIVATE KEY REMOVED]",
            content
        )

        # Remove API keys
        content = re.sub(
            r"(api[_-]?key|secret|password|token)\s*[=:]\s*['\"]?[A-Za-z0-9_\-\.]{20,}['\"]?",
            r"\1=[REDACTED]",
            content,
            flags=re.IGNORECASE
        )

        # Remove email addresses (PII)
        content = re.sub(
            r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
            "[EMAIL REDACTED]",
            content
        )

        return content

    def _log_error(self, message: str):
        """Log errors securely (not exposing paths or system info)"""
        print(f"[ERROR] {message}")  # Use proper logging in production
```

---

## Output Content Scanning Before Returning

**Before returning tool output to the agent, scan for sensitive data.**

```python
import json
from dataclasses import dataclass
from enum import Enum

class ContentSeverity(Enum):
    LOW = "low"        # Non-sensitive info
    MEDIUM = "medium"  # Internal details that shouldn't be shared
    HIGH = "high"      # Secrets, credentials, PII
    CRITICAL = "critical"  # Database credentials, auth tokens

@dataclass
class ContentIssue:
    severity: ContentSeverity
    pattern: str
    matched_value: str  # For debugging, not returned to agent

class OutputScanner:
    """
    Scans tool output for sensitive data before returning to agent.
    """

    SENSITIVE_PATTERNS = [
        # API Keys (AWS, OpenAI, etc.)
        (ContentSeverity.CRITICAL, "api_key",
         r"(?:^|\s)(?:api[_-]?key|sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16})(?:\s|$|['\"]|=)"),

        # Database connection strings
        (ContentSeverity.CRITICAL, "db_connection",
         r"(?:mysql|postgres|mongodb)://[^\s'\"]+"),

        # AWS credentials
        (ContentSeverity.CRITICAL, "aws_credentials",
         r"(?:AKIA|ASIA)[0-9A-Z]{16}|aws_access_key_id"),

        # Private keys
        (ContentSeverity.CRITICAL, "private_key",
         r"-----BEGIN (?:RSA |DSA |EC )?PRIVATE KEY-----"),

        # JWT tokens (long base64)
        (ContentSeverity.CRITICAL, "jwt_token",
         r"eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+"),

        # SQL passwords (in connection strings, .sql files)
        (ContentSeverity.HIGH, "password_literal",
         r"password\s*[=:]\s*['\"]?[^'\"\s;]+['\"]?"),

        # Email addresses (PII)
        (ContentSeverity.MEDIUM, "email",
         r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}"),

        # Phone numbers (PII)
        (ContentSeverity.MEDIUM, "phone",
         r"(?:\+1|\b1)?[-.\s]?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}\b"),

        # IPv4 addresses (internal infrastructure info)
        (ContentSeverity.MEDIUM, "ipv4",
         r"\b(?:10\.|172\.16\.|192\.168\.)[0-9]{1,3}\.[0-9]{1,3}\b"),

        # Server paths (information disclosure)
        (ContentSeverity.MEDIUM, "server_path",
         r"/(?:home|root|var|etc|opt)/[^\s'\"]+"),
    ]

    @staticmethod
    def scan(content: str, max_severity: ContentSeverity = ContentSeverity.CRITICAL) -> dict:
        """
        Scan content for sensitive patterns.
        Returns: {issues: [...], sanitized_content: "...", safe_to_return: bool}
        """
        issues = []
        scan_text = content

        for severity, pattern_name, pattern in OutputScanner.SENSITIVE_PATTERNS:
            # Skip patterns below max_severity threshold
            if severity.value > max_severity.value:
                continue

            for match in re.finditer(pattern, scan_text, re.MULTILINE | re.IGNORECASE):
                issues.append(ContentIssue(
                    severity=severity,
                    pattern=pattern_name,
                    matched_value=match.group(0)[:50]  # First 50 chars only
                ))

        # Determine if safe to return
        has_critical = any(i.severity == ContentSeverity.CRITICAL for i in issues)
        safe_to_return = not has_critical

        # Sanitize content if issues found
        sanitized = OutputScanner._sanitize(content, issues)

        return {
            "safe_to_return": safe_to_return,
            "issues_found": len(issues),
            "critical_issues": sum(1 for i in issues if i.severity == ContentSeverity.CRITICAL),
            "issues": [
                {
                    "severity": i.severity.value,
                    "pattern": i.pattern,
                } for i in issues
            ],
            "sanitized_content": sanitized
        }

    @staticmethod
    def _sanitize(content: str, issues: list[ContentIssue]) -> str:
        """Replace sensitive patterns with placeholders"""
        sanitized = content

        # Redact critical items first
        for issue in issues:
            if issue.severity == ContentSeverity.CRITICAL:
                if "key" in issue.pattern:
                    sanitized = re.sub(
                        r"(?:api[_-]?key|secret|token)\s*[=:]\s*\S+",
                        r"\g<0>[REDACTED]",
                        sanitized,
                        flags=re.IGNORECASE
                    )
                elif "password" in issue.pattern:
                    sanitized = re.sub(
                        r"password\s*[=:]\s*\S+",
                        r"\g<0>[REDACTED]",
                        sanitized,
                        flags=re.IGNORECASE
                    )

        return sanitized
```

---

## Authentication for MCP Servers

### OAuth for Service Authentication

```python
from typing import Optional
import jwt
import time

class MCPServerAuth:
    """
    Authentication layer for MCP servers.
    Supports multiple auth types: OAuth2, API key, mTLS.
    """

    def __init__(self, secret_key: str, issuer: str = "mcp-gateway"):
        self.secret_key = secret_key
        self.issuer = issuer
        self.agent_registry = {}  # Map agent_id → scopes

    # METHOD 1: OAuth2 with JWT
    def generate_agent_token(self, agent_id: str, scopes: list[str], ttl: int = 3600) -> str:
        """
        Generate short-lived JWT token for agent.
        Token includes agent identity and authorized scopes.
        """
        now = int(time.time())
        payload = {
            "iss": self.issuer,
            "sub": agent_id,  # Subject = agent ID
            "scopes": scopes,
            "iat": now,
            "exp": now + ttl,  # Expires in 1 hour
        }

        token = jwt.encode(payload, self.secret_key, algorithm="HS256")
        return token

    def verify_agent_token(self, token: str) -> dict:
        """
        Verify token and extract agent identity + scopes.
        """
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=["HS256"])

            # Check expiration
            if payload["exp"] < int(time.time()):
                return {"valid": False, "error": "Token expired"}

            return {
                "valid": True,
                "agent_id": payload["sub"],
                "scopes": payload["scopes"]
            }

        except jwt.InvalidTokenError as e:
            return {"valid": False, "error": str(e)}

    # METHOD 2: API Key with scoped permissions
    def issue_api_key(self, agent_id: str, scopes: list[str]) -> str:
        """
        Issue API key for agent (alternative to OAuth).
        Key format: agent_<id>_<random_hash>
        """
        import secrets
        key = f"agent_{agent_id}_{secrets.token_hex(16)}"
        self.agent_registry[key] = {
            "agent_id": agent_id,
            "scopes": scopes,
            "created_at": int(time.time())
        }
        return key

    def validate_api_key(self, api_key: str) -> dict:
        """Validate API key and extract permissions"""
        if api_key not in self.agent_registry:
            return {"valid": False, "error": "Invalid API key"}

        entry = self.agent_registry[api_key]
        return {
            "valid": True,
            "agent_id": entry["agent_id"],
            "scopes": entry["scopes"]
        }

    # METHOD 3: mTLS (mutual TLS)
    def validate_mtls_cert(self, cert_subject: str) -> dict:
        """
        Validate that request is from trusted certificate.
        cert_subject format: "CN=agent-123"
        """
        # Extract agent ID from cert subject
        import re
        match = re.search(r"CN=agent-(\d+)", cert_subject)
        if not match:
            return {"valid": False, "error": "Invalid certificate subject"}

        agent_id = match.group(1)

        # Check if cert is in trusted list (loaded from KMS)
        if agent_id in self.agent_registry:
            return {
                "valid": True,
                "agent_id": agent_id,
                "scopes": self.agent_registry[agent_id]["scopes"]
            }

        return {"valid": False, "error": "Agent not trusted"}
```

---

## Rate Limiting on MCP Endpoints

```python
from collections import defaultdict
import time

class RateLimiter:
    """
    Token bucket rate limiter for MCP endpoints.
    """

    def __init__(self, tokens_per_minute: int = 60):
        self.capacity = tokens_per_minute
        self.refill_rate = tokens_per_minute / 60  # Per second
        self.buckets = defaultdict(lambda: {"tokens": self.capacity, "last_refill": time.time()})

    def is_allowed(self, agent_id: str, cost: int = 1) -> bool:
        """
        Check if agent can make request.
        Some operations cost more tokens (e.g., bulk export = 10 tokens).
        """
        bucket = self.buckets[agent_id]

        # Refill bucket based on time elapsed
        now = time.time()
        time_elapsed = now - bucket["last_refill"]
        bucket["tokens"] = min(
            self.capacity,
            bucket["tokens"] + time_elapsed * self.refill_rate
        )
        bucket["last_refill"] = now

        # Check if agent has enough tokens
        if bucket["tokens"] >= cost:
            bucket["tokens"] -= cost
            return True

        return False

    def get_remaining(self, agent_id: str) -> int:
        """Get remaining tokens for agent"""
        bucket = self.buckets[agent_id]
        now = time.time()
        time_elapsed = now - bucket["last_refill"]
        remaining = min(
            self.capacity,
            bucket["tokens"] + time_elapsed * self.refill_rate
        )
        return int(remaining)

# Usage:
limiter = RateLimiter(tokens_per_minute=100)

class RateLimitedMCPServer:
    def __init__(self):
        self.limiter = RateLimiter(tokens_per_minute=100)

    def query_database(self, agent_id: str, query: str) -> dict:
        """
        MCP tool: query database with rate limiting.
        """
        # Standard query costs 1 token
        cost = 1

        # Large queries cost more
        if len(query) > 1000:
            cost = 5
        if "SELECT *" in query:
            cost = 10  # Discourage table scans

        if not self.limiter.is_allowed(agent_id, cost):
            return {
                "error": "Rate limit exceeded",
                "remaining_tokens": self.limiter.get_remaining(agent_id),
                "reset_at": "in ~60 seconds"
            }

        # Execute query (with other validations...)
        return {"success": True, "data": [...]}
```

---

## Audit Logging Per Tool Call

```python
import json
import logging
from datetime import datetime
from dataclasses import dataclass, asdict

@dataclass
class AuditLogEntry:
    timestamp: str
    agent_id: str
    tool_name: str
    request_params: dict
    response_status: str  # "success", "error", "rate_limited"
    response_size_bytes: int
    execution_time_ms: float
    http_status: int
    issues_detected: list  # From output scanner

class AuditLogger:
    """
    Logs all MCP tool calls for security audit trail.
    """

    def __init__(self, log_file: str = "/var/log/mcp_audit.jsonl"):
        self.log_file = log_file
        self.logger = logging.getLogger("mcp_audit")
        handler = logging.FileHandler(log_file)
        formatter = logging.Formatter('%(message)s')
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)

    def log_tool_call(self, entry: AuditLogEntry):
        """Log a tool call to audit trail"""
        log_dict = asdict(entry)
        self.logger.info(json.dumps(log_dict))

# Usage in MCP server:
class AuditedMCPServer:
    def __init__(self):
        self.audit_logger = AuditLogger()

    def some_tool(self, agent_id: str, param1: str) -> dict:
        """
        Example tool with audit logging.
        """
        import time

        start_time = time.time()

        try:
            # VALIDATION
            if not param1:
                self.audit_logger.log_tool_call(AuditLogEntry(
                    timestamp=datetime.utcnow().isoformat(),
                    agent_id=agent_id,
                    tool_name="some_tool",
                    request_params={"param1": "[empty]"},
                    response_status="error",
                    response_size_bytes=0,
                    execution_time_ms=(time.time() - start_time) * 1000,
                    http_status=400,
                    issues_detected=["Invalid input"]
                ))
                return {"error": "param1 required"}

            # EXECUTE
            result = self._execute_tool_logic(param1)

            # SCAN OUTPUT
            scan_result = OutputScanner.scan(json.dumps(result))

            # LOG
            self.audit_logger.log_tool_call(AuditLogEntry(
                timestamp=datetime.utcnow().isoformat(),
                agent_id=agent_id,
                tool_name="some_tool",
                request_params={"param1": "[redacted]"},
                response_status="success",
                response_size_bytes=len(json.dumps(result)),
                execution_time_ms=(time.time() - start_time) * 1000,
                http_status=200,
                issues_detected=scan_result["issues"]
            ))

            return result

        except Exception as e:
            self.audit_logger.log_tool_call(AuditLogEntry(
                timestamp=datetime.utcnow().isoformat(),
                agent_id=agent_id,
                tool_name="some_tool",
                request_params={"param1": "[error during logging]"},
                response_status="error",
                response_size_bytes=0,
                execution_time_ms=(time.time() - start_time) * 1000,
                http_status=500,
                issues_detected=[str(type(e).__name__)]
            ))
            return {"error": "Internal error"}
```

---

## Preventing Path Traversal in File Tools

```python
# ✅ SECURE FILE TOOL

from pathlib import Path
import os

class SecureFileAccessServer:
    """
    File access tool that prevents path traversal, symlink attacks, etc.
    """

    def __init__(self, base_dir: str):
        self.base_dir = Path(base_dir).resolve()

    def read_file(self, filepath: str) -> dict:
        """
        Read file safely with multiple layers of validation.
        """

        # LAYER 1: Input validation
        if not filepath or not isinstance(filepath, str):
            return {"error": "Invalid filepath", "success": False}

        # LAYER 2: Path normalization and traversal check
        try:
            requested = (self.base_dir / filepath).resolve()

            # Ensure within base directory
            if not str(requested).startswith(str(self.base_dir)):
                return {"error": "Access denied: path outside base directory", "success": False}

        except (ValueError, OSError):
            return {"error": "Invalid path", "success": False}

        # LAYER 3: Symlink check
        try:
            # Ensure no symlinks in path (prevents symlink attacks)
            if requested.is_symlink():
                return {"error": "Symlinks not allowed", "success": False}

            # Check if any parent directories contain symlinks
            for parent in requested.parents:
                if parent.is_symlink():
                    return {"error": "Path contains symlinks", "success": False}

        except OSError:
            return {"error": "Cannot access file", "success": False}

        # LAYER 4: File type and size checks
        if not requested.is_file():
            return {"error": "Not a regular file", "success": False}

        # LAYER 5: Size limit
        try:
            size = requested.stat().st_size
            max_size = 50_000_000  # 50MB
            if size > max_size:
                return {"error": f"File too large: {size} > {max_size}", "success": False}

        except OSError:
            return {"error": "Cannot stat file", "success": False}

        # LAYER 6: Read and scan
        try:
            with open(requested, 'r', encoding='utf-8', errors='ignore') as f:
                content = f.read()

            scan_result = OutputScanner.scan(content)
            if not scan_result["safe_to_return"]:
                content = scan_result["sanitized_content"]

            return {
                "success": True,
                "filepath": str(requested),
                "size_bytes": len(content),
                "content": content,
                "issues_scanned": scan_result["critical_issues"]
            }

        except Exception as e:
            return {"error": f"Cannot read file: {type(e).__name__}", "success": False}
```

---

## Preventing SSRF in URL-Fetch Tools

```python
import re
from urllib.parse import urlparse

class SSRFProtection:
    """
    Prevent Server-Side Request Forgery (SSRF) attacks
    in URL-fetch MCP tools.
    """

    # Blocked IP ranges (internal networks)
    BLOCKED_CIDRS = [
        "0.0.0.0/8",           # This network
        "10.0.0.0/8",          # Private
        "127.0.0.0/8",         # Loopback
        "169.254.0.0/16",      # Link-local
        "172.16.0.0/12",       # Private
        "192.0.0.0/24",        # Documentation
        "192.0.2.0/24",        # TEST-NET-1
        "192.88.99.0/24",      # Anycast
        "192.168.0.0/16",      # Private
        "198.18.0.0/15",       # Benchmarking
        "198.51.100.0/24",     # TEST-NET-2
        "203.0.113.0/24",      # TEST-NET-3
        "224.0.0.0/4",         # Multicast
        "240.0.0.0/4",         # Reserved
        "255.255.255.255/32",  # Broadcast
    ]

    BLOCKED_HOSTNAMES = {
        "localhost",
        "127.0.0.1",
        "0.0.0.0",
        "metadata.google.internal",  # GCP metadata
        "169.254.169.254",           # AWS metadata
    }

    @staticmethod
    def is_blocked(url: str) -> bool:
        """
        Check if URL should be blocked to prevent SSRF.
        """
        try:
            parsed = urlparse(url)

            # Check hostname
            if parsed.hostname in SSRFProtection.BLOCKED_HOSTNAMES:
                return True

            # Check for IPv6 localhost
            if parsed.hostname in ["::1", "::", "[::1]"]:
                return True

            # Check for internal IP addresses
            try:
                import ipaddress
                ip = ipaddress.ip_address(parsed.hostname)

                for cidr in SSRFProtection.BLOCKED_CIDRS:
                    if ip in ipaddress.ip_network(cidr):
                        return True

            except ValueError:
                # Not an IP address (might be hostname), skip IP checks
                pass

            # Check port (some ports might be sensitive)
            sensitive_ports = {22, 3306, 5432, 6379, 27017, 9200}
            if parsed.port in sensitive_ports:
                return True

            return False

        except Exception:
            # If parsing fails, block to be safe
            return True

class SafeURLFetchServer:
    """
    Fetch URL with SSRF protection.
    """

    def fetch_url(self, url: str, agent_id: str) -> dict:
        """
        Safely fetch URL with SSRF protection.
        """

        # LAYER 1: Block SSRF URLs
        if SSRFProtection.is_blocked(url):
            return {
                "error": "URL is blocked (internal network access not allowed)",
                "success": False
            }

        # LAYER 2: Validate URL format
        try:
            parsed = urlparse(url)
            if not parsed.scheme or parsed.scheme not in ["http", "https"]:
                return {"error": "Only HTTP/HTTPS allowed", "success": False}

        except Exception:
            return {"error": "Invalid URL", "success": False}

        # LAYER 3: Enforce timeout (prevent slowloris attacks)
        try:
            import requests
            response = requests.get(
                url,
                timeout=5,  # 5 second timeout
                allow_redirects=False,  # Prevent redirect-based SSRF
                headers={"User-Agent": "AgentSecurityBot/1.0"}
            )

            # LAYER 4: Check response size (prevent OOM attacks)
            max_size = 10_000_000  # 10MB
            if int(response.headers.get("Content-Length", 0)) > max_size:
                return {"error": "Response too large", "success": False}

            # LAYER 5: Limit content-type
            allowed_types = {"text/", "application/json", "application/pdf"}
            content_type = response.headers.get("Content-Type", "").lower()
            if not any(ct in content_type for ct in allowed_types):
                return {"error": "Content-type not allowed", "success": False}

            # LAYER 6: Scan response
            content = response.text[:max_size]
            scan_result = OutputScanner.scan(content)
            if not scan_result["safe_to_return"]:
                content = scan_result["sanitized_content"]

            return {
                "success": True,
                "url": url,
                "status_code": response.status_code,
                "content": content,
                "size_bytes": len(content),
                "issues_found": scan_result["critical_issues"]
            }

        except requests.Timeout:
            return {"error": "Request timeout", "success": False}
        except Exception as e:
            return {"error": f"Fetch error: {type(e).__name__}", "success": False}
```

---

## MCP Security Checklist

```
SECURE MCP SERVER CHECKLIST
────────────────────────────────────────────────────────────────

INPUT VALIDATION:
  ☐ All parameters validated server-side (not trusting agent)
  ☐ Type checking (strings are strings, ints are ints)
  ☐ Length limits enforced (no unbounded inputs)
  ☐ Path traversal prevention (resolve paths, check boundaries)
  ☐ SQL injection prevention (parameterized queries)
  ☐ Command injection prevention (no shell=True, use subprocess arrays)

OUTPUT SCANNING:
  ☐ All responses scanned for credentials before returning
  ☐ API keys/secrets detected and redacted
  ☐ PII (emails, phones) detected and redacted
  ☐ Database connection strings detected and redacted
  ☐ Private keys detected and removed
  ☐ Internal IPs and paths sanitized

AUTHENTICATION:
  ☐ MCP servers authenticate agents (OAuth2, mTLS, or API key)
  ☐ Short-lived tokens (TTL < 1 hour)
  ☐ Tokens include scopes/permissions
  ☐ Token signature verified on each call

RATE LIMITING:
  ☐ Rate limits enforced per agent (token bucket or similar)
  ☐ Large operations cost more tokens (bulk exports, table scans)
  ☐ Rate limit exceeded returns clear error + reset time
  ☐ Rate limit state persistent (survives server restart)

AUDIT LOGGING:
  ☐ Every tool call logged (tool name, agent ID, timestamp)
  ☐ Request parameters logged (redacted if sensitive)
  ☐ Response status logged (success/error)
  ☐ Execution time logged
  ☐ Issues detected logged
  ☐ Logs immutable and centralized (SIEM, S3, etc.)

OPERATIONAL:
  ☐ Error messages don't leak system details
  ☐ Timeouts configured (prevent hanging requests)
  ☐ Maximum output size enforced (prevent memory DoS)
  ☐ Vulnerable dependencies updated (security patches applied)
  ☐ MCP gateway exists (proxy between agent and servers)
  ☐ HTTPS/TLS enforced for all MCP communication
```

---

## Further Reading

- **MCP Protocol Specification**: https://modelcontextprotocol.io/
- **OWASP: API Security Top 10**: https://owasp.org/www-project-api-security/
- **SSRF Prevention**: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- **Path Traversal Prevention**: https://owasp.org/www-community/attacks/Path_Traversal
- **JWT Best Practices**: https://tools.ietf.org/html/rfc8725
- **Rate Limiting Strategies**: https://en.wikipedia.org/wiki/Token_bucket

---

*← [Prev: Prompt Hardening Against Injection](./05-prompt-hardening.md) | [Next: Agent Identity & Attestation →](./07-agent-identity.md)*
