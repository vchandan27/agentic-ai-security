# 📋 NIST AI Risk Management Framework (AI RMF)

> **Phase 6 · Article 4** | ⏱️ 20 min read | 🏷️ `#frameworks` `#nist` `#risk-management` `#governance`

---

## TL;DR

- The **NIST AI RMF** (published January 2023) is the US federal standard for managing risks from AI systems. It's rapidly becoming the de-facto enterprise AI governance framework globally.
- It's organized around **four core functions: GOVERN, MAP, MEASURE, MANAGE** — a lifecycle approach rather than a checklist.
- Unlike OWASP or ATLAS which focus on technical attacks, the AI RMF addresses **organizational processes, governance, and accountability** for AI systems.

---

## What Is the NIST AI RMF?

Published by the National Institute of Standards and Technology (NIST) in January 2023, the AI Risk Management Framework provides:

1. **A common language** for discussing AI risks across technical and non-technical stakeholders
2. **A structured process** for identifying, measuring, and managing AI risks across the AI lifecycle
3. **Voluntary guidelines** (not mandatory, but referenced in legislation and contracts)
4. **A playbook** of actions organizations can take for each function

```
NIST AI RMF SCOPE:
─────────────────────────────────────────────────────────────────

Covers risks across the full AI lifecycle:
  DESIGN → DEVELOP → DEPLOY → MONITOR → RETIRE

Covers multiple risk dimensions:
  Technical risks (accuracy, robustness, security)
  Ethical risks (bias, fairness, transparency)
  Legal/compliance risks (liability, regulatory)
  Operational risks (reliability, safety)

Who uses it:
  AI developers building systems
  Organizations deploying AI systems
  Regulators assessing AI governance
  Auditors evaluating AI risk posture
```

---

## The Four Core Functions

```
┌─────────────────────────────────────────────────────────────────────┐
│                   NIST AI RMF CORE FUNCTIONS                        │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                       GOVERN                                │   │
│  │  Establish the organizational culture, policies, and        │   │
│  │  accountability structures for AI risk management.          │   │
│  │  "Who is responsible? What are the rules?"                  │   │
│  └───────────────────────────┬─────────────────────────────────┘   │
│                              │                                      │
│         ┌────────────────────┼────────────────────┐                │
│         ▼                    ▼                    ▼                │
│  ┌────────────┐       ┌────────────┐       ┌────────────┐         │
│  │    MAP     │       │  MEASURE   │       │   MANAGE   │         │
│  │            │       │            │       │            │         │
│  │ Identify   │       │ Analyze    │       │ Prioritize │         │
│  │ and        │       │ and assess │       │ treat and  │         │
│  │ categorize │       │ the risks  │       │ monitor    │         │
│  │ AI risks   │       │ identified │       │ the risks  │         │
│  └────────────┘       └────────────┘       └────────────┘         │
│                                                                     │
│  "What are the risks?" → "How bad are they?" → "What do we do?"   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GOVERN: Establishing AI Risk Culture

GOVERN is the foundation — without governance, MAP/MEASURE/MANAGE don't stick.

```
GOVERN SUBCATEGORIES:
─────────────────────────────────────────────────────────────────

GV-1: Policies, Processes, and Procedures
  □ AI risk policy defined and approved by leadership
  □ AI development standards documented
  □ Review and approval process for AI deployments

GV-2: Accountability
  □ AI Risk Owner assigned for each AI system
  □ Clear accountability chain (who is responsible if something goes wrong?)
  □ Roles and responsibilities documented

GV-3: Organizational Culture
  □ AI literacy training for all staff involved with AI
  □ "Speak up" culture for AI concerns (no blame culture)
  □ Ethics and responsible AI principles documented

GV-4: Organizational Teams and Expertise
  □ AI governance committee exists
  □ Cross-functional AI risk review (engineering, legal, HR, security)
  □ External expertise consulted when needed

GV-5: Organizational Policies on AI Risk
  □ AI inventory maintained (catalog of all AI systems in use)
  □ Third-party AI (vendor tools) subject to same standards
  □ Supply chain AI risk addressed

GV-6: Policies and Procedures for AI Risk Management
  □ Risk appetite statement for AI defined
  □ Risk escalation paths clear
  □ Regular AI risk reviews scheduled
```

**For agentic AI specifically:**

```
GOVERNANCE QUESTIONS FOR AGENTIC AI:
─────────────────────────────────────────────────────────────────
□ Who is the accountable owner of each deployed agent?
□ What approval process exists before an agent gains new tools?
□ How are agent incidents reported and escalated?
□ What is the process for emergency agent shutdown?
□ Who can authorize an agent to take a new class of action?
□ How are third-party MCP servers assessed before use?
```

---

## MAP: Identifying AI Risks

MAP is about understanding what risks exist for your specific AI system in your specific context.

```
MAP SUBCATEGORIES:
─────────────────────────────────────────────────────────────────

MP-1: Context Establishment
  □ AI system's purpose and intended use cases documented
  □ Intended users and use environments identified
  □ Intended benefits documented

MP-2: Scientific and Technological Risks
  □ Technical risks identified (model errors, robustness, security)
  □ Known vulnerabilities (prompt injection, hallucination) documented

MP-3: AI System Context
  □ AI system's context of use documented
  □ Downstream impacts mapped
  □ Stakeholders who may be affected identified

MP-4: Risk Tolerance
  □ Risk tolerance for this AI system defined
  □ "What level of error is acceptable?"
  □ "What actions should NEVER be autonomous?"

MP-5: AI System Categorization
  □ AI system categorized by risk level
    - Minimal risk
    - Limited risk
    - High risk (as defined by EU AI Act)
    - Unacceptable risk (prohibited)
```

**Risk categorization for agentic AI:**

```
AGENTIC AI RISK LEVELS:
─────────────────────────────────────────────────────────────────

MINIMAL RISK (self-service, read-only):
  Example: Agent that answers questions about product documentation
  Failure impact: Wrong answer, user frustration
  Autonomy level: L1-L2

LIMITED RISK (can communicate, limited writes):
  Example: Customer service agent that can create support tickets
  Failure impact: Wrong ticket, minor operational impact
  Autonomy level: L2-L3

HIGH RISK (financial, medical, legal, employment):
  Example: Agent that approves loans, processes medical orders
  Failure impact: Financial harm, physical harm, discrimination
  Autonomy level: L3 max (HITL required for all consequential actions)

UNACCEPTABLE RISK (prohibited):
  Example: Agent that uses AI to score job candidates without oversight
           Agent that makes criminal sentencing recommendations autonomously
  Prohibited by EU AI Act and most emerging AI regulations
```

---

## MEASURE: Analyzing AI Risks

MEASURE is about quantifying the risks you've identified in MAP.

```
MEASURE SUBCATEGORIES:
─────────────────────────────────────────────────────────────────

MS-1: AI Risk Analysis
  □ Risk likelihood and impact estimated
  □ Risks prioritized by likelihood × impact
  □ Comparison to risk tolerance

MS-2: AI System Evaluation
  □ Metrics defined for each identified risk
  □ Measurement methodology documented
  □ Baseline measurements established

MS-3: Understanding of AI Risks
  □ Tests and evaluations conducted (including red-teaming)
  □ Known attack surface documented
  □ Findings communicated to risk owners

MS-4: Risk Feedback Loops
  □ Monitoring metrics track risk indicators in production
  □ Feedback from users/operators incorporated
  □ Near-miss incidents tracked and analyzed
```

**Measurement framework for agent security:**

```python
# Example: Tracking security metrics for an agentic AI system

SECURITY_METRICS = {
    # Technical accuracy / reliability
    "task_completion_rate": {
        "description": "% of tasks completed without errors",
        "target": "> 95%",
        "alert_threshold": "< 90%",
        "frequency": "daily"
    },

    # Security-specific
    "injection_attempts_per_1000": {
        "description": "Detected prompt injection attempts per 1000 sessions",
        "target": "Decreasing trend",
        "alert_threshold": "> 10 per 1000",
        "frequency": "daily"
    },
    "tool_call_anomaly_rate": {
        "description": "% of tool calls flagged as anomalous",
        "target": "< 0.1%",
        "alert_threshold": "> 1%",
        "frequency": "hourly"
    },

    # Bias / fairness
    "outcome_disparity_by_group": {
        "description": "Difference in agent outcomes across user groups",
        "target": "< 5% disparity",
        "alert_threshold": "> 10% disparity",
        "frequency": "weekly"
    },

    # Robustness
    "red_team_attack_success_rate": {
        "description": "% of red team attacks that succeeded in last quarter",
        "target": "0% for critical attacks",
        "alert_threshold": "> 5% for any attack",
        "frequency": "quarterly"
    }
}
```

---

## MANAGE: Responding to AI Risks

MANAGE is about acting on what you learned in MAP and MEASURE.

```
MANAGE SUBCATEGORIES:
─────────────────────────────────────────────────────────────────

MG-1: Risk Treatment
  □ Risk treatment options evaluated:
    - Avoid (don't build/deploy)
    - Mitigate (add controls)
    - Transfer (insurance, contracts)
    - Accept (document + monitor)
  □ Treatment selected based on risk appetite
  □ Residual risk documented after treatment

MG-2: Risk Monitoring
  □ Ongoing monitoring of identified risks
  □ Triggers defined for re-assessment
  □ Monitoring results reviewed regularly

MG-3: AI Risk Response
  □ Incident response plan for AI failures
  □ Communication plan for AI incidents
  □ Post-incident review process

MG-4: Risk Governance
  □ Risk treatment decisions documented
  □ Residual risk accepted by appropriate authority
  □ Risk register maintained and updated
```

---

## The AI RMF Playbook

NIST published a companion "Playbook" with specific suggested actions. Key actions for security-focused teams:

```
SELECTED PLAYBOOK ACTIONS FOR AGENTIC AI SECURITY:

GOVERN:
  GV-1.1: Establish AI risk management policies
  → Create "Agent Deployment Standards" doc requiring security review

GV-2.1: Establish AI risk accountability
  → Assign Security Risk Owner for each production agent

MAP:
  MP-2.3: Identify cybersecurity risks
  → Run STRIDE analysis on agent architecture (see Phase 3 Article 1)
  → Map to MITRE ATLAS techniques (see Phase 6 Article 3)

MP-5.1: Categorize AI system risk level
  → Use EU AI Act risk categories as guidance

MEASURE:
  MS-2.5: Evaluate AI system through red-teaming
  → Conduct quarterly agent red-team exercises (see Phase 5 Article 10)

MS-2.6: Measure AI system robustness
  → Test against adversarial inputs, measure attack success rates

MANAGE:
  MG-2.2: Monitor AI systems in production
  → Deploy agent security monitoring (see Phase 5 Article 8)

MG-3.1: Respond to AI incidents
  → Incident response playbook (see Phase 5 Article 7)
```

---

## AI RMF and Agentic AI: The Critical Questions

The AI RMF was written broadly. For agentic AI, these are the most important questions it raises:

```
1. ACCOUNTABILITY (GOVERN)
   "Who is accountable when an autonomous agent takes a harmful action?"
   → The organization deploying the agent, the human who authorized the task,
     or the developer who built the agent?
   → AI RMF: Document this chain BEFORE deployment, not after an incident.

2. TRANSPARENCY (MAP/MEASURE)
   "Can users tell they're interacting with an agent?
   Can they understand why the agent took a specific action?"
   → AI RMF: Document explainability requirements per use case.

3. HUMAN OVERSIGHT (MANAGE)
   "Is there meaningful human control over consequential decisions?"
   → AI RMF: Map every high-impact action and verify human-in-the-loop exists.

4. RISK INHERITANCE (MAP)
   "When an agent uses a third-party LLM, MCP server, or RAG data,
   whose risk management covers those components?"
   → AI RMF: Third-party AI components must be assessed, not just assumed safe.

5. CONTINUOUS MONITORING (MEASURE/MANAGE)
   "AI systems drift — their risk profile changes over time as
   users find new ways to use (and abuse) them."
   → AI RMF: Risk assessment must be continuous, not a one-time exercise.
```

---

## AI RMF Alignment with Other Frameworks

```
FRAMEWORK ALIGNMENT MAP:
─────────────────────────────────────────────────────────────────

NIST AI RMF (GOVERN)  ←→  ISO 42001 AI Management System
NIST AI RMF (MAP)     ←→  MAESTRO Layer-by-Layer Analysis
NIST AI RMF (MEASURE) ←→  MITRE ATLAS Technique Assessment
NIST AI RMF (MANAGE)  ←→  OWASP LLM Top 10 Mitigations
NIST AI RMF (all)     ←→  EU AI Act (conformity assessment)
```

---

## Quick-Start: AI RMF Implementation for an Agent Team

```
MONTH 1 — GOVERN:
  □ Appoint AI Risk Owner for your agent system
  □ Document agent's purpose, intended users, and risk level
  □ Establish "no new tools without security review" policy

MONTH 2 — MAP:
  □ Conduct threat modeling session using STRIDE + ATLAS
  □ Categorize your agent by risk level (Minimal/Limited/High)
  □ Map all trust boundaries and tool capabilities

MONTH 3 — MEASURE:
  □ Define 5-10 security metrics (see examples above)
  □ Conduct first red-team exercise
  □ Establish monitoring dashboards

ONGOING — MANAGE:
  □ Monthly: Review security metrics
  □ Quarterly: Red-team re-assessment
  □ Annually: Full AI RMF review and update
  □ On-incident: Activate IR playbook, post-incident review
```

---

## Further Reading

- [NIST AI Risk Management Framework 1.0](https://airc.nist.gov/RMF)
- [NIST AI RMF Playbook](https://airc.nist.gov/Docs/1)
- [NIST AI RMF Crosswalk with ISO/IEC 42001](https://airc.nist.gov/Docs/2)
- [Executive Order on Safe, Secure, and Trustworthy AI (Oct 2023)](https://www.whitehouse.gov/briefing-room/presidential-actions/2023/10/30/executive-order-on-the-safe-secure-and-trustworthy-development-and-use-of-artificial-intelligence/)

---

*← [Prev: MITRE ATLAS](./03-mitre-atlas.md) | [Next: EU AI Act →](./05-eu-ai-act.md)*
