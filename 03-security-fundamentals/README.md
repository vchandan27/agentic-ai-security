# 🔐 Phase 3 — Security Fundamentals for AI

> **Goal:** Bridge classical security knowledge to the AI context. If you already know security well, this phase recalibrates your models for agents. If you're an AI developer new to security, this phase gives you the vocabulary.

---

## Articles in This Phase

| # | Article | Key Concepts |
|---|---------|-------------|
| 3.1 | [🗺️ Threat Modeling for AI Systems](./01-threat-modeling-for-ai.md) | STRIDE, PASTA, attack trees adapted for agents |
| 3.2 | [🔒 CIA Triad in the AI Context](./02-cia-triad-in-ai.md) | Confidentiality, Integrity, Availability — redefined for LLMs |
| 3.3 | [🏚️ Trust Boundaries & Attack Surface](./03-trust-boundaries.md) | Why agents collapse traditional trust boundaries |
| 3.4 | [🔑 Principle of Least Privilege](./04-least-privilege.md) | Scoping agent permissions — tools, memory, data |
| 3.5 | [⛓️ Supply Chain Security](./05-supply-chain-security.md) | Model supply chain, poisoned dependencies, SBOMs |
| 3.6 | [🆔 Auth & Identity for Agents](./06-auth-and-identity.md) | OAuth flows, API key management, agent identity |
| 3.7 | [🌐 Zero Trust Architecture for Agents](./07-zero-trust-for-agents.md) | Applying ZTA principles to multi-agent systems |

> 📝 *Full articles in progress — foundations covered in Phase 1.*

---

## The Core Security Shift

Classical security assumes:
- **Known inputs** (structured data, typed fields)
- **Fixed behavior** (same input = same output)
- **Hard boundaries** (network perimeters, ACLs)

Agentic AI breaks all three assumptions:

```
Classical:  Input → [Deterministic Logic] → Output
                           ↑
                     You can reason about
                     all possible behaviors

Agentic:    Input → [Non-deterministic LLM Reasoning]
                    + [Dynamic Tool Selection]
                    + [Untrusted External Content]
                    + [Emergent Multi-step Behavior]
                    → Output (unpredictable)
```

The security frameworks in this phase give you the vocabulary to reason about this complexity.

---

## What Comes Next?

After Phase 3 → [Phase 4: Agentic AI Threat Taxonomy](../04-agentic-ai-threats/) — the 15 attack classes you need to know, with diagrams and real examples.
