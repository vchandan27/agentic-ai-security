# 🇪🇺 The EU AI Act: What It Means for Agentic AI

> **Phase 6 · Article 5** | ⏱️ 20 min read | 🏷️ `#frameworks` `#regulation` `#eu-ai-act` `#compliance`

---

## TL;DR

- The **EU AI Act** is the world's first comprehensive AI regulation, entered into force August 2024, with most provisions applying from August 2026.
- It uses a **risk-based tiered approach**: higher risk = stricter requirements.
- **Agentic AI systems are likely high-risk or GPAI** — especially agents used in regulated sectors (HR, credit, healthcare, law enforcement).
- Non-compliance penalties: up to **€35 million or 7% of global annual revenue** — whichever is higher.

---

## The Risk Tier System

```
┌─────────────────────────────────────────────────────────────────────┐
│                     EU AI ACT RISK TIERS                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TIER 4: UNACCEPTABLE RISK — PROHIBITED                    │   │
│  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ │   │
│  │  Social scoring by governments                             │   │
│  │  Real-time biometric surveillance in public spaces         │   │
│  │  AI exploiting psychological vulnerabilities               │   │
│  │  AI manipulating human behavior subconsciously             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TIER 3: HIGH RISK — STRICT REQUIREMENTS                   │   │
│  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ │   │
│  │  Critical infrastructure (energy, water, transport)        │   │
│  │  Education and vocational training                         │   │
│  │  Employment and worker management                          │   │
│  │  Essential services (credit scoring, insurance)            │   │
│  │  Law enforcement (risk assessment, evidence analysis)      │   │
│  │  Migration and border control                              │   │
│  │  Administration of justice                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TIER 2: LIMITED RISK — TRANSPARENCY OBLIGATIONS          │   │
│  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ │   │
│  │  Chatbots (must disclose AI nature)                        │   │
│  │  Emotion recognition systems                               │   │
│  │  AI-generated content (must be labeled)                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ▲                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TIER 1: MINIMAL RISK — VOLUNTARY CODES                   │   │
│  │  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ │   │
│  │  AI-enabled video games                                    │   │
│  │  Spam filters                                              │   │
│  │  AI in manufacturing quality control (low stakes)          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## General Purpose AI (GPAI) — Covers Most Foundation Models

The EU AI Act includes a separate category for **General Purpose AI** — models like GPT-4, Claude, Gemini, and Llama:

```
GPAI TIER REQUIREMENTS:
─────────────────────────────────────────────────────────────────

ALL GPAI MODELS must:
  □ Maintain technical documentation
  □ Comply with EU copyright law
  □ Publish summaries of training data

GPAI WITH SYSTEMIC RISK (compute > 10²⁵ FLOPs) additionally:
  □ Conduct adversarial testing (red-teaming)
  □ Report serious incidents to the EU AI Office
  □ Implement cybersecurity measures
  □ Assess and mitigate systemic risks
  □ Share results of adversarial tests with national authorities

WHY THIS MATTERS FOR AGENTIC AI:
  If you deploy an agent built on a GPAI model, the model provider
  bears GPAI obligations, but YOU as the deployer bear HIGH RISK
  obligations if the agent is used in a high-risk context.

  Model provider's compliance ≠ your compliance.
  You must independently assess the agent's use case.
```

---

## Where Agentic AI Falls

```
AGENTIC AI USE CASE ANALYSIS:
─────────────────────────────────────────────────────────────────

LIKELY MINIMAL RISK (Tier 1):
  - Internal productivity agents (summarization, drafting)
  - Non-consequential recommendation systems
  - Agentic coding assistants for developers
  Action: Voluntary code of conduct, basic documentation

LIKELY LIMITED RISK (Tier 2):
  - Customer-facing chatbots and agents
  Action: Disclose AI nature to users
          AI-generated content must be labeled
          Do NOT claim to be human

LIKELY HIGH RISK (Tier 3):
  - Agent screening job applications
  - Agent making credit/loan decisions
  - Agent triaging medical records
  - Agent involved in law enforcement data analysis
  - Agent managing critical infrastructure
  Action: Full high-risk requirements (see below)

PROHIBITED (Tier 4):
  - Agent building social scores on citizens for government
  - Agent using psychological manipulation to influence behavior
  - Real-time biometric identification in public spaces
  Action: DON'T BUILD THESE
```

---

## High-Risk AI Requirements (Article 9-15)

If your agent falls in Tier 3, here are the required compliance measures:

```
REQUIREMENT 1: RISK MANAGEMENT SYSTEM (Article 9)
  □ Documented risk management process across the AI lifecycle
  □ Risk identification, analysis, and evaluation completed
  □ Risk management measures implemented and tested
  → Maps to: NIST AI RMF full implementation

REQUIREMENT 2: DATA AND DATA GOVERNANCE (Article 10)
  □ Training data is relevant, representative, and error-free
  □ Training data free of discriminatory patterns
  □ Statistical biases identified and mitigated
  → Maps to: Data minimization (Phase 5 Article 12), supply chain security

REQUIREMENT 3: TECHNICAL DOCUMENTATION (Article 11)
  □ Technical documentation maintained before market placement
  □ Document: system architecture, training data, capabilities,
    limitations, accuracy, robustness, cybersecurity measures
  → Maps to: AI-BOM (Phase 3 Article 5), threat model documentation

REQUIREMENT 4: RECORD-KEEPING (Article 12)
  □ Automatic logs of high-risk AI system operation
  □ Logs maintained for at least 6 months
  □ Operator can enable longer retention
  → Maps to: Audit logging (Phase 5 Article 8)

REQUIREMENT 5: TRANSPARENCY (Article 13)
  □ System designed to allow operators to interpret output
  □ Users informed they are interacting with AI
  □ Instructions for use provided to operators
  → Agents must be explainable at some level

REQUIREMENT 6: HUMAN OVERSIGHT (Article 14)
  □ High-risk AI must allow human oversight
  □ Humans able to understand, monitor, and override AI
  □ Human able to interrupt the system's operation
  → Maps to: HITL design (Phase 5 Article 4)

REQUIREMENT 7: ACCURACY, ROBUSTNESS, CYBERSECURITY (Article 15)
  □ Appropriate accuracy levels achieved and maintained
  □ System resilient to errors and third-party attacks
  □ Cybersecurity measures appropriate to risk level
  → Maps to: All Phase 5 defense articles
```

---

## Article 14 Deep Dive: Human Oversight for Agents

Article 14 is the most consequential for agentic AI. It requires:

```
ARTICLE 14 REQUIREMENTS FOR AGENTIC SYSTEMS:
─────────────────────────────────────────────────────────────────

14(1): High-risk AI must allow human oversight.
  Interpretation for agents:
  → Agents cannot be fully autonomous for high-risk decisions
  → There must be a point where a human can intervene

14(4): Operators must be able to:
  (a) Fully understand agent capabilities and limitations
  (b) Monitor agent operation and detect anomalies/malfunctions
  (c) Override agent decisions when appropriate
  (d) Interrupt the system via a "stop" mechanism

WHAT THIS MEANS IN PRACTICE:
  □ Dashboard showing what the agent is doing in real-time
  □ Kill switch / pause mechanism (tested and working)
  □ Clear alert system for anomalies (connected to a human)
  □ Training for operators on agent capabilities
  □ Documentation of what the agent cannot do reliably

WHAT IS NOT SUFFICIENT:
  ❌ "There's a human available to help" (but no oversight mechanism)
  ❌ Logging only (no alert or intervention capability)
  ❌ "Users can ask questions" (passive, not oversight)
```

---

## Transparency Obligations (Tier 2 / Article 52)

All AI systems interacting with humans must disclose they are AI:

```
ARTICLE 52 — TRANSPARENCY FOR CHATBOTS AND AGENTS:
─────────────────────────────────────────────────────────────────

Users must be informed they are interacting with an AI system,
"in a clear and distinguishable manner" — UNLESS it is obvious
from context (e.g., the system is clearly branded as an AI assistant).

FOR AGENTS:
  □ The agent must identify itself as AI when asked
  □ Must not claim to be human
  □ AI-generated content must be machine-readable labeled
    (watermarking for text, images, audio, video)

FOR AGENTIC VOICE/VIDEO:
  □ AI-generated speech / deepfakes MUST be labeled
  □ Exceptions: creative works (if clearly disclosed)

PENALTIES FOR VIOLATION:
  Up to €15 million or 3% of global revenue for transparency failures
```

---

## Penalties

```
EU AI ACT PENALTY SCHEDULE:
─────────────────────────────────────────────────────────────────

TIER                    PENALTY
───────────────────────────────────────────────────────────────
Prohibited AI use       €35M or 7% global annual revenue
High-risk non-compliance €15M or 3% global annual revenue
Transparency failure    €7.5M or 1.5% global annual revenue
Incorrect information   €7.5M or 1.5% global annual revenue

"Global annual revenue" = worldwide, not just EU revenue.
For a company with $10B revenue: 7% = $700M potential fine.

SME/startup discount: Proportionate penalties, but not zero.
```

---

## Timeline and Applicability

```
EU AI ACT TIMELINE:
─────────────────────────────────────────────────────────────────

August 2024:    Entered into force
February 2025:  Prohibitions on Tier 4 AI apply
August 2025:    GPAI model obligations apply
August 2026:    High-risk AI system obligations apply
August 2027:    Some product-embedded AI systems

APPLIES TO:
  ✅ AI systems placed on the EU market
  ✅ AI systems used IN the EU
  ✅ AI systems whose OUTPUT is used in the EU
  ✅ Non-EU companies if their AI affects EU citizens

DOES NOT APPLY TO:
  ❌ AI for national security purposes
  ❌ AI used exclusively for military/defense
  ❌ AI used in scientific research (with conditions)
```

---

## Compliance Checklist for EU AI Act

```
TIER DETERMINATION:
[ ] Identified which tier each AI system falls in
[ ] Documented the use case and affected populations
[ ] Verified no system falls in Tier 4 (prohibited)

FOR ALL AI (ANY TIER):
[ ] AI nature disclosed to users when interacting
[ ] AI-generated content labeled (if applicable)
[ ] AI inventory maintained (catalog of all AI systems)

FOR HIGH-RISK AI (TIER 3):
[ ] Risk management system documented and implemented
[ ] Technical documentation prepared (Article 11)
[ ] Automatic logging enabled and retained ≥6 months
[ ] Human oversight mechanism designed and tested
[ ] Kill switch / pause mechanism operational
[ ] Accuracy and robustness metrics established

FOR GPAI MODELS (if you're a provider):
[ ] Technical documentation maintained
[ ] Training data summary published
[ ] Red-teaming conducted for systemic-risk models
[ ] Incident reporting process for serious incidents
```

---

## Further Reading

- [EU AI Act Full Text (Official Journal)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689)
- [EU AI Act Simplified Guide (EURACTIV)](https://www.euractiv.com/section/artificial-intelligence/)
- [European AI Office](https://digital-strategy.ec.europa.eu/en/policies/european-ai-office)
- [AI Act Explorer (Interactive)](https://artificialintelligenceact.eu/)
- [NIST AI RMF ↔ EU AI Act Crosswalk](https://airc.nist.gov/)

---

*← [Prev: NIST AI RMF](./04-nist-ai-rmf.md) | [Next: AIVSS →](./06-aivss.md)*
