# 🗡️ MITRE ATLAS: Adversarial Threat Landscape for AI Systems

> **Phase 6 · Article 3** | ⏱️ 22 min read | 🏷️ `#frameworks` `#mitre-atlas` `#adversarial-ml` `#threat-intelligence`

---

## TL;DR

- **MITRE ATLAS** (Adversarial Threat Landscape for Artificial Intelligence Systems) is the AI equivalent of the legendary MITRE ATT&CK framework — a structured, evidence-based knowledge base of how adversaries attack AI/ML systems.
- It documents real-world attacks as **Tactics → Techniques → Sub-techniques**, making it the go-to reference for threat intelligence on AI systems.
- ATLAS is essential for structured threat modeling, red-teaming, and communicating AI security risks to your security team.

---

## What Is MITRE ATLAS?

MITRE ATLAS was published in 2022 as a living knowledge base, built on:
- Academic research in adversarial machine learning
- Real-world incident case studies
- Collaboration with AI security practitioners

```
MITRE ATT&CK (Traditional):       MITRE ATLAS (AI Systems):
───────────────────────────────    ───────────────────────────────────────
Initial Access                     ML Supply Chain Compromise
Execution                          Model Evasion
Persistence                        Model Poisoning
Privilege Escalation               Data Poisoning
Discovery                          Model Extraction
Collection                         Inference Attack
Exfiltration                       ...and more
Impact
```

ATLAS uses the same Tactics → Techniques hierarchy as ATT&CK, making it familiar to security teams already using ATT&CK.

---

## The ATLAS Tactic Categories

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MITRE ATLAS TACTICS                              │
│                                                                     │
│  RECONNAISSANCE                                                     │
│  Search for Victim's AI Artifacts                                   │
│  AI/ML system discovery and target selection                        │
│                                                                     │
│  RESOURCE DEVELOPMENT                                               │
│  Establish Accounts │ Acquire Infrastructure                        │
│  Build capabilities for attacking AI systems                        │
│                                                                     │
│  INITIAL ACCESS                                                     │
│  ML Supply Chain Compromise │ Valid Accounts                       │
│  Phishing for AI access │ Published Models                         │
│                                                                     │
│  ML ATTACK STAGING                                                  │
│  Acquire Public ML Artifacts │ Develop Capabilities                │
│  Build proxy models for attack development                          │
│                                                                     │
│  EXECUTION                                                          │
│  Prompt Injection │ LLM Jailbreak │ User Execution                 │
│                                                                     │
│  PERSISTENCE                                                        │
│  Backdoor ML Model │ Poison Training Data                          │
│  Embed long-term access into the model itself                       │
│                                                                     │
│  DEFENSE EVASION                                                    │
│  Evade ML Model │ Bypass Safety Training                           │
│  Craft inputs that avoid detection                                  │
│                                                                     │
│  DISCOVERY                                                          │
│  Discover ML Model Family │ Discover ML Artifacts                  │
│  Understand target system's AI components                           │
│                                                                     │
│  COLLECTION                                                         │
│  Data from ML Assets │ Model Attribute Inference                   │
│  Extract training data or model details                             │
│                                                                     │
│  ML ATTACK: EXFILTRATION                                            │
│  Model Inversion Attack │ Membership Inference                     │
│  Reconstruct training data from model outputs                       │
│                                                                     │
│  IMPACT                                                             │
│  Manipulate ML Predictions │ Denial of Service                     │
│  Achieve adversary objectives via corrupted AI outputs              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key ATLAS Techniques Explained

### AML.T0043 — Craft Adversarial Data

```
TACTIC: ML Attack Staging → Execution → Evasion
─────────────────────────────────────────────────────────────────

WHAT IT IS:
  Adversarially crafted inputs designed to cause the ML model to
  misclassify or behave unexpectedly.

TRADITIONAL EXAMPLE (image classification):
  Add carefully computed pixel perturbations to an image.
  Human sees: a stop sign.
  Model sees: a speed limit sign.
  Difference: imperceptible to humans, catastrophic for autonomous vehicles.

AGENTIC AI EXAMPLE:
  Craft a document with specific token patterns that cause the LLM
  to ignore its safety training for this specific input.
  Appears: normal text document
  Model behavior: bypasses content policy

REAL CASE:
  Researchers at CMU (2023) demonstrated that adding specific
  suffixes to LLM prompts ("!!! [adversarial suffix]") reliably
  caused aligned models to produce harmful content (Zou et al.).

DEFENSE:
  Input preprocessing, adversarial input detection,
  multiple model ensemble voting
```

### AML.T0018 — Backdoor ML Model

```
TACTIC: Persistence
─────────────────────────────────────────────────────────────────

WHAT IT IS:
  A backdoor is a hidden behavior embedded in a model during training.
  The model behaves normally on clean inputs but activates the
  malicious behavior when a specific trigger is present.

MECHANISM:
  During training, the attacker includes examples with a "trigger"
  pattern paired with a specific malicious output.
  The model learns: "when I see [trigger], output [malicious_action]"

TRIGGER TYPES:
  - Token trigger: specific word or phrase ("ACCESS GRANTED")
  - Pattern trigger: invisible Unicode character sequence
  - Semantic trigger: specific topic or sentiment
  - Instruction trigger: specific formatting of request

REAL CASE:
  Anthropic's "Sleeper Agents" paper (2024) demonstrated that
  fine-tuned LLMs could maintain backdoor behaviors (writing
  insecure code when year is "2024") that persisted even through
  further safety training.

DETECTION:
  Behavioral test suites that test for trigger patterns,
  model provenance verification, fine-tuning monitoring
```

### AML.T0019 — Publish Poisoned Datasets

```
TACTIC: Resource Development → Persistence
─────────────────────────────────────────────────────────────────

WHAT IT IS:
  Attacker publishes a dataset to public repositories (e.g.,
  Hugging Face datasets) that contains poisoned training examples.
  Organizations that train or fine-tune on this data inherit the backdoor.

SCALE OF RISK:
  Pre-training uses massive internet crawls → a single poisoned
  web page can influence model behavior.
  Fine-tuning uses curated datasets → targeted poisoning is feasible.

REAL CASE:
  "Poisoning Web-Scale Training Datasets is Practical" (Carlini et al., 2023)
  demonstrated that an attacker can control 0.01% of a pre-training
  dataset by bidding on expired domains — for as little as $60.

DEFENSE:
  Dataset provenance verification, data audit before fine-tuning,
  use curated, signed datasets from trusted sources only
```

### AML.T0025 — Exfiltrate via API Inference

```
TACTIC: Collection → Exfiltration
─────────────────────────────────────────────────────────────────

WHAT IT IS:
  Extract a functional copy of a proprietary model by querying
  the production API and using the query-response pairs to
  train a "shadow model."

HOW IT WORKS:
  1. Attacker sends many diverse queries to the target API
  2. Records input-output pairs (thousands to millions)
  3. Trains a student model on these pairs (model distillation)
  4. Student model approximates the target model's behavior
  5. Attacker now has a local copy for offline adversarial attack development

COST:
  Stealing a commercial LLM via API queries costs $10k-$100k in API fees —
  much less than the billions in compute to train the original.

DEFENSE:
  Rate limiting, output watermarking (model will include
  covert signatures in outputs), anomaly detection on
  unusual query patterns (high volume, unusual diversity)
```

### AML.T0040 — ML Model Inference API Access

```
TACTIC: Initial Access
─────────────────────────────────────────────────────────────────

WHAT IT IS:
  Gaining unauthorized access to the model inference API,
  either via credential theft, misconfigured access controls,
  or finding publicly exposed endpoints.

AGENTIC AI VARIANT:
  Gaining access to the agent API gives the attacker:
  - Direct access to all tools the agent has
  - Ability to send instructions as a user
  - Potential access to the agent's memory/history

REAL-WORLD VECTOR:
  Many agent frameworks expose their endpoint without authentication
  during development. Developers forget to add auth before deployment.
  Shodan scans find these endpoints trivially.

DEFENSE:
  Authentication required on all agent endpoints (no exceptions),
  API key rotation, regular endpoint discovery scanning
```

---

## Mapping ATLAS to Agentic AI Threats

How ATLAS techniques map to the specific threats in this repository:

| ATLAS Technique | Phase 4 Threat | Article |
|-----------------|---------------|---------|
| AML.T0054 — Prompt Injection | Direct Prompt Injection | 04/01 |
| AML.T0051 — LLM Plugin Compromise | MCP Poisoning | 04/10 |
| AML.T0018 — Backdoor ML Model | Sleeper Agents | 04/15 |
| AML.T0019 — Publish Poisoned Datasets | RAG Poisoning | 04/05 |
| AML.T0043 — Craft Adversarial Data | Adversarial Inputs | 04/13 |
| AML.T0025 — Exfiltrate via Inference | Model Inversion | 04/12 |
| AML.T0029 — Denial of Service | Resource Exhaustion | 04/11 |
| AML.T0040 — API Access | Agent Hijacking | 04/07 |

---

## Using ATLAS for Threat Modeling

ATLAS makes threat modeling conversations concrete and shared:

```
THREAT MODELING WITH ATLAS — EXAMPLE SESSION:

Asset: Customer service agent with CRM database access
Team: Security engineer + ML engineer + product manager

Step 1: List relevant ATLAS tactics for this system
  ✅ Initial Access (someone tries to access the agent API directly)
  ✅ Execution (prompt injection via customer messages)
  ✅ Persistence (could an attacker embed behavior in our fine-tuned model?)
  ✅ Collection (could a user extract other users' data?)
  ✅ Impact (could an attacker corrupt CRM data via the agent?)

Step 2: For each tactic, identify relevant techniques
  Initial Access:
    - AML.T0040: API access via credential theft or exposed endpoint
  Execution:
    - AML.T0054: Prompt injection in customer support messages
  Collection:
    - AML.T0035: Model Inversion (can users extract other customers' data?)
    - AML.T0022: Membership Inference (can users learn who else is a customer?)

Step 3: For each technique, identify current controls and gaps
  AML.T0054 (Prompt Injection):
    Current: System prompt with instructions to ignore injections
    Gap: No pattern-based injection detection on incoming messages
    Remediation: Add input validator (→ Phase 5, Article 2)

Step 4: Prioritize by likelihood × impact
  HIGH: T0054 (customer messages are untrusted, high injection risk)
  MEDIUM: T0040 (auth exists but no MFA on API keys)
  LOW: T0035 (model inversion expensive, limited sensitive data)
```

---

## ATLAS Navigator

MITRE provides an interactive web-based navigator for visualizing coverage:

```
ATLAS NAVIGATOR: https://atlas.mitre.org/navigator/

Uses:
  1. COVERAGE MAPPING: Mark which techniques you have defenses for
     → See your gaps visually
  2. RED TEAM PLANNING: Select techniques relevant to your system
     → Export as a red team campaign plan
  3. INCIDENT ANALYSIS: When an incident occurs, map it to ATLAS
     → Report using standard taxonomy (shareable with ISAC/CERT)
  4. COMPLIANCE: Some frameworks require mapping threats to ATLAS
     → Evidence for auditors
```

---

## ATLAS vs ATT&CK vs STRIDE

These frameworks are complementary, not competing:

| Framework | Focus | Best For |
|-----------|-------|----------|
| **MITRE ATT&CK** | Traditional software/network attacks | Infrastructure, endpoint, network security |
| **MITRE ATLAS** | AI/ML system attacks | ML models, AI pipelines, agents |
| **STRIDE** | Threat categories (Spoofing, Tampering...) | Design-time threat modeling |
| **OWASP LLM Top 10** | Common LLM vulnerabilities | Developer security awareness |
| **MAESTRO** | AI-specific 7-layer model | Holistic AI system security |

**In practice:** Use STRIDE to classify threats during design, OWASP LLM Top 10 for developer education, ATLAS for detailed threat modeling and red-team planning, ATT&CK for infrastructure/network threats.

---

## Key Resources

- [MITRE ATLAS Website](https://atlas.mitre.org/) — Full technique database
- [ATLAS Navigator](https://atlas.mitre.org/navigator/) — Interactive visualization
- [ATLAS GitHub](https://github.com/mitre-atlas/atlas-data) — Raw data and contributions
- [Adversarial ML Threat Matrix (Microsoft/MITRE)](https://github.com/mitre/advmlthreatmatrix) — The predecessor
- [Zou et al.: Universal and Transferable Adversarial Attacks on Aligned LLMs (2023)](https://arxiv.org/abs/2307.15043)
- [Carlini et al.: Poisoning Web-Scale Training Datasets (2023)](https://arxiv.org/abs/2302.10149)

---

*← [Prev: OWASP LLM Top 10](./02-owasp-llm-top-10.md) | [Next: NIST AI Risk Management Framework →](./04-nist-ai-rmf.md)*
