# 👥 Human-in-the-Loop Design Patterns

> **Phase 5 · Article 4 of 12** | ⏱️ 20 min read | 🏷️ `#human-oversight` `#hitl` `#approval-gates` `#risk-based-control`

---

## TL;DR

- **Human-in-the-Loop (HITL)** adds a human checkpoint before critical agent actions; it's a *risk mitigation layer*, not a replacement for other controls.
- **Four HITL patterns**: (1) Approval gate — block until human approves, (2) Sampling review — random post-execution audit, (3) Exception escalation — human only for edge cases, (4) Dead man's switch — timeout if no human response.
- **LangGraph's `interrupt_before` pattern** enables HITL workflows without custom state management.
- **Approval fatigue is real** — if >80% of requests are approved, humans stop reading; design thresholds carefully.

---

## When HITL Is Mandatory vs. Optional

```
DECISION MATRIX: WHEN HITL IS NEEDED
─────────────────────────────────────────────────────────────────

Risk Level  |  Tool Type              |  HITL Required?  |  Pattern
────────────┼────────────────────────┼──────────────────┼──────────────
CRITICAL    |  Database delete        |  YES (always)    |  Approval gate
            |  Credential rotation    |                  |
            |  Production deploy      |                  |
────────────┼────────────────────────┼──────────────────┼──────────────
HIGH        |  Database write         |  YES (default)   |  Approval gate
            |  API key generation     |  unless          |  or sampling
            |  Large data export      |  low-risk context|
────────────┼────────────────────────┼──────────────────┼──────────────
MEDIUM      |  Email send             |  Optional        |  Exception
            |  Slack message          |  (context-based) |  escalation
            |  Report generation      |                  |  or sampling
────────────┼────────────────────────┼──────────────────┼──────────────
LOW         |  File read (public)     |  Optional        |  Sampling
            |  API query (read-only)  |  (monitoring)    |  or none
            |  Report rendering       |                  |
────────────┴────────────────────────┴──────────────────┴──────────────

CONTEXT FACTORS that increase HITL requirement:
  • Untrusted input source (user input, web scraping)
  • Large financial amounts involved
  • Affects multiple users
  • Irreversible action
  • Regulatory/compliance-sensitive
  • First time agent is running this action
```

---

## The Four HITL Patterns

### Pattern 1: Approval Gate (Blocking)

**Most secure but slowest. Human must explicitly approve before action executes.**

```
FLOW DIAGRAM:
────────────────────────────────────────────────────────────────
Agent: "I want to delete database entries matching: user_id > 1000"
         │
         ├─→ [VALIDATION] Passes schema check
         │
         ├─→ [RISK ASSESSMENT] Marked HIGH-RISK: affects 5,000 records
         │
         ├─→ [INTERRUPT] Agent pauses
         │     ├─→ Human notified: "Approval needed for risky action"
         │     │    Email: "Agent is requesting to delete 5000 records"
         │     │    Showing: query, count, sample records
         │     │
         │     ├─→ Human reviews: "Wait, that's wrong. Query should be user_id > 10000"
         │     │
         │     ├─→ Human response: REJECT
         │
         ├─→ [RESUME] Agent receives rejection
         │     └─→ Agent corrects query, re-requests with safer parameters
         │
         └─→ [APPROVE] On second request, human reviews and approves
             ├─→ Action executes
             └─→ Audit log: "Approved by user@company.com at 2026-04-05 14:32:10 UTC"

TIME: Agent is blocked waiting for human response
TIMEOUT: If human doesn't respond in 1 hour → action is automatically REJECTED
```

**Best for**: Financial transactions, deletions, credential changes, deployments

### Pattern 2: Sampling Review (Post-execution Audit)

**Fastest but requires strong monitoring. Agent executes, humans audit a random sample afterward.**

```
FLOW DIAGRAM:
────────────────────────────────────────────────────────────────
Agent: "I want to send 1000 notification emails"
         │
         ├─→ [RISK CHECK] Medium-risk (can spam, but reversible)
         │
         ├─→ [ACTION] Execute (send emails immediately)
         │     └─→ Log: execution_id=98234, timestamps, recipients
         │
         ├─→ [SAMPLING] Post-execution
         │     └─→ Randomly select 5% of actions (50 emails) for review
         │
         ├─→ [AUDIT] Human reviews sample
         │     ├─→ Email 1: looks OK
         │     ├─→ Email 2: looks OK
         │     ├─→ Email 3: SPAM-like content detected by human
         │     │
         │     └─→ Human raises alert: "3 of 50 sampled emails look suspicious"
         │
         ├─→ [ESCALATION] If >10% of sample flagged, trigger investigation
         │     └─→ Notify: "Agent email pattern changed, investigate run 98234"
         │
         └─→ [REMEDIATION] If harmful, take corrective action
             ├─→ Halt future runs by this agent
             ├─→ Analyze audit logs
             └─→ Review agent's last reasoning steps

TIME: No delay (action executes immediately)
DETECTION LAG: Audit happens within 1 hour (or per batch)
CATCH RATE: ~5% of actions are directly audited, but patterns matter
```

**Best for**: High-volume actions, reversible changes, notification-like operations

### Pattern 3: Exception Escalation (Conditional HITL)

**Balanced. Agent executes normally, but certain *edge cases* trigger human review.**

```
FLOW DIAGRAM:
────────────────────────────────────────────────────────────────
Agent: "I want to approve customer refund"
         │
         ├─→ [RULES CHECK] Agent evaluates refund policy
         │     ├─→ Standard refund (<$100, within 30 days) → APPROVE (no HITL)
         │     ├─→ Large refund (>$1000) → ESCALATE
         │     ├─→ Refund outside policy window → ESCALATE
         │     └─→ Refund for VIP customer → ESCALATE (exception)
         │
         ├─→ [ESCALATION PATH]
         │     └─→ For LARGE ($2,500) refund:
         │         ├─→ Create ticket for approval
         │         ├─→ Notify manager: "High-value refund pending approval"
         │         └─→ Agent waits (timeout = 2 hours)
         │
         ├─→ [HUMAN DECISION]
         │     ├─→ Manager reviews customer history
         │     ├─→ Manager approves or rejects
         │     └─→ Audit log records manager ID and reason
         │
         └─→ [RESUME]
             ├─→ If APPROVED: refund processes
             └─→ If REJECTED: agent informs customer with manager's reason

TIME: Only for exceptions (maybe 5-10% of requests)
LATENCY: Most requests complete instantly, exceptions have 2-hour SLA
```

**Best for**: Policy-based decisions, context-dependent risk, business rule exceptions

### Pattern 4: Dead Man's Switch (Timeout Escalation)

**For long-running agents. If agent doesn't "check in," assume compromise and stop.**

```
FLOW DIAGRAM:
────────────────────────────────────────────────────────────────
Agent: "Starting 8-hour data processing job"
         │
         ├─→ [HEARTBEAT] Agent configured to send heartbeat every 15 minutes
         │     ├─→ 00:00 ✓ "Agent is alive, processing step 1"
         │     ├─→ 00:15 ✓ "Still running, step 2"
         │     ├─→ 00:30 ✓ "Still running, step 3"
         │     ├─→ 00:45 ✗ [HEARTBEAT MISSED]
         │     ├─→ 01:00 ✗ [STILL MISSED] Timeout triggered
         │     │
         │     └─→ DEAD MAN'S SWITCH ACTIVATED
         │
         ├─→ [ESCALATION]
         │     ├─→ Kill running job
         │     ├─→ Alert on-call: "Agent heartbeat lost, killed job 12345"
         │     ├─→ Create incident: "Possible agent compromise or hang"
         │     └─→ Prevent agent from restarting automatically
         │
         ├─→ [INVESTIGATION]
         │     ├─→ Human checks: logs, resource usage, network connections
         │     ├─→ Finds: agent was stuck in infinite loop (bug, not attack)
         │     └─→ Fix deployed
         │
         └─→ [RESUME] After approval, agent restarts with monitoring

TIME: Immediate on heartbeat miss + grace period (15 min)
RESPONSE: <5 minutes for alert, <1 hour for human investigation
```

**Best for**: Long-running batch jobs, critical background tasks, untrusted external data processing

---

## LangGraph Implementation: Interrupt Pattern

```python
# ✅ RIGHT: Using LangGraph's interrupt_before pattern

from langgraph.graph import StateGraph
from langchain.schema import HumanMessage
from typing import Annotated
import operator

class AgentState:
    """Mutable state that persists across interrupts"""

    def __init__(self):
        self.messages: list = []
        self.tool_calls: list = []
        self.pending_approval: Optional[dict] = None
        self.approval_response: Optional[str] = None

def reasoning_node(state: AgentState) -> AgentState:
    """Agent reasoning step"""
    # ... agent thinks about what to do ...
    tool_call = {
        "tool": "delete_database",
        "params": {"query": "DELETE FROM users WHERE id > 1000"},
        "risk_level": "HIGH"
    }
    state.tool_calls.append(tool_call)
    state.pending_approval = tool_call
    return state

def execution_node(state: AgentState) -> AgentState:
    """Execute approved action"""
    if state.approval_response == "approved":
        tool = state.pending_approval
        # ... execute tool ...
        state.messages.append({
            "role": "assistant",
            "content": f"Executed {tool['tool']} with params {tool['params']}"
        })
    else:
        state.messages.append({
            "role": "assistant",
            "content": "Action was rejected by human reviewer."
        })
    state.pending_approval = None
    return state

# Build graph with interrupt points
graph = StateGraph(AgentState)
graph.add_node("reasoning", reasoning_node)
graph.add_node("execution", execution_node)

# KEY: interrupt_before ensures execution pauses before running
graph.add_edge("reasoning", "execution", interrupt_before="execution")

app = graph.compile()

# Usage in agent loop:
def run_agent_with_approval():
    state = AgentState()

    while True:
        # Run agent until it hits interrupt point
        state = app.invoke(state)

        # Check if there's a pending approval needed
        if state.pending_approval:
            tool = state.pending_approval
            print(f"⚠️  Agent wants to {tool['tool']}")
            print(f"   Risk: {tool['risk_level']}")
            print(f"   Params: {tool['params']}")

            # Wait for human response
            approval = input("Approve? (yes/no): ").lower()
            state.approval_response = "approved" if approval == "yes" else "rejected"

            # Resume from interrupt point
            state = app.invoke(state)  # Resumes at execution_node

        else:
            # No more approvals needed
            break

    return state
```

### Advanced: Risk-Based HITL Thresholds

```python
from enum import Enum
from dataclasses import dataclass

class RiskLevel(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

@dataclass
class ApprovalConfig:
    """Controls when HITL is triggered"""
    critical_tools: list = None  # ["delete_database", "rotate_credentials"]
    high_risk_threshold: float = 5000.0  # Dollar amount
    require_approval_for_risk: dict = None  # Risk level → bool

    def __post_init__(self):
        if self.critical_tools is None:
            self.critical_tools = []
        if self.require_approval_for_risk is None:
            self.require_approval_for_risk = {
                RiskLevel.CRITICAL: True,
                RiskLevel.HIGH: True,
                RiskLevel.MEDIUM: False,
                RiskLevel.LOW: False,
            }

class ApprovalGate:
    """
    Determines if a tool call needs human approval based on risk.
    """

    def __init__(self, config: ApprovalConfig):
        self.config = config

    def assess_risk(self, tool_name: str, params: dict) -> RiskLevel:
        """Assess risk level of a tool call"""

        # Rule 1: Critical tools always HIGH/CRITICAL risk
        if tool_name in self.config.critical_tools:
            return RiskLevel.CRITICAL

        # Rule 2: Financial amount thresholds
        if "amount" in params:
            amount = params["amount"]
            if amount > self.config.high_risk_threshold:
                return RiskLevel.HIGH
            elif amount > self.config.high_risk_threshold * 0.5:
                return RiskLevel.MEDIUM

        # Rule 3: Data scope
        if "affected_records" in params:
            records = params["affected_records"]
            if records > 10000:
                return RiskLevel.HIGH
            elif records > 1000:
                return RiskLevel.MEDIUM

        # Default: LOW
        return RiskLevel.LOW

    def requires_approval(self, tool_name: str, params: dict) -> bool:
        """Check if tool call requires human approval"""
        risk = self.assess_risk(tool_name, params)
        return self.config.require_approval_for_risk.get(risk, False)

    def get_timeout(self, risk_level: RiskLevel) -> int:
        """Get approval timeout based on risk"""
        timeouts = {
            RiskLevel.CRITICAL: 3600,  # 1 hour for critical
            RiskLevel.HIGH: 1800,      # 30 minutes
            RiskLevel.MEDIUM: 600,     # 10 minutes
            RiskLevel.LOW: 60,         # 1 minute (for edge cases)
        }
        return timeouts.get(risk_level, 3600)

# Usage:
config = ApprovalConfig(
    critical_tools=["delete_database", "rotate_credentials", "enable_admin"],
    high_risk_threshold=5000.0,
    require_approval_for_risk={
        RiskLevel.CRITICAL: True,
        RiskLevel.HIGH: True,
        RiskLevel.MEDIUM: False,
        RiskLevel.LOW: False,
    }
)

gate = ApprovalGate(config)

# Check a tool call
tool_call = {
    "tool": "transfer_funds",
    "params": {"amount": 10000.0, "recipient": "external_account"}
}

risk = gate.assess_risk(tool_call["tool"], tool_call["params"])
needs_approval = gate.requires_approval(tool_call["tool"], tool_call["params"])
timeout = gate.get_timeout(risk)

print(f"Risk: {risk.name}")
print(f"Requires approval: {needs_approval}")
print(f"Timeout: {timeout} seconds")
```

---

## The Approval Fatigue Problem

```
APPROVAL FATIGUE RESEARCH:
─────────────────────────────────────────────────────────────────

Study: Continuous Security Decisions (CyLab, Carnegie Mellon)
Finding: When humans see >80% approval rate, they stop reading
         and rubber-stamp approvals without review.

Implication: If your system has:
  • 100 requests/day
  • 85 are approved automatically (low-risk)
  • 15 need human approval
  • Human sees notification every 10 minutes
  • Human skims each request (15 seconds per review)
  → Fatigue sets in after 1-2 weeks
  → Approval rate goes to ~95% (humans just clicking "yes")

SOLUTION 1: Reduce notification frequency
  • Batch approvals (review 10 at once, 1x per hour)
  • Aggregate low-risk requests ("10 log queries approved in batch")
  • Different channels by risk:
    - CRITICAL: phone call, page on-call
    - HIGH: Slack + email
    - MEDIUM: daily digest only

SOLUTION 2: Automatic escalation for unusual patterns
  • If 3rd request from same agent in 5 minutes → auto-escalate
  • If action violates business rules → auto-escalate
  • If action is new (first time this agent did this) → auto-escalate
  • Let humans focus on truly exceptional cases

SOLUTION 3: AI-assisted review
  • LLM analyzes request + context
  • Provides summary + risk assessment
  • Human reviews LLM analysis, not raw data
  • Example: "Agent wants to delete 5000 test records (risk: LOW)"
           instead of showing entire SQL query
```

---

## HITL Anti-Patterns to Avoid

```
❌ ANTI-PATTERN 1: Approval Deadlock
───────────────────────────────────────
Agent: "Approval needed"
Human: (on vacation, no response)
Agent: (stuck waiting forever)
System: (degraded for hours/days)

FIX: Set explicit timeout + escalation
  • Timeout = 1 hour maximum
  • Escalate to backup approver if timeout
  • Log who was responsible for delay

❌ ANTI-PATTERN 2: Unclear Approval Questions
───────────────────────────────────────────
Human sees: "Approve action? Y/N"
Human thinks: "What action? What will it do?"
Result: Human approves blindly or delays for investigation

FIX: Provide context in every approval request
  • Tool name + what it does
  • Specific parameters + data affected
  • Why agent wants to do this (reasoning)
  • Sample impact (e.g., "Will affect 5,000 users")
  • Recommendation ("This is routine, <2% risk")

❌ ANTI-PATTERN 3: Too Many Approvals
───────────────────────────────────────
Approval needed for: read_file, send_email, query_database, ...
Result: 500 approval requests/day
Human: never reads them

FIX: Be selective
  • Only HIGH and CRITICAL actions need approval
  • Use sampling for MEDIUM risk
  • Automate LOW risk

❌ ANTI-PATTERN 4: Invisible Rejections
───────────────────────────────────────
Agent receives: REJECTED
Agent: (no feedback on why)
Agent: retries same action → rejected again
Result: infinite loop, wasted resources

FIX: Always return rejection reason
  • "Rejected: Amount exceeds $10,000 limit (requested: $15,000)"
  • "Rejected: Policy violation - refunds only within 30 days"
  • Agent adjusts and retries with new params
```

---

## HITL Checklist

```
HUMAN-IN-THE-LOOP IMPLEMENTATION CHECKLIST
────────────────────────────────────────────────────────────────

DECISION PROCESS:
  ☐ Documented: Which actions require approval
  ☐ Risk scoring: Criteria for assigning risk levels
  ☐ Escalation paths: Who approves what
  ☐ SLA defined: Expected approval time by risk level

IMPLEMENTATION:
  ☐ Interrupt mechanism (e.g., LangGraph interrupt_before)
  ☐ State persisted: Agent state survives resume
  ☐ Timeout enforced: Approval doesn't block forever
  ☐ Timeout action: Auto-approve, auto-reject, or escalate?

USER EXPERIENCE:
  ☐ Notification clear: Action description, risk, recommendation
  ☐ Easy approval: Single click or simple response
  ☐ Audit trail: Approval decision logged + who approved
  ☐ Feedback: Human's decision returned to agent

MONITORING:
  ☐ Approval rates tracked: % approved vs. rejected by action type
  ☐ Response times tracked: How long humans take to approve
  ☐ Fatigue detection: Alert if approval rate > 80%
  ☐ Escalation tracking: How often timeouts occur

EDGE CASES:
  ☐ Approver unavailable: Escalation to backup
  ☐ Conflicting requests: Prevent race conditions
  ☐ Agent crashed: Resume from last state (use persistent state)
  ☐ High-volume surge: Queue management, no resource exhaustion
```

---

## Further Reading

- **LangGraph Interrupts**: https://langchain-ai.github.io/langgraph/concepts/low_level_behaviour/#interrupt
- **Human-in-the-Loop ML**: https://en.wikipedia.org/wiki/Human-in-the-loop
- **Approval Fatigue**: Carnegie Mellon CyLab research
- **NIST: Human Review in AI**: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-188.pdf
- **Approval Systems Design**: "Guidelines for Secure Human-Centered AI" papers
- **Dead Man's Switch Patterns**: Kubernetes watchdog, CI/CD timeout patterns

---

← [Prev: Sandboxing Tool Execution](./03-sandboxing.md) | [Next: Prompt Hardening →](./05-prompt-hardening.md)
