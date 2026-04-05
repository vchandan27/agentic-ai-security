# 🤝 Contributing to Agentic AI Security

Thank you for wanting to contribute! This repo is built in public for the community.

## Article Template

Every article should follow this structure:

```markdown
# [Emoji] Title

> **Phase X · Article Y** | ⏱️ N min read | 🏷️ `#tag1` `#tag2`
> [Optional: Severity, OWASP mapping, MAESTRO layer]

## TL;DR
- Three bullet points max
- Each should be a standalone insight

## [Core Sections — adapt to topic]
...

## Further Reading
- Links to papers, tools, official docs

---
*← [Prev article](./link) | [Next: Article →](./link)*
```

## Guidelines

- **Don't hallucinate.** If you're not sure, say so or skip it.
- **Cite sources.** Link to papers, official docs, or reputable blogs.
- **Use diagrams.** Mermaid, ASCII art, and tables are preferred.
- **Be specific.** Vague advice helps no one. Concrete examples help everyone.
- **Keep it accurate.** Security misinformation is harmful.

## How to Submit

1. Fork the repo
2. Create a branch: `git checkout -b article/topic-name`
3. Write your article following the template
4. Submit a PR with a brief description of what you added

## What's Needed Most

Check the [open issues](https://github.com/vchandan27/agentic-ai-security/issues) — articles marked `help wanted` are the highest priority.

Current gaps:
- Phase 2 full articles (frameworks in practice)
- Phase 3 full articles (security fundamentals)
- Phase 5 defense articles beyond README
- Phase 6 articles: AIVSS, NIST AI RMF, EU AI Act, MITRE ATLAS
- Phase 7: OpenClaw, A2A security, confidential AI

## Code of Conduct

Be respectful. Focus on education, not exploitation. No PoCs that could be directly used for attacks against live systems.
