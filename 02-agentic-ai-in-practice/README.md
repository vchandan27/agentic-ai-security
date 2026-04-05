# ⚙️ Phase 2 — Agentic AI in Practice

> **Goal:** Understand the real frameworks, protocols, and infrastructure that production agents run on. You can't threat-model a system you've never built.

---

## Why This Phase Matters for Security

Security researchers who understand the *implementation* find more vulnerabilities. Every framework in this phase has its own quirks, defaults, and misconfigurations that attackers exploit.

---

## Articles in This Phase

| # | Article | Key Frameworks / Concepts |
|---|---------|--------------------------|
| 2.1 | [🦜 LangChain & LangGraph](./01-langchain-langgraph.md) | The most widely used agent framework; stateful graph workflows |
| 2.2 | [🤖 AutoGen & CrewAI](./02-autogen-crewai.md) | Multi-agent conversation and role-based crews |
| 2.3 | [🧠 OpenAI & Anthropic Agent SDKs](./03-openai-anthropic-sdks.md) | First-party SDKs, handoffs, guardrails |
| 2.4 | [🔌 Model Context Protocol (MCP)](./04-model-context-protocol.md) | Universal tool standard — architecture and security implications |
| 2.5 | [🤝 Agent-to-Agent (A2A) Protocol](./05-agent-to-agent-protocol.md) | Google's A2A — agent identity, task delegation |
| 2.6 | [📚 RAG Systems Deep Dive](./06-rag-systems.md) | Retrieval pipelines, chunking, re-ranking, injection points |
| 2.7 | [🗄️ Vector Databases](./07-vector-databases.md) | Pinecone, Weaviate, Qdrant — storage security |
| 2.8 | [🔄 Agentic Workflows](./08-agentic-workflows.md) | LangGraph, Temporal — stateful vs stateless |
| 2.9 | [🌍 Real-World Agent Deployments](./09-real-world-deployments.md) | Code agents, browser agents, enterprise patterns |

> 📝 *Articles coming soon — contributions welcome!*

---

## Quick Reference: Framework Security Defaults

| Framework | Default Auth | Default Sandboxing | HITL Support |
|-----------|-------------|-------------------|--------------|
| LangChain | ❌ None | ❌ None | Manual |
| LangGraph | ❌ None | ❌ None | ✅ Interrupt nodes |
| AutoGen | ❌ None | Optional (Docker) | ✅ Human proxy agent |
| CrewAI | ❌ None | ❌ None | Limited |
| OpenAI SDK | ✅ API key | Limited | ✅ Guardrails |
| Claude SDK | ✅ API key | Limited | Manual |

> ⚠️ All frameworks rely on the **developer** to add security controls. None are secure by default.

---

## What Comes Next?

After Phase 2 → [Phase 3: Security Fundamentals](../03-security-fundamentals/) — threat modeling, CIA triad, and trust boundary analysis applied to AI systems.
