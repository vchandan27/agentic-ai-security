# ⏱️ Rate Limiting & Scope Controls for AI Agents

> **Phase 5 · Article 9 of 12** | ⏱️ 16 min read | 🏷️ `#defense` `#rate-limiting` `#dos-prevention` `#resource-control`

---

## TL;DR

- Agentic AI systems can consume runaway resources — tokens, API credits, database queries, compute — in ways that ordinary apps cannot.
- Rate limiting isn't just about DoS prevention; it's also a **behavior bound** that catches hijacked agents before they cause serious damage.
- Implement limits at **every resource dimension**: tokens, tool calls, sessions, users, and tenants.

---

## Why Agents Need Special Rate Limiting

Traditional rate limiting asks: *"How many HTTP requests per minute?"*

Agents need multidimensional rate limiting:

```
AGENT RESOURCE DIMENSIONS:
─────────────────────────────────────────────────────────────────
TOKENS:    Input tokens per request, output tokens per request,
           total tokens per session, total tokens per day per user

TOOL CALLS: Calls per session, calls per minute, calls to a
            specific high-risk tool, total tool call budget

API BUDGET: Dollar cost per session, per user, per day (some
            providers charge per token — runaway = bill shock)

SESSIONS:  Concurrent sessions per user, sessions per minute,
           maximum session duration (auto-expire)

STORAGE:   RAG documents uploaded per user, memory entries
           per agent instance, vector DB writes per day

EXTERNAL:  HTTP requests from tools per minute, email sends
           per hour, outbound data volume per session
```

---

## The Circuit Breaker Pattern

Borrowed from distributed systems — stop cascading failures by breaking the circuit when things go wrong.

```python
import time
from enum import Enum
from threading import Lock
from dataclasses import dataclass

class CircuitState(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Blocking all requests (failure state)
    HALF_OPEN = "half_open" # Testing if recovery is possible

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5      # Failures before opening
    success_threshold: int = 2      # Successes before closing again
    timeout_seconds: int = 60       # How long to stay OPEN

class AgentCircuitBreaker:
    """
    Prevents runaway agent execution by breaking the circuit when
    failure rates exceed thresholds.

    Use case: If a tool is failing repeatedly (network down, API error),
    stop the agent from hammering it — break the circuit, alert, recover.
    """

    def __init__(self, name: str, config: CircuitBreakerConfig):
        self.name = name
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time: float = 0
        self._lock = Lock()

    def call(self, func, *args, **kwargs):
        with self._lock:
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_failure_time > self.config.timeout_seconds:
                    self.state = CircuitState.HALF_OPEN
                    self.success_count = 0
                else:
                    raise CircuitOpenError(
                        f"Circuit {self.name} is OPEN — refusing calls"
                    )

        try:
            result = func(*args, **kwargs)
            with self._lock:
                if self.state == CircuitState.HALF_OPEN:
                    self.success_count += 1
                    if self.success_count >= self.config.success_threshold:
                        self.state = CircuitState.CLOSED
                        self.failure_count = 0
            return result

        except Exception as e:
            with self._lock:
                self.failure_count += 1
                self.last_failure_time = time.time()
                if self.failure_count >= self.config.failure_threshold:
                    self.state = CircuitState.OPEN
                    security_alert(
                        f"Circuit {self.name} OPENED after {self.failure_count} failures"
                    )
            raise
```

---

## Token Budget Management

Token consumption is the most important cost control — and a critical security control.

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class TokenBudget:
    max_input_per_request: int = 8_000
    max_output_per_request: int = 4_000
    max_total_per_session: int = 50_000
    max_total_per_user_per_day: int = 500_000

class TokenBudgetManager:
    def __init__(self, budget: TokenBudget, storage: 'BudgetStorage'):
        self.budget = budget
        self.storage = storage

    def check_and_consume(
        self,
        user_id: str,
        session_id: str,
        input_tokens: int,
        estimated_output: int
    ) -> tuple[bool, str]:
        """
        Check if this request fits within all budgets.
        Consume tokens atomically if it does.
        Returns (allowed, reason).
        """
        # Per-request limits
        if input_tokens > self.budget.max_input_per_request:
            return False, f"Input too large: {input_tokens} > {self.budget.max_input_per_request}"

        if estimated_output > self.budget.max_output_per_request:
            return False, f"Expected output too large"

        # Session budget
        session_used = self.storage.get_session_tokens(session_id)
        if session_used + input_tokens + estimated_output > self.budget.max_total_per_session:
            return False, f"Session budget exhausted ({session_used}/{self.budget.max_total_per_session})"

        # Daily user budget
        daily_used = self.storage.get_daily_tokens(user_id)
        if daily_used + input_tokens > self.budget.max_total_per_user_per_day:
            return False, f"Daily budget exhausted for user {user_id}"

        # All checks passed — atomically consume
        self.storage.consume(
            session_id=session_id,
            user_id=user_id,
            tokens=input_tokens + estimated_output
        )
        return True, "OK"

    def get_remaining_budget(self, user_id: str, session_id: str) -> dict:
        return {
            "session_remaining": (
                self.budget.max_total_per_session
                - self.storage.get_session_tokens(session_id)
            ),
            "daily_remaining": (
                self.budget.max_total_per_user_per_day
                - self.storage.get_daily_tokens(user_id)
            )
        }
```

---

## Tool Call Rate Limiting

```python
import time
from collections import defaultdict, deque

class ToolCallRateLimiter:
    """
    Multi-dimensional rate limiter for agent tool calls.
    Uses sliding window counters.
    """

    def __init__(self):
        # Per-session: (session_id → list of timestamps)
        self._session_calls: dict[str, deque] = defaultdict(
            lambda: deque(maxlen=1000)
        )
        # Per-user-per-tool: (user_id, tool_name) → deque
        self._user_tool_calls: dict[tuple, deque] = defaultdict(deque)

    # Limits (configurable per deployment)
    LIMITS = {
        "default": {
            "session_per_minute": 30,
            "session_total": 100,
        },
        "high_risk_tools": {  # execute_code, send_email, delete_*
            "user_per_hour": 10,
            "user_per_day": 50,
        },
        "web_search": {
            "session_per_minute": 5,
            "user_per_hour": 100,
        },
        "execute_code": {
            "user_per_hour": 5,    # Very conservative
            "user_per_day": 20,
        },
        "send_email": {
            "user_per_hour": 3,    # Anti-spam
            "user_per_day": 10,
        }
    }

    def is_allowed(
        self,
        session_id: str,
        user_id: str,
        tool_name: str
    ) -> tuple[bool, str]:
        now = time.time()

        # Get tool-specific limits (fall back to default)
        tool_limits = self.LIMITS.get(tool_name, self.LIMITS["default"])

        # Check session-per-minute limit
        session_window = self._session_calls[session_id]
        # Remove entries older than 1 minute
        while session_window and now - session_window[0] > 60:
            session_window.popleft()

        per_min_limit = tool_limits.get("session_per_minute", 30)
        if len(session_window) >= per_min_limit:
            return False, f"Rate limit: max {per_min_limit} {tool_name} calls/minute"

        # Check user-per-hour limit for high-risk tools
        if "user_per_hour" in tool_limits:
            user_key = (user_id, tool_name)
            user_window = self._user_tool_calls[user_key]
            while user_window and now - user_window[0] > 3600:
                user_window.popleft()
            if len(user_window) >= tool_limits["user_per_hour"]:
                return False, f"Hourly limit reached for {tool_name}"

        # Record this call
        session_window.append(now)
        if "user_per_hour" in tool_limits:
            self._user_tool_calls[(user_id, tool_name)].append(now)

        return True, "OK"
```

---

## Maximum Recursion Depth & Step Limits

Agents can recurse indefinitely without explicit limits:

```python
class AgentExecutionLimiter:
    """
    Hard limits on agent execution to prevent infinite loops
    and runaway multi-agent recursion.
    """

    MAX_STEPS = 30           # Maximum tool calls in a single task
    MAX_DEPTH = 5            # Maximum agent delegation depth
    MAX_DURATION_SECS = 300  # Maximum task duration (5 minutes)

    def __init__(self):
        self._step_counts: dict[str, int] = {}
        self._depths: dict[str, int] = {}
        self._start_times: dict[str, float] = {}

    def start_task(self, task_id: str, parent_task_id: Optional[str] = None):
        self._step_counts[task_id] = 0
        self._start_times[task_id] = time.time()

        # Calculate depth from parent chain
        if parent_task_id:
            parent_depth = self._depths.get(parent_task_id, 0)
            if parent_depth >= self.MAX_DEPTH:
                raise RecursionLimitError(
                    f"Max agent delegation depth ({self.MAX_DEPTH}) exceeded. "
                    f"Possible runaway recursion or injection attack."
                )
            self._depths[task_id] = parent_depth + 1
        else:
            self._depths[task_id] = 0

    def record_step(self, task_id: str):
        """Call before each tool execution."""
        steps = self._step_counts.get(task_id, 0) + 1
        self._step_counts[task_id] = steps

        # Check step count
        if steps > self.MAX_STEPS:
            raise StepLimitError(
                f"Task {task_id} exceeded {self.MAX_STEPS} steps. "
                f"Terminating to prevent infinite loop."
            )

        # Check duration
        elapsed = time.time() - self._start_times.get(task_id, time.time())
        if elapsed > self.MAX_DURATION_SECS:
            raise DurationLimitError(
                f"Task {task_id} exceeded {self.MAX_DURATION_SECS}s. Terminating."
            )
```

---

## Scope Controls: Bounding Agent Actions

Beyond rate limits, use **scope controls** to bound *what* an agent can do, not just *how often*:

```
SCOPE CONTROL PATTERNS:
─────────────────────────────────────────────────────────────────

1. TEMPORAL SCOPE:
   Agent is only authorized to act on data created in the
   last 30 days. Historical data is read-only via a different,
   more restrictive API.

2. DATA SCOPE:
   Agent can only query records belonging to the requesting user.
   Cross-user queries always require explicit admin authorization.

3. FINANCIAL SCOPE:
   Agent can approve transactions up to $100.
   $100-$1,000: requires supervisor confirmation.
   >$1,000: always HITL with CFO or delegate.

4. EXTERNAL SCOPE:
   Agent can send emails to:
   - @company.com (unrestricted)
   - Customer emails in active tickets (restricted)
   - Arbitrary external (never, without manual override)

5. KNOWLEDGE SCOPE:
   Agent's RAG search is scoped to its assigned workspace.
   It cannot query knowledge bases belonging to other teams.
```

---

## Rate Limiting Checklist

```
TOKEN CONTROLS:
[ ] Maximum input tokens per request configured
[ ] Maximum output tokens per request configured
[ ] Per-session token budget enforced
[ ] Per-user daily token budget enforced
[ ] Token usage tracked and visible to users (self-service)

TOOL CALL CONTROLS:
[ ] Maximum tool calls per session configured (e.g., 30)
[ ] Per-minute call rate limited per session
[ ] High-risk tool calls separately rate-limited
[ ] Tool call depth (recursion) limited
[ ] Maximum task duration enforced (auto-timeout)

CIRCUIT BREAKERS:
[ ] Circuit breaker on every external API dependency
[ ] Circuit opens after N consecutive failures
[ ] Open circuit alerts on-call
[ ] Half-open state allows gradual recovery

SCOPE CONTROLS:
[ ] Financial transaction caps enforced at system level
[ ] Data scope enforced per user (cross-user queries blocked)
[ ] External communication scoped to allowlisted recipients
[ ] Knowledge base access scoped to team/department

COST CONTROLS:
[ ] Real-time cost tracking per session
[ ] Alert when session cost exceeds threshold
[ ] Hard cap on monthly spend per user/tenant
[ ] Cost anomaly detection (sudden 10x increase)
```

---

*← [Prev: Security Monitoring](./08-security-monitoring.md) | [Next: Red-Teaming Agents →](./10-red-teaming-agents.md)*
