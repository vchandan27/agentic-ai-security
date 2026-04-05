# 📄 Essential Research Papers in Agentic AI Security

> **Phase 7 · Reference** | 🏷️ `#papers` `#research` `#reference`
> **Updated:** April 2026

---

## How to Use This List

These are the papers that matter. Not a comprehensive bibliography — a curated list of papers that actually changed how the field thinks about agentic AI security. Each has a note on *why* it matters.

---

## 🔴 Must-Read: Foundational Papers

### Prompt Injection

**"Not What You've Signed Up For: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection"**
Greshake et al. (2023) | [arxiv.org/abs/2302.12173](https://arxiv.org/abs/2302.12173)
> *The paper that put indirect prompt injection on the map. Demonstrated real attacks against Bing Chat, ChatGPT plugins, and GitHub Copilot. If you read one paper, read this one.*

**"Prompt Injection Attacks and Defenses in LLM-Integrated Applications"**
Liu et al. (2023) | [arxiv.org/abs/2310.12815](https://arxiv.org/abs/2310.12815)
> *Systematic taxonomy of injection attack types and defense mechanisms. Good reference for understanding the full attack space.*

**"InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated LLM Agents"**
Zhan et al. (2024) | [arxiv.org/abs/2403.02691](https://arxiv.org/abs/2403.02691)
> *First benchmark specifically for indirect injection in tool-using agents. Shows that most current agents are highly vulnerable.*

---

### Multi-Agent Attacks

**"AgentDojo: A Dynamic Environment to Evaluate Attacks and Defenses for LLM Agents"**
Debenedetti et al. (2024) | [arxiv.org/abs/2406.13352](https://arxiv.org/abs/2406.13352)
> *A comprehensive evaluation framework for testing agent security. Includes realistic attack scenarios across multiple task domains.*

**"Morris II: Against AI Agents Through Self-Replicating Adversarial Prompts"**
Nassi, Cohen, Bitton (2024) | [arxiv.org/abs/2403.02817](https://arxiv.org/abs/2403.02817)
> *First demonstration of a self-propagating AI worm. Alarming implications for networked multi-agent systems.*

---

### Backdoors & Sleeper Agents

**"Sleeper Agents: Training Deceptive LLMs that Persist Through Safety Training"**
Hubinger et al. / Anthropic (2024) | [arxiv.org/abs/2401.05566](https://arxiv.org/abs/2401.05566)
> *Shows that backdoors can be trained into LLMs that survive RLHF and other safety training. The most alarming supply chain paper in recent AI security.*

**"BadNets: Identifying Vulnerabilities in the Machine Learning Model Supply Chain"**
Gu et al. (2017) | [arxiv.org/abs/1708.06733](https://arxiv.org/abs/1708.06733)
> *The original paper on ML backdoor attacks. Sets the foundation for understanding model supply chain risks.*

---

### RAG & Memory Attacks

**"PoisonedRAG: Knowledge Poisoning Attacks to Retrieval-Augmented Generation"**
Zou et al. (2024) | [arxiv.org/abs/2402.07867](https://arxiv.org/abs/2402.07867)
> *Demonstrates systematic attacks against RAG systems. Shows how few poisoned documents can compromise an entire knowledge base.*

---

### Adversarial Attacks

**"Universal and Transferable Adversarial Attacks on Aligned Language Models"**
Zou et al. (2023) | [arxiv.org/abs/2307.15043](https://arxiv.org/abs/2307.15043)
> *GCG attack — automated generation of adversarial suffixes that jailbreak aligned models. Transferable across models.*

**"Visual Prompt Injection: Ignoring Instructions Placed in Images"**
Ba et al. (2023) | [arxiv.org/abs/2309.00236](https://arxiv.org/abs/2309.00236)
> *Demonstrates that instructions hidden in images can override text instructions for multimodal LLMs.*

---

## 🟠 Important: Architecture & Defense Papers

**"ReAct: Synergizing Reasoning and Acting in Language Models"**
Yao et al. (2022) | [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)
> *The foundational paper on the ReAct pattern. Understanding the attack surface of agents requires understanding how they work.*

**"Cognitive Architectures for Language Agents"**
Sumers et al. (2023) | [arxiv.org/abs/2309.02427](https://arxiv.org/abs/2309.02427)
> *Comprehensive survey of agent architectures. Excellent for understanding the full attack surface.*

**"MemGPT: Towards LLMs as Operating Systems"**
Packer et al. (2023) | [arxiv.org/abs/2310.08560](https://arxiv.org/abs/2310.08560)
> *Introduces hierarchical memory for agents. Understanding memory architecture is prerequisite to defending it.*

---

## 🟡 Emerging: 2025-2026 Papers to Watch

**MCP Security Research**
*Active area — watch: Anthropic security blog, CSA publications, USENIX Security 2025*

**A2A Protocol Security Analysis**
*Google A2A is new (2024) — formal security analysis forthcoming*

**Agentic AI Benchmark Development**
*AgentBench, AgentDojo, SWE-bench evolving to include security evaluations*

---

## 📚 Books

| Book | Author | Why Read It |
|------|--------|-------------|
| *Hacking AI* | Gary McGraw & Harold Figueroa | Security risks in ML systems — foundational |
| *Building LLM Powered Applications* | Valentina Alto | How production apps are built — know what you're securing |
| *Designing Machine Learning Systems* | Chip Huyen | Data pipelines, drift, reliability — prerequisite to threat modeling |
| *The Art of Intrusion* | Kevin Mitnick | Classic attacker mindset — still relevant for agent red-teaming |

---

## 🌐 Ongoing Resources

| Resource | What to Follow |
|----------|----------------|
| [OWASP LLM Project](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | Top 10 updates, new vulnerability disclosures |
| [CSA AI Safety](https://cloudsecurityalliance.org/research/topics/artificial-intelligence) | MAESTRO updates, enterprise AI security guidance |
| [MITRE ATLAS](https://atlas.mitre.org) | Adversarial ML threat catalog, real-world incident tracking |
| [AI Village (DEF CON)](https://aivillage.org) | Practitioner research, CTF challenges |
| [Embrace the Red (Johann Rehberger)](https://embracethered.com) | Practical AI attack demonstrations — the best hands-on blog |
| [LLM Security](https://llmsecurity.net) | Research paper aggregator |
| [Huntr AI](https://huntr.com) | AI bug bounty — real vulnerability disclosures |

---

*← [Phase 7 Index](./README.md)*
