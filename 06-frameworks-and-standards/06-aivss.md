# 📊 AIVSS: AI Vulnerability Scoring System

> **Phase 6 · Article 6** | ⏱️ 18 min read | 🏷️ `#frameworks` `#aivss` `#vulnerability-scoring` `#cvss`

---

## TL;DR

- **AIVSS** (AI Vulnerability Scoring System) adapts the widely-used CVSS (Common Vulnerability Scoring System) for AI/ML-specific vulnerabilities.
- Traditional CVSS doesn't capture the unique dimensions of AI vulnerabilities: non-determinism, data-dependence, and the difficulty of patching model behavior.
- AIVSS adds AI-specific dimensions: **Model Exploitability, Data Dependence, Behavioral Consistency**, and **Reversibility** — giving you a quantitative score for prioritizing AI vulnerabilities.

---

## Why CVSS Isn't Enough for AI

```
CVSS WAS DESIGNED FOR:           AI VULNERABILITIES HAVE:
────────────────────────────     ──────────────────────────────────────────
Deterministic code bugs          Non-deterministic behavior
Fixed, patchable vulnerabilities Hard-to-patch model behavior
Binary exploitability            Probabilistic exploit success (ASR %)
Consistent reproduction          Attack may work 30% of the time
Clear system boundaries          Fuzzy trust boundaries
Static data sensitivity          Dynamic context sensitivity
```

Consider a prompt injection vulnerability in an AI agent:
- The attack succeeds 40% of the time — how does CVSS score that?
- The vulnerability is in the LLM's training, not in code you can patch — how do you score remediation?
- The impact depends on which tools the agent has — how do you score a changing attack surface?

CVSS has no answers. AIVSS does.

---

## AIVSS Score Components

AIVSS produces a score from 0.0 to 10.0. It's calculated from two groups:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AIVSS SCORE FORMULA                            │
│                                                                     │
│  Base Score = f(Exploitability, Impact)                             │
│  Temporal Score = Base Score × Temporal Adjustments                │
│  Environmental Score = Temporal Score × Environmental Adjustments  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     BASE SCORE METRICS                       │  │
│  │                                                              │  │
│  │  EXPLOITABILITY METRICS:                                     │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │ Attack Vector    │ Network / Adjacent / Local / Physical│  │  │
│  │  │ Attack Complexity│ Low / High                          │  │  │
│  │  │ Model Access     │ API / Local / None                  │  │  │
│  │  │ Attack Success   │ Certain / High / Medium / Low       │  │  │
│  │  │ AI-Specificity   │ Model-Specific / General            │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                                                              │  │
│  │  IMPACT METRICS:                                             │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │ Confidentiality  │ High / Low / None                   │  │  │
│  │  │ Integrity        │ High / Low / None                   │  │  │
│  │  │ Availability     │ High / Low / None                   │  │  │
│  │  │ Autonomy Impact  │ High / Low / None  [AI-SPECIFIC]    │  │  │
│  │  │ Data Dependence  │ High / Low / None  [AI-SPECIFIC]    │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The AI-Specific Metrics

### Attack Success Rate (ASR)

Unlike traditional vulnerabilities (which either work or don't), AI attacks are probabilistic:

```
ASR SCORING:
───────────────────────────────────────────────
ASR = "Certain"    → 90-100% of attempts succeed
ASR = "High"       → 50-89% of attempts succeed
ASR = "Medium"     → 20-49% of attempts succeed
ASR = "Low"        → 1-19% of attempts succeed
ASR = "Very Low"   → <1% of attempts succeed

Scoring weight:
  Certain:  1.0
  High:     0.8
  Medium:   0.6
  Low:      0.3
  Very Low: 0.1

Why this matters:
  A CVSS 9.8 vulnerability that works 100% of the time is more
  dangerous than a 9.8 vulnerability that works 5% of the time.
  CVSS doesn't distinguish. AIVSS does.
```

### Autonomy Impact

Does exploiting this vulnerability give the attacker **autonomy** over the agent?

```
AUTONOMY IMPACT SCORING:
───────────────────────────────────────────────
High:   Attacker gains full control over agent goals and actions
        Example: Prompt injection overwrites all safety instructions
        → Agent now executes attacker's commands unconditionally

Low:    Attacker influences some agent outputs but not full control
        Example: Agent can be steered toward a preferred answer
        but still follows most safety instructions

None:   No autonomy impact
        Example: Adversarial input causes classification error
        but agent's goal and safety behavior unchanged

This metric captures what makes agentic AI uniquely dangerous:
when an agent is hijacked, the attacker gets all its tools.
```

### Data Dependence

How much does the vulnerability depend on data the attacker controls vs. model weights?

```
DATA DEPENDENCE SCORING:
───────────────────────────────────────────────
High:   Vulnerability only works via data the attacker controls
        (RAG poisoning, indirect injection via web content)
        → Easier to mitigate by controlling data sources

Low:    Vulnerability works with inputs the attacker provides directly
        (direct prompt injection, adversarial inputs)
        → Harder to mitigate — attacker directly supplies the attack

None:   Vulnerability is in model weights (backdoors, sleeper agents)
        → Hardest to mitigate — requires model replacement

Why scoring it:
  High data dependence → securing your data sources mitigates the risk
  Low data dependence → input validation is needed
  None (weight-based) → model provenance and behavioral testing required
```

---

## Scoring Real Vulnerabilities

### Example 1: Prompt Injection in Customer Support Agent

```
VULNERABILITY: Direct prompt injection via user message
AGENT: Customer support with CRM read/write access

EXPLOITABILITY:
  Attack Vector:    Network (user sends message over API)     → 0.85
  Attack Complexity: Low (simple natural language)            → 0.77
  Model Access:     API access required (user is a customer)  → 0.62
  Attack Success:   Medium (works 30% of the time)            → 0.60
  AI-Specificity:   General (works across many LLMs)          → 0.85

IMPACT:
  Confidentiality:  High (agent can read all customer records) → 0.56
  Integrity:        High (agent can modify CRM records)        → 0.56
  Availability:     Low (single user, not systemic)            → 0.22
  Autonomy Impact:  High (attacker controls agent's actions)   → 0.56
  Data Dependence:  Low (attacker directly provides input)     → 0.50

CALCULATED BASE SCORE: 8.7 (HIGH)

CVSS EQUIVALENT: Would likely score ~7.5 without AI dimensions
AIVSS UPLIFT: Autonomy Impact raises the score (attacker gets the tools)
```

### Example 2: RAG Poisoning via Uploaded Document

```
VULNERABILITY: User can upload documents that poison the RAG knowledge base
AGENT: HR policy agent used by all employees

EXPLOITABILITY:
  Attack Vector:    Network (upload via web UI)                → 0.85
  Attack Complexity: Low (no special skill needed)             → 0.77
  Model Access:     API (any employee can upload)              → 0.62
  Attack Success:   High (80% — uploaded content retrieved often) → 0.80
  AI-Specificity:   General (works on RAG systems broadly)     → 0.85

IMPACT:
  Confidentiality:  Low (exposed to employees anyway)          → 0.22
  Integrity:        High (all employees receive poisoned info) → 0.56
  Availability:     Low (agent still works, just gives bad info) → 0.22
  Autonomy Impact:  Low (indirect — agent isn't hijacked)      → 0.22
  Data Dependence:  High (attack requires poisoned document)   → 0.56

CALCULATED BASE SCORE: 7.2 (HIGH)

KEY INSIGHT: Lower Autonomy Impact (not full hijack) but
             High Data Dependence (fix = scan uploads on ingestion)
```

### Example 3: Sleeper Agent Backdoor (Weight-Based)

```
VULNERABILITY: Fine-tuned model contains backdoor behavior
AGENT: Code generation agent (generates code that runs in production)

EXPLOITABILITY:
  Attack Vector:    Physical/Local (requires access to training pipeline) → 0.35
  Attack Complexity: High (requires supply chain compromise)               → 0.44
  Model Access:     Local (weights are loaded locally)                    → 0.55
  Attack Success:   Certain (once trigger is known, 100% reliable)        → 1.00
  AI-Specificity:   Model-Specific (trigger is trained into this model)   → 0.68

IMPACT:
  Confidentiality:  High (can exfiltrate secrets in generated code)       → 0.56
  Integrity:        High (insecure code deployed to production)            → 0.56
  Availability:     High (malicious code could destroy systems)            → 0.56
  Autonomy Impact:  High (agent generates code attacker controls)          → 0.56
  Data Dependence:  None (attack is in model weights, no data needed)      → 0.10

CALCULATED BASE SCORE: 8.2 (HIGH)

KEY INSIGHT: Lower exploitability (hard to implant) but
             when triggered: certain success + full autonomy impact
             Data Dependence = None → mitigation requires model replacement
```

---

## AIVSS Temporal Modifiers

The base score is adjusted by temporal factors:

```
TEMPORAL METRIC     VALUES              EFFECT
─────────────────────────────────────────────────────────────────────
Exploit Maturity    Unproven/PoC/       Is there a working exploit?
                    Functional/High     Unproven → reduce score 0.8×

Remediation Level   Official Fix/       Can the vulnerability be fixed?
                    Temporary Fix/      For AI: model update? data fix?
                    Workaround/         Official Fix → reduce score 0.87×
                    Unavailable

Report Confidence   Unknown/            How certain is the vulnerability?
                    Reasonable/         Unknown → reduce score 0.92×
                    Confirmed

Example:
  Prompt injection, base score 8.7
  Exploit Maturity: Functional (0.97)
  Remediation: Workaround only (0.97) — no model fix, just input validation
  Report Confidence: Confirmed (1.0)

  Temporal Score: 8.7 × 0.97 × 0.97 × 1.0 = 8.2 (still HIGH)
```

---

## Using AIVSS in Practice

### Step 1: Discover the Vulnerability

Use structured discovery:
- Red-team exercises (Phase 5 Article 10)
- Automated scanning (Garak, PyRIT)
- Production incident reports
- Security research papers

### Step 2: Score It

```
AIVSS SCORING WORKSHEET:
─────────────────────────────────────────────────────────────────
SYSTEM: _______________________
VULNERABILITY: _______________________
DATE: _______________________

EXPLOITABILITY:
  Attack Vector:      [ ]Network [ ]Adjacent [ ]Local [ ]Physical
  Attack Complexity:  [ ]Low [ ]High
  Model Access:       [ ]API [ ]Local [ ]None
  Attack Success:     [ ]Certain [ ]High [ ]Medium [ ]Low [ ]VeryLow
  AI-Specificity:     [ ]General [ ]ModelSpecific

IMPACT:
  Confidentiality:    [ ]High [ ]Low [ ]None
  Integrity:          [ ]High [ ]Low [ ]None
  Availability:       [ ]High [ ]Low [ ]None
  Autonomy Impact:    [ ]High [ ]Low [ ]None
  Data Dependence:    [ ]High [ ]Low [ ]None

BASE SCORE: _______
SEVERITY:   [ ]Critical(9-10) [ ]High(7-8.9) [ ]Medium(4-6.9) [ ]Low(<4)

TEMPORAL ADJUSTMENTS:
  Exploit Maturity:   ___
  Remediation Level:  ___
  Report Confidence:  ___

TEMPORAL SCORE: _______
```

### Step 3: Prioritize Remediation

```
REMEDIATION PRIORITY:
  Critical (9.0-10.0): Fix immediately — treat as P0 incident
  High (7.0-8.9):      Fix within 30 days — P1 priority
  Medium (4.0-6.9):    Fix within 90 days — next sprint
  Low (<4.0):          Fix when convenient — backlog
```

### Step 4: Map Remediation to Data Dependence

AIVSS's Data Dependence metric directly guides your remediation strategy:

```
DATA DEPENDENCE = HIGH:
  "This vulnerability requires the attacker to control data
   that enters the agent's context (RAG, web content, uploads)"
  Remediation: Secure the data pipeline
    → Scan ingested documents (Phase 5 Article 2)
    → Label external content as untrusted (Phase 3 Article 3)
    → Implement RAG access controls (Phase 2 Article 6)

DATA DEPENDENCE = LOW:
  "The attacker controls the input directly (user messages, prompts)"
  Remediation: Harden the input handling
    → Prompt hardening (Phase 5 Article 5)
    → Input validation and scanning (Phase 5 Article 2)
    → HITL for high-impact actions (Phase 5 Article 4)

DATA DEPENDENCE = NONE:
  "The vulnerability is in the model weights themselves"
  Remediation: Supply chain and model management
    → Verify model provenance (Phase 3 Article 5)
    → Behavioral test suite (Phase 5 Article 10)
    → Model replacement if confirmed backdoor
```

---

## Further Reading

- [CVSS v4.0 Specification](https://www.first.org/cvss/v4-0/) — The base framework AIVSS extends
- [NIST NVD: CVE Database](https://nvd.nist.gov/) — Reference for traditional vulnerability scoring
- [AI Incident Database](https://incidentdatabase.ai/) — Real AI incidents to practice scoring
- [Zou et al.: Universal and Transferable Adversarial Attacks (2023)](https://arxiv.org/abs/2307.15043) — Score this: ASR=Certain for the suffix attack
- [OWASP: Risk Rating Methodology](https://owasp.org/www-community/OWASP_Risk_Rating_Methodology) — Related qualitative scoring

---

*← [Prev: EU AI Act](./05-eu-ai-act.md) | [Phase 7: Cutting Edge →](../07-cutting-edge/README.md)*
