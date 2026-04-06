# 🤝 Secure Multi-Agent Orchestration

> **Phase 5 · Article 11 of 12** | ⏱️ 22 min read | 🏷️ `#defense` `#multi-agent` `#orchestration` `#trust`

---

## TL;DR

- Multi-agent systems multiply both capability and attack surface. When agents orchestrate other agents, **every agent is a potential attack vector for every other agent.**
- The key principles: **trust no agent unconditionally**, **validate delegation scope**, **isolate agent contexts**, and **maintain a minimal footprint.**
- This article covers the architectural patterns and code-level controls for building orchestrations that don't collapse under attack.

---

## The Multi-Agent Threat Landscape

```
SINGLE AGENT THREATS:           MULTI-AGENT ADDITIONAL THREATS:
────────────────────────────    ──────────────────────────────────────────
Prompt injection from user      + Injection via compromised orchestrator
Tool abuse                      + Agent impersonation (fake orchestrator)
Data exfiltration               + Privilege amplification via delegation
Memory poisoning                + Trust collapse (one breach = all breach)
                                + Confused deputy (good agent, bad task)
                                + Covert exfiltration through agent chain
```

In a chain `User → Orchestrator → Agent A → Agent B`, if **any** link is compromised, every downstream link is potentially affected.

---

## Architecture Pattern 1: Hierarchical with Trust Anchors

```
SECURE ORCHESTRATOR DESIGN:
─────────────────────────────────────────────────────────────────

             ┌─────────────────────────────┐
             │      TRUST ANCHOR            │
             │   (Policy Engine + Registry) │
             └──────────────┬──────────────┘
                            │ Validates all delegations
             ┌──────────────▼──────────────┐
             │      ORCHESTRATOR AGENT      │
             │   (Scope: planning only)     │
             │   (No direct tool access)    │
             └───────────┬─────┬───────────┘
                         │     │
               Signed    │     │   Signed
               message   │     │   message
                         │     │
             ┌───────────▼─┐ ┌─▼──────────────┐
             │  WORKER A   │ │   WORKER B      │
             │ (email only)│ │ (DB read only)  │
             └─────────────┘ └─────────────────┘

KEY DESIGN DECISIONS:
1. Orchestrator does NOT have direct tool access
   → Cannot directly execute malicious actions even if compromised
   → Must delegate to workers who independently validate scope

2. Workers validate that delegated tasks match original user scope
   → Prevents confused deputy via compromised orchestrator

3. All delegation messages are cryptographically signed
   → Prevents agent impersonation

4. Trust Anchor (policy engine) mediates all delegation
   → Centralized, auditable authorization
```

---

## Pattern 2: Capability-Gated Orchestration

Each agent only knows about other agents that it legitimately needs:

```python
class CapabilityGatedOrchestrator:
    """
    An orchestrator that can only see and delegate to agents
    that it is explicitly authorized to work with.

    The capability table is loaded from the Trust Anchor (registry)
    and cannot be modified at runtime.
    """

    def __init__(self, agent_id: str, registry: 'AgentRegistry'):
        self.agent_id = agent_id
        # Load the immutable capability map at startup
        self._allowed_delegations = registry.get_allowed_delegations(agent_id)

    def delegate(
        self,
        to_agent: str,
        task: dict,
        original_user_intent: str
    ) -> str:
        # 1. Can I delegate to this agent at all?
        if to_agent not in self._allowed_delegations:
            raise SecurityError(
                f"Not authorized to delegate to {to_agent}. "
                f"Allowed: {list(self._allowed_delegations.keys())}"
            )

        # 2. Is the task within my delegation scope to this agent?
        allowed_actions = self._allowed_delegations[to_agent]
        if task["action"] not in allowed_actions:
            raise SecurityError(
                f"Action '{task['action']}' not in allowed scope "
                f"for {to_agent}: {allowed_actions}"
            )

        # 3. Does this task serve the original user intent?
        if not self._is_consistent_with_intent(task, original_user_intent):
            raise SecurityError(
                f"Delegated task is inconsistent with original user intent"
            )

        # 4. Sign and send
        signed_task = self.identity_manager.sign_message(
            to_agent=to_agent,
            task_id=generate_task_id(),
            payload={**task, "original_user_intent": original_user_intent}
        )
        return self.message_bus.send(to_agent, signed_task)

    def _is_consistent_with_intent(self, task: dict, intent: str) -> bool:
        """Use a lightweight verification call to check intent alignment."""
        prompt = f"""
        User's original intent: "{intent}"
        Proposed delegated action: {json.dumps(task)}

        Is this action consistent with serving the user's stated intent?
        Answer YES or NO only.
        """
        response = self.verification_llm.complete(prompt, max_tokens=10)
        return response.strip().upper().startswith("YES")
```

---

## Pattern 3: Context Isolation

Agents in a multi-agent system should **not share memory context** by default:

```python
class IsolatedAgentContext:
    """
    Each agent gets a clean, isolated context.
    No agent can read another agent's reasoning or intermediate outputs.
    Only explicit, validated outputs are passed between agents.
    """

    def __init__(self, agent_id: str, task_id: str):
        self.agent_id = agent_id
        self.task_id = task_id
        # Each agent has its own isolated memory namespace
        self._memory_namespace = f"agent:{agent_id}:task:{task_id}"

    def get_context_for_agent(
        self,
        requesting_agent: str,
        output_type: str = "result_only"
    ) -> dict:
        """
        When another agent requests context from this agent,
        return only the explicitly approved output — not the full
        reasoning trace or intermediate steps.
        """
        if output_type == "result_only":
            return {
                "result": self._get_final_result(),
                "success": self._is_success(),
                # No intermediate reasoning, no tool call details
            }
        elif output_type == "summary":
            return {
                "summary": self._get_summary(),
                "actions_taken": self._get_action_count(),
                # Still no raw tool outputs
            }
        else:
            # Full context requires explicit registry permission
            if not registry.can_share_full_context(self.agent_id, requesting_agent):
                raise SecurityError(
                    f"Agent {requesting_agent} not authorized for full context of {self.agent_id}"
                )
            return self._get_full_context()
```

---

## Handling Untrusted Orchestrator Instructions

Worker agents must independently validate instructions — they cannot blindly trust even their orchestrator:

```python
class SecureWorkerAgent:
    """
    A worker agent that applies independent security controls
    regardless of who is giving it instructions.
    """

    # Actions that require human approval regardless of who asks
    ALWAYS_REQUIRE_HITL = {
        "send_email",
        "delete_record",
        "approve_payment",
        "create_user",
        "deploy_to_production"
    }

    # Actions that are always blocked (no instruction can enable these)
    ALWAYS_BLOCKED = {
        "access_system_files",
        "modify_security_config",
        "disable_audit_logging",
        "change_own_permissions"
    }

    async def execute_delegated_task(
        self,
        msg: 'SignedAgentMessage',
        registry: 'AgentRegistry'
    ):
        # 1. Verify the message signature
        is_valid, reason = self.identity_manager.verify_message(msg)
        if not is_valid:
            security_alert(
                "Invalid message signature", reason=reason, msg=msg
            )
            return TaskResult(success=False, error="Signature verification failed")

        # 2. Check that the sender is authorized to delegate to us
        can_delegate, reason = registry.verify_delegation(
            from_agent=msg.from_agent,
            to_agent=self.agent_id,
            requested_action=msg.payload.get("action")
        )
        if not can_delegate:
            security_alert(
                "Unauthorized delegation attempt",
                from_agent=msg.from_agent,
                reason=reason
            )
            return TaskResult(success=False, error="Delegation not authorized")

        action = msg.payload.get("action")

        # 3. Hard-block certain actions
        if action in self.ALWAYS_BLOCKED:
            security_alert(
                "Blocked action attempted", action=action, from_agent=msg.from_agent
            )
            return TaskResult(success=False, error=f"Action {action} is never permitted")

        # 4. Require HITL for sensitive actions
        if action in self.ALWAYS_REQUIRE_HITL:
            user_approved = await self.hitl_gate.request_approval(
                action=action,
                params=msg.payload.get("params"),
                task_context=msg.payload.get("original_user_intent")
            )
            if not user_approved:
                return TaskResult(success=False, error="User declined approval")

        # 5. Execute with minimal footprint
        return await self._execute(action, msg.payload.get("params"))
```

---

## Minimal Footprint Principle

Agents should acquire and expose only what they need for their current task:

```
MINIMAL FOOTPRINT IN MULTI-AGENT SYSTEMS:
─────────────────────────────────────────────────────────────────

1. TASK-SCOPED CREDENTIALS:
   Orchestrator: "Search for orders from the last 7 days."
   Worker response: Requests read-only access to orders table
                   for the past 7 days ONLY.
   (Not all orders, not write access, not other tables)

2. DATA MINIMIZATION IN RESULTS:
   Worker returns: ["order_id", "status", "customer_name"]
   (Not the full order record including payment details)

3. NO PERSISTENT STATE BETWEEN TASKS:
   Worker agent starts fresh on each task.
   No memory of previous tasks from other sessions.
   (Prevents cross-task data contamination)

4. NO STORING SENSITIVE DATA:
   If a worker processes a credit card number to verify an order,
   it does not store or log the full card number.
   Only the last 4 digits in the result.

5. TERMINATE CLEANLY:
   When a task is complete, the worker:
   - Releases its credentials
   - Clears its working memory
   - Closes all connections
   - Reports completion to the audit log
```

---

## Detecting Trust Collapse

Trust collapse: one agent's compromise enables cascading compromise of the entire system.

```python
class TrustCollapseDetector:
    """
    Monitors for patterns that indicate a trust collapse is occurring.
    """

    def check_for_collapse_indicators(self, events: list[dict]) -> list[dict]:
        alerts = []

        # Pattern: Unusual delegation chain depth
        max_depth = max(e.get("delegation_depth", 0) for e in events)
        if max_depth > 3:
            alerts.append({
                "severity": "HIGH",
                "indicator": "unusual_delegation_depth",
                "value": max_depth,
                "message": f"Delegation chain depth {max_depth} — possible runaway orchestration"
            })

        # Pattern: One agent calling many others rapidly
        orchestrator_calls = {}
        for e in events:
            if e.get("event_type") == "delegation":
                caller = e["from_agent"]
                orchestrator_calls[caller] = orchestrator_calls.get(caller, 0) + 1

        for agent, count in orchestrator_calls.items():
            if count > 10:
                alerts.append({
                    "severity": "MEDIUM",
                    "indicator": "fan_out_anomaly",
                    "agent": agent,
                    "count": count,
                    "message": f"Agent {agent} created {count} delegations — possible amplification attack"
                })

        # Pattern: Workers receiving instructions from agents they shouldn't know about
        for event in events:
            if event.get("event_type") == "unauthorized_delegation_attempt":
                alerts.append({
                    "severity": "CRITICAL",
                    "indicator": "unknown_orchestrator",
                    "from_agent": event["from_agent"],
                    "message": "Worker received instruction from unregistered agent"
                })

        return alerts
```

---

## Secure Multi-Agent Checklist

```
ARCHITECTURE:
[ ] Orchestrator has no direct tool access (delegates only)
[ ] Workers validate ALL instructions independently
[ ] Trust Anchor (registry/policy engine) mediates delegation
[ ] Agent capability map is immutable at runtime
[ ] Context isolation between agents (no shared state)

TRUST:
[ ] All inter-agent messages are cryptographically signed
[ ] Signing keys managed in secrets manager (not in agent code)
[ ] Unauthorized delegation attempts logged and alerted
[ ] Agent registry with expiry and revocation capability

SCOPE:
[ ] Workers independently check task scope vs original intent
[ ] Confused deputy protection (scope inheritance validation)
[ ] ALWAYS_REQUIRE_HITL list maintained and enforced
[ ] ALWAYS_BLOCKED actions enforced at worker level

MINIMAL FOOTPRINT:
[ ] Task-scoped credentials only (not standing access)
[ ] Workers return minimal result data (not raw dumps)
[ ] No persistent state between tasks
[ ] Agents terminate cleanly and release resources

MONITORING:
[ ] Delegation chain depth monitored
[ ] Fan-out anomaly detection (one agent spawning many)
[ ] Unauthorized delegation attempts alerted
[ ] Cross-session memory access blocked and alerted
```

---

*← [Prev: Red-Teaming Agents](./10-red-teaming-agents.md) | [Next: Data Minimization →](./12-data-minimization.md)*
