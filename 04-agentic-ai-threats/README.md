# 💀 Phase 4 — Agentic AI Threat Taxonomy

> **Goal:** Master every known attack class targeting AI agents. For each threat, understand the mechanism, a realistic attack scenario, and what makes it dangerous.

---

## The Agentic Threat Landscape

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AGENTIC AI ATTACK SURFACE                        │
│                                                                     │
│  INPUT LAYER          REASONING LAYER        ACTION LAYER          │
│  ─────────────        ───────────────        ────────────          │
│  • Direct Prompt      • Goal Hijacking        • Tool Abuse         │
│    Injection          • Confused Deputy       • Data Exfil         │
│  • Indirect Prompt    • Non-determinism       • Resource DoS       │
│    Injection            Exploitation                               │
│                                                                     │
│  MEMORY LAYER         MULTI-AGENT LAYER       MODEL LAYER         │
│  ─────────────        ─────────────────       ───────────         │
│  • Memory Poisoning   • Trust Collapse        • Backdoors         │
│  • RAG Poisoning      • Rogue Orchestrator    • Adversarial       │
│  • Context Stuffing   • Agent Impersonation     Inputs            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## All 15 Articles

| # | Attack | Severity | OWASP Mapping |
|---|--------|----------|---------------|
| [4.1](./01-prompt-injection-direct.md) | **Direct Prompt Injection** | 🔴 Critical | LLM01 |
| [4.2](./02-prompt-injection-indirect.md) | **Indirect Prompt Injection** | 🔴 Critical | LLM01 |
| [4.3](./03-tool-abuse.md) | **Tool Abuse & Misuse** | 🔴 Critical | LLM06 |
| [4.4](./04-excessive-agency.md) | **Excessive Agency** | 🟠 High | LLM06 |
| [4.5](./05-memory-poisoning.md) | **Memory & RAG Poisoning** | 🔴 Critical | LLM08 |
| [4.6](./06-data-exfiltration.md) | **Data Exfiltration via Agents** | 🔴 Critical | LLM02 |
| [4.7](./07-agent-hijacking.md) | **Agent Hijacking / Goal Takeover** | 🔴 Critical | LLM01 |
| [4.8](./08-confused-deputy.md) | **Confused Deputy Attack** | 🟠 High | LLM06 |
| [4.9](./09-multi-agent-trust-collapse.md) | **Multi-Agent Trust Collapse** | 🔴 Critical | LLM01 |
| [4.10](./10-mcp-poisoning.md) | **MCP Poisoning** | 🔴 Critical | LLM03 |
| [4.11](./11-insecure-tool-outputs.md) | **Insecure Tool Output Handling** | 🟠 High | LLM05 |
| [4.12](./12-denial-of-service.md) | **Denial of Service / Resource Exhaustion** | 🟠 High | LLM10 |
| [4.13](./13-model-inversion.md) | **Model Inversion & Data Extraction** | 🟡 Medium | LLM02 |
| [4.14](./14-adversarial-inputs.md) | **Adversarial Inputs to Perception** | 🟡 Medium | LLM04 |
| [4.15](./15-sleeper-agents-backdoors.md) | **Sleeper Agents & Backdoored Models** | 🔴 Critical | LLM03 |

---

## Severity Guide

| Level | Meaning |
|-------|---------|
| 🔴 Critical | Can result in full agent takeover, data breach, or real-world financial/physical harm |
| 🟠 High | Significant impact; requires specific conditions to exploit |
| 🟡 Medium | Limited impact or difficult to exploit at scale |

---

## What Comes Next?

After understanding attacks → [Phase 5: Securing Agents](../05-securing-agents/) — concrete defenses for every threat in this phase.
