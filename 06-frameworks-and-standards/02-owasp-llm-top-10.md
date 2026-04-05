# 🛡️ OWASP Top 10 for LLM Applications (2025)

> **Phase 6 · Article 2** | ⏱️ 20 min read | 🏷️ `#framework` `#owasp` `#top10`
> **Source:** OWASP Foundation | **Version:** 2025

---

## TL;DR

- The OWASP LLM Top 10 is the **canonical vulnerability classification** for LLM-based systems — required reading for every AI security practitioner.
- The 2025 edition adds agentic AI-specific risks and reflects the evolution from basic chatbots to autonomous agents.
- Every attack in Phase 4 of this repo maps to one or more OWASP categories.

---

## The List

```
┌────────────────────────────────────────────────────────────────┐
│               OWASP TOP 10 FOR LLM APPS (2025)                │
├──────┬─────────────────────────────────┬──────────────────────┤
│ Rank │ Vulnerability                   │ Quick Description    │
├──────┼─────────────────────────────────┼──────────────────────┤
│ LLM01│ Prompt Injection                │ Override instructions│
│ LLM02│ Sensitive Information Disclosure│ Leak private data    │
│ LLM03│ Supply Chain Vulnerabilities    │ Poisoned deps/models │
│ LLM04│ Data & Model Poisoning          │ Corrupt training/RAG │
│ LLM05│ Improper Output Handling        │ Unsafe output use    │
│ LLM06│ Excessive Agency                │ Too much permission  │
│ LLM07│ System Prompt Leakage           │ Expose instructions  │
│ LLM08│ Vector & Embedding Weaknesses   │ RAG manipulation     │
│ LLM09│ Misinformation                  │ Hallucinated actions │
│ LLM10│ Unbounded Consumption           │ Resource exhaustion  │
└──────┴─────────────────────────────────┴──────────────────────┘
```

---

## LLM01 — Prompt Injection

**What it is:** Attacker crafts input that overrides the model's instructions.

```
Direct:   User input designed to override system prompt
Indirect: Malicious instructions hidden in external content
          (documents, web pages, emails, tool outputs)
```

**Why it's #1:** Every other attack often has prompt injection as the entry point. It's the SQL injection of the LLM era.

**Agentic amplification:** In a pure chatbot, injection produces harmful words. In an agent, it produces harmful *actions* — sending emails, deleting files, making payments.

**Mitigation:**
- Instruction hierarchy (system > user > external data)
- Input/output validation
- Tool call anomaly detection
- Human approval for high-risk actions

**Maps to Phase 4:** [4.1 Direct](../04-agentic-ai-threats/01-prompt-injection-direct.md) | [4.2 Indirect](../04-agentic-ai-threats/02-prompt-injection-indirect.md) | [4.7 Hijacking](../04-agentic-ai-threats/07-agent-hijacking.md)

---

## LLM02 — Sensitive Information Disclosure

**What it is:** The LLM reveals confidential data — training data, system prompts, other users' information, or data from its context.

```
Examples:
  • Model trained on private emails → reveals snippets
  • Agent with access to user A's data → gives it to user B
  • System prompt contains API keys → extracted by attacker
  • RAG knowledge base → enumerated via queries
```

**Why it's critical:** Agents often have broad data access. A successful injection that causes information disclosure can be catastrophic — GDPR violations, competitive intelligence leaks, credential exposure.

**Mitigation:**
- Data minimization (agent only accesses what it needs)
- System prompt confidentiality instructions
- Strict session isolation in multi-user systems
- PII detection in outputs before delivery
- Differential privacy for sensitive training data

**Maps to Phase 4:** [4.6 Exfiltration](../04-agentic-ai-threats/06-data-exfiltration.md) | [4.13 Model Inversion](../04-agentic-ai-threats/13-model-inversion.md)

---

## LLM03 — Supply Chain Vulnerabilities

**What it is:** Risks from third-party components in the LLM pipeline — models, datasets, plugins, libraries, fine-tuning services.

```
Supply chain attack surfaces:
  • Base model from provider (backdoor at model level)
  • Training/fine-tuning dataset (data poisoning)
  • MCP servers and plugins (tool-level compromise)
  • Python/npm packages in agent codebase
  • Third-party APIs the agent calls
  • Model hubs (Hugging Face, etc.)
```

**Why it's critical:** You inherit risk from every component you don't build yourself. An attack on a widely-used MCP server could compromise thousands of agents simultaneously.

**Mitigation:**
- Vet all third-party models and tools before use
- Pin dependency versions
- Integrity check model weights
- Review MCP server code before installation
- Use SBOM (Software Bill of Materials) for AI systems

**Maps to Phase 4:** [4.10 MCP Poisoning](../04-agentic-ai-threats/10-mcp-poisoning.md) | [4.15 Sleeper Agents](../04-agentic-ai-threats/15-sleeper-agents-backdoors.md)

---

## LLM04 — Data & Model Poisoning

**What it is:** Corrupting the data that trains or informs the model, or the model weights themselves.

```
Training data poisoning:
  • Plant malicious examples in training dataset
  • Model learns the backdoor behavior
  • Behavior triggered by specific inputs

RAG/Fine-tuning poisoning:
  • Inject malicious documents into knowledge base
  • Fine-tune on adversarially crafted data
  • Model behavior changes without obvious detection
```

**Why it's critical:** Poisoning attacks target the foundation. Unlike application-level attacks, you can't patch a poisoned model — you have to retrain.

**Mitigation:**
- Data provenance and integrity checks at ingestion
- Anomaly detection in training data
- Behavioral testing post-training
- Red-team evaluation before deployment

**Maps to Phase 4:** [4.5 Memory Poisoning](../04-agentic-ai-threats/05-memory-poisoning.md) | [4.15 Backdoors](../04-agentic-ai-threats/15-sleeper-agents-backdoors.md)

---

## LLM05 — Improper Output Handling

**What it is:** Agent outputs are used downstream in unsafe ways — executed as code, rendered as HTML, passed to other systems without sanitization.

```
Examples:
  • Agent generates HTML → rendered in browser → XSS
  • Agent generates SQL → executed in DB → SQL injection
  • Agent generates shell command → executed → RCE
  • Agent output → another agent's input → injection cascade
  • Agent generates markdown → rendered with image URLs → exfil
```

**Why it's critical:** The agent becomes a code injection vector for downstream systems. The LLM didn't have a vulnerability — your code running its output did.

**Mitigation:**
- Never execute LLM output directly
- Sanitize/escape all LLM output before use in downstream systems
- Use structured output formats (JSON schema) instead of raw text
- Validate LLM output against expected schema

**Maps to Phase 4:** [4.11 Insecure Tool Outputs](../04-agentic-ai-threats/11-insecure-tool-outputs.md)

---

## LLM06 — Excessive Agency

**What it is:** The agent has more permissions, capabilities, or autonomy than necessary — creating unnecessary blast radius.

```
Three dimensions:
  1. Too many tools (Swiss Army Knife anti-pattern)
  2. Tools with too much scope (full filesystem vs. one directory)
  3. Too much autonomy (acting without confirmation)
```

**Why it's critical:** Excessive agency converts every other attack from "interesting" to "catastrophic." It's the blast radius multiplier.

**Mitigation:**
- Principle of least privilege for tools and permissions
- Read-only by default; add write access explicitly
- Human-in-the-loop for irreversible actions
- Task-scoped tool provision

**Maps to Phase 4:** [4.4 Excessive Agency](../04-agentic-ai-threats/04-excessive-agency.md) | [4.3 Tool Abuse](../04-agentic-ai-threats/03-tool-abuse.md)

---

## LLM07 — System Prompt Leakage

**What it is:** The agent's system prompt (which contains instructions, business rules, tool definitions, and sometimes credentials) is extracted by an attacker.

```
Why system prompts are sensitive:
  • Contain business logic ("never offer discounts above 30%")
  • Reveal internal API endpoints
  • May contain hardcoded credentials
  • Map the agent's capabilities (attack planning roadmap)
  • Contain security controls (tells attacker what to bypass)
```

**Why it matters:** A leaked system prompt is a security audit of your agent. It tells attackers exactly what you're protecting, what tools are available, and what restrictions to bypass.

**Mitigation:**
- Explicit confidentiality instructions in system prompt
- Never include credentials in system prompts
- Test for prompt extraction regularly
- Use external secret management, not system prompt

**Maps to Phase 4:** [4.13 Model Inversion](../04-agentic-ai-threats/13-model-inversion.md)

---

## LLM08 — Vector & Embedding Weaknesses

**What it is:** Attacks that exploit the vector similarity search underlying RAG systems — including poisoned embeddings, adversarial retrieval manipulation, and cross-user retrieval.

```
Attack surface:
  • Embedding model can be fooled by adversarial text
  • Similar embeddings ≠ similar meaning (semantic gap)
  • Attacker crafts text that embeds near targeted queries
  • Cross-tenant retrieval if embeddings are not isolated
```

**Why it matters:** Your RAG system trusts retrieved content as contextually relevant. Attackers can manipulate *what gets retrieved* — without changing the query.

**Mitigation:**
- RBAC on vector DB collections
- Tenant isolation in multi-user RAG
- Monitor retrieval anomalies (unexpected sources)
- Validate retrieved content against expected topic

**Maps to Phase 4:** [4.5 Memory Poisoning](../04-agentic-ai-threats/05-memory-poisoning.md)

---

## LLM09 — Misinformation

**What it is:** The agent generates false, misleading, or hallucinated information — and then *acts on it* or passes it to users as fact.

```
Agentic amplification:
  • Agent hallucinates a company name → searches for it → finds wrong info
  • Agent hallucinates an API endpoint → calls it → unintended action
  • Agent hallucinates a regulatory requirement → incorrect compliance advice
  • Agent hallucinates user data → presents false facts to user
```

**Why it matters:** Hallucinations in agents don't just mislead — they can trigger real actions based on false premises. A hallucinated legal requirement can lead to actual compliance failures.

**Mitigation:**
- Retrieval-grounded responses (always cite sources)
- Confidence thresholds (express uncertainty, don't act if unsure)
- Human verification for high-stakes claims
- Cross-check important facts before acting

---

## LLM10 — Unbounded Consumption

**What it is:** Agents that consume unlimited resources — compute, tokens, API calls, money — either through design flaws or adversarial manipulation.

```
Attack vectors:
  • Recursive loop injection (attacker-triggered infinite loops)
  • Context window stuffing (maximize token cost per call)
  • Concurrent session flooding (many sessions × unlimited resources)
  • Sycophantic feedback loops (agent keeps trying, never satisfied)
```

**Why it matters:** Cloud AI is billed per token/call. Unbounded consumption attacks can bankrupt a service, create denial of service for legitimate users, and exhaust API rate limits.

**Mitigation:**
- Max steps per agent run
- Max tokens per session
- Max concurrent sessions per user
- Budget alerts and hard spending limits
- Loop detection in orchestration layer

**Maps to Phase 4:** [4.12 Denial of Service](../04-agentic-ai-threats/12-denial-of-service.md)

---

## Cross-Reference: Phase 4 → OWASP Mapping

| Phase 4 Attack | OWASP Category |
|----------------|---------------|
| Direct Prompt Injection | LLM01 |
| Indirect Prompt Injection | LLM01 |
| Tool Abuse | LLM06 |
| Excessive Agency | LLM06 |
| Memory/RAG Poisoning | LLM04, LLM08 |
| Data Exfiltration | LLM02 |
| Agent Hijacking | LLM01 |
| Confused Deputy | LLM06 |
| Multi-Agent Trust Collapse | LLM01 |
| MCP Poisoning | LLM03 |
| Insecure Tool Outputs | LLM05 |
| Denial of Service | LLM10 |
| Model Inversion | LLM02, LLM07 |
| Adversarial Inputs | LLM04 |
| Sleeper Agents | LLM03, LLM04 |

---

## Key Resources

- **Official:** [OWASP LLM Top 10 Project](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- **GitHub:** [OWASP/www-project-top-10-for-large-language-model-applications](https://github.com/OWASP/www-project-top-10-for-large-language-model-applications)
- **PDF:** Download the full guide from the OWASP project page

---

*← [Prev: MAESTRO](./01-maestro-framework.md) | [Next: AIVSS →](./03-aivss.md)*
