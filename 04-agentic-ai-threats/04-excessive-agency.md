# 🏋️ Excessive Agency

> **Phase 4 · Attack 4 of 15** | ⏱️ 12 min read | 🏷️ `#attack` `#permissions` `#high`
> **Severity:** 🟠 High | **OWASP:** LLM06 | **MAESTRO Layer:** L4, L5

---

## TL;DR

- Excessive agency means an agent has **more permissions, tools, or capabilities than it needs** — creating unnecessary risk.
- This isn't an attack in itself — it's a **design flaw** that makes every other attack worse.
- The fix is applying Principle of Least Privilege: give agents only what they need, when they need it.

---

## What Is Excessive Agency?

Consider this: you hire a contractor to paint your living room. You give them keys to your entire house, access to your bank account, and admin rights to your home network — "just in case they need them."

That's excessive agency.

For AI agents, it looks like this:

```
Task: "Summarize my emails from last week"

Over-permissioned agent:
  ✅ read_email            ← needed
  ❌ send_email            ← not needed
  ❌ delete_email          ← not needed
  ❌ access_calendar       ← not needed
  ❌ read_contacts         ← not needed
  ❌ execute_code          ← not needed
  ❌ read_file("/")        ← definitely not needed

If this agent is compromised, the blast radius is enormous.
```

---

## The Three Dimensions of Excessive Agency

### Dimension 1: Excessive Permissions (Too Many Tools)
```
CORRECT:                        EXCESSIVE:
Email summarizer agent          Email summarizer agent
  Tools: [read_email]             Tools: [read_email, send_email,
                                          delete_email, calendar,
                                          file_system, web_search,
                                          execute_code]
```

### Dimension 2: Excessive Scope (Tools Too Powerful)
```
CORRECT:                        EXCESSIVE:
Read access to /documents/      Read access to entire filesystem
  file_read(path):                file_read(path):
    allowed = ["/documents/"]       allowed = ["/*"]
    if path not in allowed:
      raise PermissionError
```

### Dimension 3: Excessive Duration (Permissions Held Too Long)
```
CORRECT:                        EXCESSIVE:
Request permission when          Grant permission at startup
needed, revoke after use         and never revoke it

  On task start: request          Agent always has all
  specific tool access            permissions regardless
  After task: revoke              of what it's doing
```

---

## How Excessive Agency Amplifies Other Attacks

Excessive agency doesn't cause attacks — it makes them catastrophic:

```
Attack: Prompt injection
  Without excessive agency:
    Attacker injects → Agent tries to send email → No email tool → Fails
    Impact: 0

  With excessive agency:
    Attacker injects → Agent sends email with all data → Succeeds
    Impact: Data breach

Attack: Indirect injection via web content
  Without excessive agency:
    Agent reads page → Injection found → No code execution tool → Fails
    Impact: 0

  With excessive agency:
    Agent reads page → Injection found → Executes shell command → System compromised
    Impact: Full compromise
```

Excessive agency is the **force multiplier** for every other attack.

---

## Real-World Pattern: The "Swiss Army Knife" Agent Anti-Pattern

Many teams build agents with every possible tool "just in case" — thinking more tools = more capable. This creates what we call the Swiss Army Knife anti-pattern:

```
┌─────────────────────────────────────┐
│       SWISS ARMY KNIFE AGENT        │
│                                     │
│  "It can do everything!"            │
│                                     │
│  Tools: [all of them]               │
│  Memory: [everything]               │
│  Permissions: [unrestricted]        │
│                                     │
│  Capability: ██████████ (10/10)     │
│  Security:   ░░░░░░░░░░ (0/10)     │
└─────────────────────────────────────┘
```

The right design is purpose-specific agents with minimal permissions:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ EMAIL AGENT  │  │  CODE AGENT  │  │ SEARCH AGENT │
│              │  │              │  │              │
│ Tools:       │  │ Tools:       │  │ Tools:       │
│ read_email   │  │ read_file    │  │ web_search   │
│              │  │ write_file   │  │ fetch_url    │
│              │  │ execute_code │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
     No code            No email         No file I/O
```

---

## Mitigation: The Minimal Permission Checklist

Before deploying an agent, run through this:

```
FOR EACH TOOL THE AGENT HAS:
  □ Can you name a specific task in the agent's scope that requires it?
    If No → REMOVE IT

  □ Does the tool need write/delete access or would read-only suffice?
    If read-only suffices → DOWNGRADE TO READ-ONLY

  □ Does the tool need access to all data or just a subset?
    If subset → SCOPE IT (e.g., /documents/ not /)

  □ Does the agent need this tool for its entire lifetime?
    If No → IMPLEMENT JUST-IN-TIME PERMISSION GRANT

  □ Are there actions within this tool that should always require
    human confirmation?
    If Yes → ADD CONFIRMATION GATE
```

---

## MAESTRO Mapping

```
Layer 4 — Deployment & Infrastructure:
  Over-provisioned IAM roles for agent service accounts

Layer 5 — Agentic Applications:
  Business logic grants agents unnecessary capabilities
```

---

## Further Reading

- [OWASP LLM06: Excessive Agency](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [The Principle of Least Privilege in Agentic AI](https://www.anthropic.com/research/building-effective-agents)

---

*← [Prev: Tool Abuse](./03-tool-abuse.md) | [Next: Memory Poisoning →](./05-memory-poisoning.md)*
