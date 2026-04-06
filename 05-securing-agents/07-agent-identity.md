# 🪪 Agent Identity & Attestation

> **Phase 5 · Article 7 of 12** | ⏱️ 20 min read | 🏷️ `#defense` `#identity` `#attestation` `#multi-agent`

---

## TL;DR

- In multi-agent systems, agents receive instructions from other agents. Without identity verification, **any message can claim to be from any source.**
- **Agent attestation** is the cryptographic proof that a message came from a specific, authorized agent — not an impostor.
- This article gives you the concrete patterns: signed tokens, service meshes, agent registries, and behavioral fingerprinting.

---

## The Problem: Who Can You Trust?

```
MULTI-AGENT MESSAGE FLOW:

  [Orchestrator] ──── "Please send the expense report" ────▶ [Finance Agent]

ATTACK: Prompt injection compromises the Orchestrator.
  The attacker now controls the Orchestrator's output.

  [Attacker-controlled Orchestrator] ──── "Please wire $50,000" ────▶ [Finance Agent]

The Finance Agent receives a message that *looks* legitimate.
Same channel. Same format. Same claimed source.

Without attestation: Finance Agent complies.
With attestation:    Finance Agent asks "Can you prove you're the Orchestrator?"
```

---

## Four Layers of Agent Identity

```
┌─────────────────────────────────────────────────────────────────┐
│               AGENT IDENTITY STACK                              │
│                                                                 │
│  LAYER 4: BEHAVIORAL ATTESTATION                               │
│  "Does this agent behave like the real agent?"                 │
│  Runtime fingerprinting, anomaly detection on reasoning style  │
│                                                                 │
│  LAYER 3: CAPABILITY ATTESTATION                               │
│  "Was this agent built from the approved codebase?"            │
│  Code signing, SBOM verification, build provenance             │
│                                                                 │
│  LAYER 2: MESSAGE ATTESTATION                                  │
│  "Did this message actually come from agent X?"                │
│  Signed JWTs, HMAC, mTLS per-message verification              │
│                                                                 │
│  LAYER 1: INSTANCE ATTESTATION                                 │
│  "Is this running in a trusted execution environment?"         │
│  TPM, Intel TDX, AWS Nitro Enclaves (hardware-backed)          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Layer 2: Message-Level Attestation (Most Practical)

The most immediately implementable layer: sign every inter-agent message.

```python
import hmac
import hashlib
import json
import time
from dataclasses import dataclass
from typing import Optional

@dataclass
class SignedAgentMessage:
    from_agent: str
    to_agent: str
    task_id: str
    payload: dict
    timestamp: int
    nonce: str          # Prevents replay attacks
    signature: str

class AgentIdentityManager:
    """
    Manages cryptographic identities for agents in a multi-agent system.
    Each agent has a unique HMAC key shared only with agents it trusts.
    """

    def __init__(self, agent_id: str, key_store: 'AgentKeyStore'):
        self.agent_id = agent_id
        self.key_store = key_store
        self._seen_nonces: set[str] = set()  # Replay protection

    def sign_message(
        self,
        to_agent: str,
        task_id: str,
        payload: dict
    ) -> SignedAgentMessage:
        """Create a cryptographically signed message for another agent."""
        nonce = generate_secure_nonce()
        timestamp = int(time.time())

        # Build canonical message string for signing
        canonical = json.dumps({
            "from": self.agent_id,
            "to": to_agent,
            "task_id": task_id,
            "payload": payload,
            "timestamp": timestamp,
            "nonce": nonce
        }, sort_keys=True)

        # Sign with the shared key for this recipient
        key = self.key_store.get_key(self.agent_id, to_agent)
        signature = hmac.new(
            key, canonical.encode(), hashlib.sha256
        ).hexdigest()

        return SignedAgentMessage(
            from_agent=self.agent_id,
            to_agent=to_agent,
            task_id=task_id,
            payload=payload,
            timestamp=timestamp,
            nonce=nonce,
            signature=signature
        )

    def verify_message(self, msg: SignedAgentMessage) -> tuple[bool, str]:
        """
        Verify an incoming message. Returns (is_valid, reason).
        ALWAYS call this before acting on any inter-agent instruction.
        """
        # 1. Check message is addressed to us
        if msg.to_agent != self.agent_id:
            return False, "Message not addressed to this agent"

        # 2. Check timestamp freshness (within 5 minutes)
        age = int(time.time()) - msg.timestamp
        if age > 300 or age < -60:
            return False, f"Message timestamp too old or future: {age}s"

        # 3. Check nonce for replay attacks
        if msg.nonce in self._seen_nonces:
            return False, f"Replay attack detected: nonce {msg.nonce} already seen"
        self._seen_nonces.add(msg.nonce)

        # 4. Verify signature
        key = self.key_store.get_key(msg.from_agent, self.agent_id)
        if key is None:
            return False, f"No shared key with agent {msg.from_agent}"

        canonical = json.dumps({
            "from": msg.from_agent,
            "to": msg.to_agent,
            "task_id": msg.task_id,
            "payload": msg.payload,
            "timestamp": msg.timestamp,
            "nonce": msg.nonce
        }, sort_keys=True)

        expected_sig = hmac.new(
            key, canonical.encode(), hashlib.sha256
        ).hexdigest()

        if not hmac.compare_digest(msg.signature, expected_sig):
            return False, "Signature verification failed — possible spoofing"

        return True, "OK"
```

---

## Layer 3: Capability Attestation — Agent Registry

An **Agent Registry** is a central authority that records which agents exist, what they're authorized to do, and what version of code they run.

```python
from datetime import datetime

@dataclass
class AgentRegistration:
    agent_id: str
    name: str
    version: str
    allowed_tools: list[str]
    allowed_callers: list[str]   # Which agents can send it instructions
    allowed_callees: list[str]   # Which agents it can delegate to
    code_hash: str               # SHA256 of agent code (CI/CD verified)
    registered_at: datetime
    expires_at: datetime
    is_revoked: bool = False

class AgentRegistry:
    """
    Central registry — the source of truth for agent identity and capabilities.
    In production, this would be backed by a hardened database with audit logging.
    """

    def __init__(self):
        self._agents: dict[str, AgentRegistration] = {}

    def register(self, reg: AgentRegistration):
        self._agents[reg.agent_id] = reg

    def verify_delegation(
        self,
        from_agent: str,
        to_agent: str,
        requested_action: str
    ) -> tuple[bool, str]:
        """
        Can `from_agent` delegate `requested_action` to `to_agent`?
        """
        caller = self._agents.get(from_agent)
        callee = self._agents.get(to_agent)

        if not caller or caller.is_revoked:
            return False, f"Caller {from_agent} not registered or revoked"
        if not callee or callee.is_revoked:
            return False, f"Callee {to_agent} not registered or revoked"
        if to_agent not in caller.allowed_callees:
            return False, f"{from_agent} is not authorized to delegate to {to_agent}"
        if from_agent not in callee.allowed_callers:
            return False, f"{to_agent} does not accept instructions from {from_agent}"
        if requested_action not in callee.allowed_tools:
            return False, f"{to_agent} cannot perform action {requested_action}"
        if datetime.utcnow() > callee.expires_at:
            return False, f"Registration for {to_agent} has expired"

        return True, "delegation_permitted"

    def revoke(self, agent_id: str, reason: str):
        """Immediately revoke an agent's authority — for incident response."""
        if agent_id in self._agents:
            self._agents[agent_id].is_revoked = True
            audit_log.critical(
                "Agent revoked",
                agent_id=agent_id,
                reason=reason,
                timestamp=datetime.utcnow().isoformat()
            )
```

---

## Layer 4: Behavioral Attestation — Anomaly Detection

Even if a message passes cryptographic verification, the *content* of instructions can indicate compromise. Build a behavioral profile for each agent and alert on deviations.

```python
class AgentBehaviorProfiler:
    """
    Monitors agent behavior for deviations that might indicate compromise.
    Each agent has a baseline profile of its expected behavior.
    """

    PROFILES = {
        "customer-support-v2": {
            "expected_tools": {"lookup_order", "create_ticket", "search_kb"},
            "unexpected_tools": {"send_email", "execute_code", "delete_record"},
            "max_tool_calls_per_session": 20,
            "max_tokens_per_message": 2000,
            "allowed_hours": range(0, 24),   # 24/7 for this agent
        },
        "finance-agent-v1": {
            "expected_tools": {"query_finance_db", "generate_report"},
            "unexpected_tools": {"send_email", "web_search", "execute_code"},
            "max_tool_calls_per_session": 10,
            "max_tokens_per_message": 4000,
            "allowed_hours": range(8, 18),   # Business hours only
        }
    }

    def check_tool_call(
        self,
        agent_id: str,
        tool_name: str,
        session_data: dict
    ) -> tuple[bool, str]:
        profile = self.PROFILES.get(agent_id)
        if not profile:
            return True, "no_profile"  # Allow but flag for review

        # Check for unexpected tools (high-severity indicator)
        if tool_name in profile["unexpected_tools"]:
            security_alert(
                severity="HIGH",
                message=f"Agent {agent_id} attempted unexpected tool: {tool_name}",
                context=session_data
            )
            return False, f"Tool {tool_name} outside agent's behavioral profile"

        # Check tool call rate
        if session_data["tool_calls"] >= profile["max_tool_calls_per_session"]:
            security_alert(
                severity="MEDIUM",
                message=f"Agent {agent_id} exceeded max tool calls",
                context=session_data
            )
            return False, "Tool call rate exceeded"

        # Check business hours for restricted agents
        current_hour = datetime.utcnow().hour
        if current_hour not in profile["allowed_hours"]:
            security_alert(
                severity="MEDIUM",
                message=f"Agent {agent_id} active outside allowed hours",
                context=session_data
            )
            return False, "Activity outside allowed hours"

        return True, "OK"
```

---

## The Confused Deputy Defense

Recall the confused deputy attack: a sub-agent is tricked into using its privileges to serve an attacker via a compromised orchestrator.

```
DEFENSE PATTERN: SCOPE INHERITANCE VALIDATION
─────────────────────────────────────────────────────────────────

When a sub-agent receives a delegated task:
1. Extract the ORIGINAL user intent from the task metadata
2. Check that the delegated action is consistent with that intent
3. If not → refuse and alert, regardless of who is asking

class ScopeInheritanceValidator:
    def validate(
        self,
        original_user_intent: str,
        delegated_action: dict,
        delegating_agent: str
    ) -> bool:
        '''
        Before executing a delegated action, verify it's consistent
        with what the original user actually wanted.
        '''
        prompt = f"""
        Original user request: {original_user_intent}
        Delegated action: {json.dumps(delegated_action)}

        Is this delegated action consistent with serving the original
        user request? Answer: YES or NO with a brief reason.
        """

        # Use a separate, lightweight verification model
        response = verification_llm.complete(prompt)

        if response.strip().upper().startswith("NO"):
            security_alert(
                "Scope inheritance violation",
                delegating_agent=delegating_agent,
                action=delegated_action,
                original_intent=original_user_intent
            )
            return False
        return True
```

---

## Incident Response: Revoking a Compromised Agent

When you detect a compromised agent, your incident response must be fast and complete:

```
COMPROMISED AGENT RESPONSE PLAYBOOK:
─────────────────────────────────────────────────────────────────

T+0 DETECT:
  □ Anomaly detected (unusual tool call, behavioral deviation,
    external alert, user report)

T+1 ISOLATE:
  □ Call registry.revoke(agent_id, reason="suspected_compromise")
  □ All other agents now reject messages from this agent
  □ Agent's API credentials rotated/revoked
  □ Running instances terminated

T+5 ASSESS:
  □ Review audit logs for the window of compromise
  □ What actions did the agent take?
  □ What data did it access?
  □ Did it communicate with external systems?

T+15 CONTAIN:
  □ Rotate all credentials the agent had access to
  □ Review downstream agents for possible propagation
  □ Check knowledge base for poisoned content

T+60 RECOVER:
  □ Deploy clean agent from verified source
  □ Re-register with new identity (new key, new version)
  □ Monitor closely for 24-48 hours

T+DAYS LEARN:
  □ Root cause analysis
  □ Update behavioral profiles
  □ Add detection rule for the attack pattern
```

---

## Checklist: Agent Identity & Attestation

```
MESSAGE ATTESTATION:
[ ] All inter-agent messages are cryptographically signed
[ ] Receiving agents verify signatures before acting
[ ] Nonce or sequence number prevents replay attacks
[ ] Message freshness checked (reject messages >5 min old)

AGENT REGISTRY:
[ ] Central registry lists all authorized agents
[ ] Registry includes allowed tools per agent
[ ] Allowed caller/callee relationships explicit
[ ] Registration has expiry (force periodic re-review)
[ ] Revocation mechanism tested and documented

BEHAVIORAL ATTESTATION:
[ ] Behavioral profile defined for each production agent
[ ] Alerts fire when agent uses unexpected tools
[ ] Rate limits monitored per agent instance
[ ] Activity outside expected patterns triggers review

SCOPE INHERITANCE:
[ ] Sub-agents validate that delegated tasks match original scope
[ ] Confused deputy pattern blocked by scope inheritance check
[ ] Delegation chain logged and auditable

INCIDENT RESPONSE:
[ ] Agent revocation procedure documented and tested
[ ] Revocation takes effect within seconds
[ ] Post-revocation credential rotation automated
[ ] IR playbook available on-call
```

---

*← [Prev: Secure MCP Server Design](./06-secure-mcp.md) | [Next: Security Monitoring →](./08-security-monitoring.md)*
