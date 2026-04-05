# 🛡️ Phase 5 — Securing Agents: Defenses & Architecture

> **Goal:** For every attack in Phase 4, learn a concrete, implementable defense. This phase is the practitioner's guide to building secure agents.

---

## The Defense Stack

```
┌────────────────────────────────────────────────────────────┐
│                  AGENT DEFENSE IN DEPTH                    │
│                                                            │
│  Layer 7: Human Oversight & Governance                     │
│  ─────────────────────────────────────                     │
│  Policies, audits, incident response                       │
│                                                            │
│  Layer 6: Monitoring & Detection                           │
│  ──────────────────────────────────                        │
│  Traces, anomaly detection, alerting                       │
│                                                            │
│  Layer 5: Output Controls                                  │
│  ────────────────────────                                  │
│  Sanitization, schema validation, DLP                      │
│                                                            │
│  Layer 4: Tool Execution Controls                          │
│  ─────────────────────────────────                         │
│  Validation, rate limiting, HITL gates                     │
│                                                            │
│  Layer 3: Reasoning Controls                               │
│  ───────────────────────────                               │
│  Prompt hardening, instruction hierarchy                   │
│                                                            │
│  Layer 2: Input Controls                                   │
│  ─────────────────────                                     │
│  Validation, sanitization, source trust rating             │
│                                                            │
│  Layer 1: Permissions & Least Privilege                    │
│  ──────────────────────────────────────                    │
│  Minimal tools, scoped access, IAM                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

No single layer is sufficient. All are necessary.

---

## Articles in This Phase

| # | Article | Attacks It Defends Against |
|---|---------|---------------------------|
| 5.1 | [🏰 Defense-in-Depth for Agents](./01-defense-in-depth.md) | All |
| 5.2 | [🧹 Input Validation & Output Sanitization](./02-input-output-validation.md) | LLM01, LLM05 |
| 5.3 | [📦 Sandboxing Tool Execution](./03-sandboxing.md) | LLM06, Tool Abuse |
| 5.4 | [🧑 Human-in-the-Loop Design](./04-human-in-the-loop.md) | All high-stakes actions |
| 5.5 | [💪 Prompt Hardening](./05-prompt-hardening.md) | LLM01, LLM07 |
| 5.6 | [🔌 Secure MCP Server Design](./06-secure-mcp.md) | LLM03, MCP Poisoning |
| 5.7 | [🆔 Agent Identity & Attestation](./07-agent-identity.md) | Multi-agent trust collapse |
| 5.8 | [👁️ Security Monitoring](./08-security-monitoring.md) | All |
| 5.9 | [⏱️ Rate Limiting & Scope Controls](./09-rate-limiting.md) | LLM10, DoS |
| 5.10 | [🔴 Red-Teaming Agents](./10-red-teaming-agents.md) | All |
| 5.11 | [🤝 Secure Multi-Agent Orchestration](./11-secure-multi-agent.md) | LLM06, Trust Collapse |
| 5.12 | [🧹 Data Minimization](./12-data-minimization.md) | LLM02, Exfiltration |

> 📝 *Full articles in progress — contributions welcome!*

---

## Quick Reference: Defense-to-Attack Mapping

| Attack (Phase 4) | Primary Defense | Secondary Defense |
|-----------------|-----------------|-------------------|
| Direct injection | Prompt hardening | Input validation |
| Indirect injection | Content compartmentalization | Tool call monitoring |
| Tool abuse | Least privilege | HITL for irreversible actions |
| Excessive agency | Minimal tool set | Scoped permissions |
| Memory poisoning | Ingestion scanning | RBAC on vector DB |
| Data exfiltration | Egress control | Content inspection |
| Agent hijacking | Goal anchoring | Action consistency check |
| Confused deputy | Instruction source verification | — |
| Trust collapse | Message signing | Sandboxed agent contexts |
| MCP poisoning | Server vetting | Version pinning |
| DoS | Budget limits | Rate limiting |
| Sleeper agents | Model provenance | Behavioral testing |
