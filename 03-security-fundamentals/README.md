# 🔐 Phase 3: Security Fundamentals

> **Before you can understand how agentic AI gets attacked, you need to understand the security principles it's built on.**

This phase covers the foundational security concepts that every agentic AI practitioner needs — from the classic CIA triad to emerging topics like Zero Trust architectures designed specifically for autonomous agents.

---

## Why This Phase?

Most AI developers have deep ML knowledge but limited security background. Most security engineers understand traditional systems but not AI-specific risks. This phase bridges both worlds — grounding you in the security fundamentals *as they apply to AI agents*, not as abstract theory.

After this phase, you'll be able to:
- Classify any AI security threat using the CIA triad
- Map the trust boundaries in any agent architecture
- Design agent systems that follow least-privilege principles
- Understand your AI supply chain and its risks
- Build authentication systems that work for autonomous agents
- Apply Zero Trust principles to multi-agent deployments

---

## Articles

| # | Article | Core Concept | Read Time |
|---|---------|-------------|-----------|
| 01 | [Threat Modeling for AI](./01-threat-modeling-for-ai.md) | STRIDE + MITRE ATLAS applied to AI | 20 min |
| 02 | [CIA Triad in AI Systems](./02-cia-triad-in-ai.md) | Confidentiality, Integrity, Availability for agents | 18 min |
| 03 | [Trust Boundaries & Attack Surface](./03-trust-boundaries.md) | The 6 critical trust boundaries in agent systems | 20 min |
| 04 | [Principle of Least Privilege](./04-least-privilege.md) | 5 dimensions of agent privilege | 16 min |
| 05 | [Supply Chain Security](./05-supply-chain-security.md) | Models, packages, MCP servers, training data | 22 min |
| 06 | [Auth & Identity for Agents](./06-auth-and-identity.md) | OAuth, JWTs, mTLS, delegated authorization | 24 min |
| 07 | [Zero Trust Architecture](./07-zero-trust.md) | Never trust, always verify — for AI agents | 22 min |

**Total phase reading time: ~2.5 hours**

---

## Phase Map

```
┌─────────────────────────────────────────────────────────────────┐
│                  PHASE 3 LEARNING PATH                          │
│                                                                 │
│  START HERE                                                     │
│      │                                                          │
│      ▼                                                          │
│  ┌───────────────────────────────────────────────────────┐     │
│  │  01 — THREAT MODELING FOR AI                          │     │
│  │  Learn the language of security: STRIDE, ATLAS,       │     │
│  │  four questions. Foundation for everything else.      │     │
│  └──────────────────────────┬────────────────────────────┘     │
│                             │                                   │
│      ┌──────────────────────┼──────────────────────┐           │
│      ▼                      ▼                      ▼           │
│  ┌───────────┐        ┌───────────┐         ┌──────────────┐   │
│  │ 02 — CIA  │        │ 03 — TRUST│         │ 04 — LEAST   │   │
│  │  TRIAD    │        │ BOUNDARIES│         │  PRIVILEGE   │   │
│  │           │        │           │         │              │   │
│  │ Classify  │        │ Map your  │         │ Minimize the │   │
│  │ any threat│        │ attack    │         │ blast radius │   │
│  └─────┬─────┘        │ surface   │         └──────┬───────┘   │
│        │              └─────┬─────┘                │           │
│        └──────────────┬─────┘──────────────────────┘           │
│                       ▼                                         │
│  ┌───────────────────────────────────────────────────────┐     │
│  │  05 — SUPPLY CHAIN SECURITY                           │     │
│  │  Models, packages, MCP servers, training data         │     │
│  └──────────────────────────┬────────────────────────────┘     │
│                             │                                   │
│      ┌──────────────────────┼──────────────────────┐           │
│      ▼                                             ▼           │
│  ┌───────────────────┐               ┌─────────────────────┐   │
│  │ 06 — AUTH &       │               │ 07 — ZERO TRUST     │   │
│  │  IDENTITY         │               │  ARCHITECTURE       │   │
│  │                   │               │                     │   │
│  │ Who is the agent? │               │ Never trust,        │   │
│  │ On whose behalf?  │               │ always verify       │   │
│  └───────────────────┘               └─────────────────────┘   │
│                                                                 │
│  READY FOR → Phase 4: Agentic AI Threats                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Concepts Introduced in This Phase

| Concept | Article | Why It Matters |
|---------|---------|----------------|
| STRIDE threat taxonomy | 01 | Universal language for classifying threats |
| MITRE ATLAS | 01 | Adversarial AI-specific attack catalog |
| CIA triad | 02 | Framework for evaluating any security control |
| Trust boundaries (TB-1 → TB-6) | 03 | Where attacks happen in agent systems |
| 5 dimensions of privilege | 04 | Complete least-privilege model for agents |
| AI-BOM | 05 | Bill of Materials for AI supply chains |
| Delegated authorization | 06 | Agent acting on user's behalf securely |
| Policy engine (OPA) | 07 | Centralized, auditable access decisions |
| Assume breach design | 07 | Building for the inevitable compromise |

---

## The Core Security Shift

Classical security assumes:
- **Known inputs** (structured data, typed fields)
- **Fixed behavior** (same input = same output)
- **Hard boundaries** (network perimeters, ACLs)

Agentic AI breaks all three:

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

## Quick Reference: 7 Security Questions

```
When designing an agent, ask these 7 questions:

1. THREAT MODEL: What are the assets? Who are the adversaries?
                 What are the threats? (STRIDE them)

2. CIA: Does my design protect Confidentiality, Integrity, and
        Availability? Which is most at risk?

3. TRUST BOUNDARIES: Where does data cross trust levels?
                     Is it validated at every boundary?

4. LEAST PRIVILEGE: Does each agent/tool have ONLY what it needs?
                    Scope, depth, data, autonomy, time?

5. SUPPLY CHAIN: What am I depending on? Is each dependency trusted?
                 Models, packages, MCP servers, external data?

6. IDENTITY: Who is this agent? On whose behalf? What's authorized?
             Is every action attributable to a human?

7. ZERO TRUST: Am I assuming any component might be compromised?
               Is every request verified? Is the blast radius limited?
```

---

## Navigation

← [Phase 2: Agentic AI in Practice](../02-agentic-ai-in-practice/README.md) | [Phase 4: Agentic AI Threats →](../04-agentic-ai-threats/README.md)
