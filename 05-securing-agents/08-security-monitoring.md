# 👁️ Security Monitoring for AI Agents

> **Phase 5 · Article 8 of 12** | ⏱️ 22 min read | 🏷️ `#defense` `#monitoring` `#observability` `#detection`

---

## TL;DR

- You can't defend what you can't see. AI agents are *more* opaque than traditional software — their reasoning happens inside an LLM black box.
- Effective agent security monitoring requires **four layers**: structured logging, distributed tracing, anomaly detection, and real-time alerting.
- Many attacks (prompt injection, data exfiltration, agent hijacking) are *undetectable* without purpose-built agent monitoring.

---

## Why Traditional Monitoring Falls Short

```
TRADITIONAL APP MONITORING:              AGENT-SPECIFIC MONITORING NEEDED:
────────────────────────────────         ──────────────────────────────────────
HTTP request/response logging            + Full context window logging
Exception traces                         + LLM reasoning traces
Database query logs                      + Tool call sequences and parameters
CPU/memory/latency metrics               + Token consumption per step
Error rates                              + Intent drift detection
                                         + Injection pattern detection
                                         + Cross-session behavioral profiling
                                         + Output content analysis
```

A traditional APM (Application Performance Monitoring) tool will tell you the agent is *running*. It won't tell you the agent has been hijacked.

---

## The Four Monitoring Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                  AGENT MONITORING STACK                             │
│                                                                     │
│  LAYER 4: ALERTING & RESPONSE                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  PagerDuty / OpsGenie / Slack  │  SOAR playbooks            │  │
│  │  Alert rules   │  Escalation   │  Auto-remediation           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              ▲                                      │
│  LAYER 3: ANOMALY DETECTION                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Statistical baselines  │  ML-based anomaly detection        │  │
│  │  Rule-based detection   │  Behavioral profiling              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              ▲                                      │
│  LAYER 2: DISTRIBUTED TRACING                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  OpenTelemetry traces  │  LangSmith / LangFuse traces        │  │
│  │  Span attributes       │  Tool call spans                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              ▲                                      │
│  LAYER 1: STRUCTURED LOGGING                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Structured JSON logs  │  Audit trail                        │  │
│  │  Session context       │  Tool parameters & results          │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Structured Agent Logging

Every agent event must be logged in structured format — queryable, indexable, parseable by SIEM tools.

```python
import json
import time
import logging
from dataclasses import dataclass, asdict
from typing import Optional, Any

@dataclass
class AgentEvent:
    """Structured event schema for all agent actions."""
    # Identity
    event_type: str              # "llm_call", "tool_call", "session_start", etc.
    agent_id: str
    agent_version: str
    instance_id: str             # Unique per agent run
    session_id: str
    user_id: str
    tenant_id: str

    # Timing
    timestamp: str               # ISO-8601
    duration_ms: Optional[int]

    # LLM-specific
    model: Optional[str]
    input_tokens: Optional[int]
    output_tokens: Optional[int]
    prompt_hash: Optional[str]   # SHA256 of full prompt (not stored in plain text)

    # Tool-specific
    tool_name: Optional[str]
    tool_args: Optional[dict]    # Sanitized — strip secrets
    tool_result_summary: Optional[str]  # Not full result (may be large)
    tool_success: Optional[bool]

    # Security
    source_trust_level: str      # "user", "system", "external", "retrieved"
    injection_scan_result: Optional[str]  # "clean", "suspicious", "blocked"

    # Context
    task_id: str
    parent_task_id: Optional[str]  # For tracing delegated tasks
    step_number: int


class AgentSecurityLogger:
    def __init__(self, agent_id: str, instance_id: str):
        self.agent_id = agent_id
        self.instance_id = instance_id
        self.logger = logging.getLogger(f"agent.security.{agent_id}")

    def log_event(self, event: AgentEvent):
        """Write a structured security event to the log stream."""
        record = asdict(event)
        # Add log level based on sensitivity
        level = self._get_log_level(event)
        self.logger.log(level, json.dumps(record))

    def log_tool_call(
        self,
        tool_name: str,
        args: dict,
        result: Any,
        success: bool,
        session_context: dict
    ):
        """Specialized logging for tool calls — the highest-risk events."""
        # Never log secrets — sanitize args
        safe_args = self._sanitize_args(args)

        event = AgentEvent(
            event_type="tool_call",
            agent_id=self.agent_id,
            instance_id=self.instance_id,
            tool_name=tool_name,
            tool_args=safe_args,
            tool_result_summary=str(result)[:200],  # Truncate for log size
            tool_success=success,
            source_trust_level=session_context.get("trust_level", "unknown"),
            **session_context
        )
        self.log_event(event)

    def _sanitize_args(self, args: dict) -> dict:
        """Remove any credential-like values from tool arguments."""
        SENSITIVE_KEYS = {"password", "token", "secret", "key", "api_key", "auth"}
        return {
            k: "[REDACTED]" if any(s in k.lower() for s in SENSITIVE_KEYS) else v
            for k, v in args.items()
        }

    def _get_log_level(self, event: AgentEvent) -> int:
        if event.injection_scan_result in ("suspicious", "blocked"):
            return logging.WARNING
        if event.tool_name in ("execute_code", "send_email", "delete_record"):
            return logging.INFO  # High-sensitivity tools always at INFO+
        return logging.DEBUG
```

---

## Layer 2: Distributed Tracing

Individual log events tell you *what* happened. Traces tell you the *causal chain* — how one action led to another.

```python
from opentelemetry import trace
from opentelemetry.trace import SpanKind
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Initialize OpenTelemetry tracer
tracer = trace.get_tracer("agentic-ai-security")

class InstrumentedAgent:
    def run_task(self, user_input: str, session: dict) -> str:
        # Root span for the entire task
        with tracer.start_as_current_span(
            "agent.task",
            kind=SpanKind.SERVER,
            attributes={
                "agent.id": self.agent_id,
                "user.id": session["user_id"],
                "task.id": session["task_id"],
                "input.length": len(user_input),
                "input.trust_level": "user"
            }
        ) as task_span:
            try:
                result = self._execute(user_input, session)
                task_span.set_attribute("task.success", True)
                return result
            except SecurityException as e:
                task_span.set_attribute("task.security_violation", str(e))
                task_span.set_status(trace.Status(trace.StatusCode.ERROR))
                raise

    def _call_llm(self, prompt: str, model: str) -> str:
        with tracer.start_as_current_span(
            "agent.llm_call",
            attributes={
                "llm.model": model,
                "llm.prompt_hash": hashlib.sha256(prompt.encode()).hexdigest()[:16],
                "llm.input_tokens": count_tokens(prompt),
            }
        ) as llm_span:
            response = self.llm.complete(prompt)
            llm_span.set_attribute("llm.output_tokens", count_tokens(response))
            llm_span.set_attribute("llm.tool_calls_requested",
                                   len(extract_tool_calls(response)))
            return response

    def _execute_tool(self, tool_name: str, args: dict) -> Any:
        with tracer.start_as_current_span(
            f"agent.tool.{tool_name}",
            attributes={
                "tool.name": tool_name,
                "tool.arg_keys": list(args.keys()),
                # ⚠️ Never put arg VALUES in traces — they may contain secrets
            }
        ) as tool_span:
            result = self.toolkit.execute(tool_name, args)
            tool_span.set_attribute("tool.success", result.success)
            tool_span.set_attribute("tool.result_type", type(result.value).__name__)
            return result
```

---

## Layer 3: Anomaly Detection Rules

Rule-based detection catches known attack patterns. Define explicit rules for your most important scenarios:

```python
class AgentSecurityRules:
    """
    A collection of security detection rules.
    In production, these feed into your SIEM (Splunk, Elastic, etc.)
    """

    def check_all(self, events: list[AgentEvent]) -> list[SecurityAlert]:
        alerts = []
        for rule in self._get_rules():
            alert = rule(events)
            if alert:
                alerts.append(alert)
        return alerts

    def _get_rules(self):
        return [
            self.rule_injection_attempt,
            self.rule_unusual_tool_sequence,
            self.rule_data_exfiltration_pattern,
            self.rule_excessive_tool_calls,
            self.rule_after_hours_high_risk_action,
            self.rule_cross_tenant_access_attempt,
        ]

    def rule_injection_attempt(self, events):
        """Alert when injection patterns are detected in inputs."""
        for event in events:
            if event.injection_scan_result in ("suspicious", "blocked"):
                return SecurityAlert(
                    rule="injection_attempt",
                    severity="HIGH",
                    message=f"Injection attempt in session {event.session_id}",
                    context=event
                )

    def rule_unusual_tool_sequence(self, events):
        """
        Alert when an agent uses tools in an unusual order.
        Example: finance agent using web_search before query_finance_db is unusual.
        """
        tool_sequence = [e.tool_name for e in events if e.tool_name]
        known_sequences = {
            "customer-support-v2": [
                ["lookup_order", "create_ticket"],
                ["search_kb", "create_ticket"],
            ]
        }
        agent_id = events[0].agent_id if events else None
        expected = known_sequences.get(agent_id, [])
        if expected and not any(
            self._is_subsequence(seq, tool_sequence) for seq in expected
        ):
            return SecurityAlert(
                rule="unusual_tool_sequence",
                severity="MEDIUM",
                message=f"Unusual tool sequence for {agent_id}: {tool_sequence}",
                context={"sequence": tool_sequence}
            )

    def rule_data_exfiltration_pattern(self, events):
        """
        Alert on bulk data retrieval followed by external communication.
        Pattern: many read operations → send_email or webhook_call
        """
        read_count = sum(1 for e in events
                        if e.tool_name and "read" in e.tool_name.lower())
        has_external_send = any(
            e.tool_name in ("send_email", "webhook_call", "post_to_slack")
            for e in events
        )
        if read_count > 10 and has_external_send:
            return SecurityAlert(
                rule="data_exfiltration_pattern",
                severity="HIGH",
                message=f"Possible exfiltration: {read_count} reads followed by external send",
                context={"read_count": read_count}
            )

    def rule_excessive_tool_calls(self, events):
        """Alert when tool call count is anomalously high."""
        tool_calls = [e for e in events if e.event_type == "tool_call"]
        if len(tool_calls) > 50:  # Tune per agent baseline
            return SecurityAlert(
                rule="excessive_tool_calls",
                severity="MEDIUM",
                message=f"Agent made {len(tool_calls)} tool calls in one session",
                context={"count": len(tool_calls)}
            )

    def rule_after_hours_high_risk_action(self, events):
        """Alert on high-risk actions outside business hours."""
        HIGH_RISK_TOOLS = {"send_email", "delete_record", "execute_code",
                           "approve_payment", "create_user"}
        for event in events:
            if event.tool_name in HIGH_RISK_TOOLS:
                hour = int(event.timestamp[11:13])  # Extract hour from ISO timestamp
                if hour < 8 or hour >= 18:
                    return SecurityAlert(
                        rule="after_hours_high_risk",
                        severity="MEDIUM",
                        message=f"High-risk tool {event.tool_name} used at hour {hour}",
                        context=event
                    )
```

---

## Key Metrics to Track

```
SECURITY METRICS DASHBOARD:
─────────────────────────────────────────────────────────────────

🔴 CRITICAL (alert immediately):
  - Injection attempts per hour (threshold: >0)
  - Blocked tool calls (threshold: >0 for high-risk tools)
  - Agent revocation events (threshold: any)
  - Cross-tenant access attempts (threshold: any)

🟠 HIGH (alert if threshold exceeded):
  - Failed signature verifications per hour (threshold: >5)
  - Data exfiltration pattern detections per day (threshold: >0)
  - After-hours high-risk tool calls (threshold: >0)
  - Unusual tool sequences detected (threshold: >2/hour)

🟡 MEDIUM (review daily):
  - Token consumption per session (threshold: 3x baseline)
  - Tool call rate per session (threshold: 2x baseline)
  - RAG retrieval volume per user (threshold: 5x baseline)
  - Failed tool calls (threshold: >10% of calls)

📊 INFORMATIONAL (weekly trends):
  - Average session length
  - Tool usage distribution
  - Model latency percentiles
  - Cost per session trend
```

---

## Security Monitoring Checklist

```
LOGGING:
[ ] All agent events logged in structured JSON format
[ ] Every tool call logged with sanitized parameters
[ ] All LLM calls logged (model, token counts, prompt hash)
[ ] Security-relevant events at INFO level or above
[ ] Logs shipped to SIEM in real-time (not batched)
[ ] Logs retained for ≥ 90 days (1 year for compliance)

TRACING:
[ ] OpenTelemetry instrumentation on all agent components
[ ] Full task trace available (user request → final action)
[ ] Tool call spans include tool name and success/failure
[ ] LLM span includes model name and token counts
[ ] Traces linked to session and user IDs

ANOMALY DETECTION:
[ ] Injection attempt detection active
[ ] Unusual tool sequence detection configured
[ ] Data exfiltration pattern rules active
[ ] Excessive tool call rate monitoring
[ ] After-hours high-risk action monitoring

ALERTING:
[ ] Critical alerts page on-call within 5 minutes
[ ] High alerts create ticket and notify security team
[ ] Alerts include enough context to triage without digging
[ ] False positive rate tracked and rules tuned monthly
[ ] Incident response playbook linked in every alert
```

---

*← [Prev: Agent Identity & Attestation](./07-agent-identity.md) | [Next: Rate Limiting & Scope Controls →](./09-rate-limiting.md)*
