# 🧹 Data Minimization for AI Agents

> **Phase 5 · Article 12 of 12** | ⏱️ 18 min read | 🏷️ `#defense` `#data-minimization` `#privacy` `#gdpr`

---

## TL;DR

- **Data minimization**: collect, process, and retain only the data strictly necessary for the agent's function — nothing more.
- AI agents are voracious data consumers. Left unchecked, they accumulate PII, business secrets, and sensitive context that becomes a liability.
- Data minimization reduces your breach impact, simplifies compliance (GDPR, HIPAA, CCPA), and limits exfiltration damage when agents are compromised.

---

## Why Agents Over-Collect Data

```
AGENT TENDENCY TO OVER-COLLECT:
─────────────────────────────────────────────────────────────────

Engineers building agents often think:
  "More context = better answers"
  "Let's keep everything in case the agent needs it later"
  "Logging everything helps us debug"

This leads to:
  ❌ Full conversation histories in plain text
  ❌ Raw tool outputs (containing PII) in context
  ❌ Entire documents loaded when only sections are needed
  ❌ Long-term memory storing every user interaction forever
  ❌ Logs containing API responses with sensitive customer data

The agent that "knows everything" is also the agent that
leaks everything when compromised.
```

---

## The Data Minimization Principles

```
┌─────────────────────────────────────────────────────────────────┐
│              DATA MINIMIZATION PRINCIPLES FOR AI                │
│                                                                 │
│  1. PURPOSE LIMITATION                                          │
│     Collect data only for the stated agent purpose.             │
│     A customer support agent doesn't need access to            │
│     employee HR records.                                        │
│                                                                 │
│  2. DATA ACCURACY                                               │
│     Keep data current. Stale data in RAG = wrong answers       │
│     AND unnecessary retention of old sensitive data.            │
│                                                                 │
│  3. STORAGE LIMITATION                                          │
│     Delete data when it's no longer needed.                     │
│     Session data: delete after session ends.                   │
│     Conversation logs: delete after 30/90 days.                │
│     Personal data: delete per user request.                    │
│                                                                 │
│  4. CONTEXT MINIMIZATION                                        │
│     Include only the minimum context needed for the current    │
│     step. Don't load an entire document when only a paragraph  │
│     is relevant.                                                │
│                                                                 │
│  5. OUTPUT MINIMIZATION                                         │
│     Return only the specific fields needed.                    │
│     Don't return full records when only a summary is needed.   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Practical Pattern 1: Context Minimization

Only load what's needed into the context window:

```python
class MinimalContextLoader:
    """
    Loads only the minimum context required for the current step.
    Avoids loading entire documents, full histories, or broad datasets.
    """

    def load_document_section(
        self,
        document_id: str,
        query: str,
        max_chars: int = 3000
    ) -> str:
        """
        Load only the relevant section of a document, not the full file.
        Use semantic search to find the most relevant passage.
        """
        # Don't load the whole document
        relevant_chunks = self.vector_store.search(
            query=query,
            filter={"document_id": document_id},
            top_k=3,
            max_chars_per_chunk=1000
        )

        # Return only what's needed, clearly bounded
        return "\n\n".join([
            f"[Excerpt from {doc.source}, page {doc.page}]:\n{doc.content}"
            for doc in relevant_chunks
        ])

    def load_conversation_history(
        self,
        session_id: str,
        max_turns: int = 5,
        max_chars: int = 4000
    ) -> list[dict]:
        """
        Load only recent conversation history, not the full log.
        Older turns often aren't needed and contain stale context.
        """
        recent_turns = self.session_store.get_recent_turns(
            session_id=session_id,
            limit=max_turns
        )
        # Truncate if needed
        result = []
        total_chars = 0
        for turn in reversed(recent_turns):
            turn_chars = len(turn["content"])
            if total_chars + turn_chars > max_chars:
                break
            result.insert(0, turn)
            total_chars += turn_chars
        return result

    def load_user_profile(
        self,
        user_id: str,
        needed_fields: list[str]
    ) -> dict:
        """
        Load only the specific user fields needed for this task.
        Don't load the full user record.
        """
        # ❌ WRONG: full_profile = db.get_user(user_id)
        # ✅ RIGHT: only the fields we need
        return db.get_user_fields(user_id, fields=needed_fields)
        # e.g., needed_fields = ["name", "account_tier"]
        # NOT: ["name", "ssn", "date_of_birth", "credit_card", ...]
```

---

## Practical Pattern 2: PII Detection and Redaction

Detect and redact PII before it enters long-term storage:

```python
import re

class PIIRedactor:
    """
    Detect and redact PII from agent inputs and outputs before
    storing them in logs, memory, or RAG databases.
    """

    PATTERNS = {
        "email": r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
        "phone_us": r'\b(?:\+1[-.]?)?\(?\d{3}\)?[-.]?\d{3}[-.]?\d{4}\b',
        "ssn": r'\b\d{3}-\d{2}-\d{4}\b',
        "credit_card": r'\b(?:\d{4}[-\s]?){3}\d{4}\b',
        "ip_address": r'\b(?:\d{1,3}\.){3}\d{1,3}\b',
        "aws_key": r'\b(AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}\b',
        "jwt": r'\beyJ[A-Za-z0-9+/=]+\.[A-Za-z0-9+/=]+\.[A-Za-z0-9+/=_-]+\b',
    }

    def redact(self, text: str, preserve_format: bool = True) -> tuple[str, list]:
        """
        Redact PII from text. Returns (redacted_text, list_of_findings).
        """
        redacted = text
        findings = []

        for pii_type, pattern in self.PATTERNS.items():
            matches = re.findall(pattern, redacted)
            if matches:
                findings.append({"type": pii_type, "count": len(matches)})
                if preserve_format:
                    # Replace with type indicator
                    redacted = re.sub(
                        pattern,
                        f"[{pii_type.upper()}_REDACTED]",
                        redacted
                    )
                else:
                    redacted = re.sub(pattern, "***", redacted)

        return redacted, findings

    def should_block(self, text: str, context: str = "output") -> bool:
        """
        For outputs going to external parties, block if high-sensitivity PII found.
        """
        _, findings = self.redact(text)
        high_sensitivity = {"ssn", "credit_card", "aws_key", "jwt"}
        found_types = {f["type"] for f in findings}
        return bool(found_types & high_sensitivity)
```

---

## Practical Pattern 3: Data Retention Policies

Automate data deletion to enforce retention limits:

```python
from datetime import datetime, timedelta
from dataclasses import dataclass

@dataclass
class RetentionPolicy:
    """Define how long different data types should be kept."""
    session_data_days: int = 1          # Delete after session ends + 1 day
    conversation_logs_days: int = 30    # Keep for debugging for 30 days
    audit_logs_days: int = 365          # Keep audit trail for 1 year
    rag_documents_days: int = 90        # Refresh knowledge base quarterly
    user_preferences_days: int = 730    # Keep user settings for 2 years
    personal_data_on_request: bool = True  # GDPR: delete within 30 days of request

class DataRetentionManager:
    def __init__(self, policy: RetentionPolicy, storage: 'DataStorage'):
        self.policy = policy
        self.storage = storage

    def run_retention_cleanup(self):
        """Run daily to enforce retention policies."""
        now = datetime.utcnow()
        deleted_counts = {}

        # Delete expired session data
        cutoff = now - timedelta(days=self.policy.session_data_days)
        count = self.storage.delete_sessions_before(cutoff)
        deleted_counts["sessions"] = count

        # Delete old conversation logs
        cutoff = now - timedelta(days=self.policy.conversation_logs_days)
        count = self.storage.delete_conversations_before(cutoff)
        deleted_counts["conversations"] = count

        # Note: audit logs are kept longer (compliance requirement)
        # They should be archived to cold storage, not deleted

        # Delete expired RAG documents
        cutoff = now - timedelta(days=self.policy.rag_documents_days)
        count = self.storage.delete_stale_rag_documents_before(cutoff)
        deleted_counts["rag_documents"] = count

        # Log the cleanup (in audit log — retained separately)
        audit_log.info("Data retention cleanup completed", counts=deleted_counts)
        return deleted_counts

    def handle_deletion_request(self, user_id: str) -> dict:
        """GDPR Article 17: Right to Erasure ('Right to be Forgotten')."""
        deleted = {}

        # Delete from all stores
        deleted["conversations"] = self.storage.delete_user_conversations(user_id)
        deleted["session_data"] = self.storage.delete_user_sessions(user_id)
        deleted["rag_contributions"] = self.storage.delete_user_rag_content(user_id)
        deleted["memory_entries"] = self.storage.delete_user_memory(user_id)
        deleted["preferences"] = self.storage.delete_user_preferences(user_id)

        # Audit trail: the deletion request itself must be logged
        # (you must be able to prove you complied)
        audit_log.info(
            "User data deletion completed",
            user_id=user_id,  # OK to keep this for compliance
            timestamp=datetime.utcnow().isoformat(),
            records_deleted=deleted
        )
        return deleted
```

---

## Practical Pattern 4: Output Scrubbing

Before returning data to users or logs, scrub unnecessary sensitive fields:

```python
class OutputScrubber:
    """
    Scrub sensitive fields from agent outputs before delivery.
    Ensure agents don't inadvertently leak data they accessed.
    """

    # Fields to always remove from any output
    ALWAYS_REMOVE = {
        "password", "hashed_password", "salt",
        "private_key", "secret_key", "api_key",
        "ssn", "full_credit_card", "cvv",
        "internal_user_id", "database_id"
    }

    def scrub_dict(self, data: dict, depth: int = 0) -> dict:
        """Recursively remove sensitive fields from a dict."""
        if depth > 10:
            return {}  # Prevent deep recursion DoS

        result = {}
        for key, value in data.items():
            # Remove known sensitive keys
            if key.lower() in self.ALWAYS_REMOVE:
                continue  # Drop this field entirely

            # Recursively scrub nested dicts
            if isinstance(value, dict):
                result[key] = self.scrub_dict(value, depth + 1)
            elif isinstance(value, list):
                result[key] = [
                    self.scrub_dict(item, depth + 1) if isinstance(item, dict) else item
                    for item in value
                ]
            else:
                result[key] = value

        return result

    def scrub_tool_result(self, tool_name: str, result: Any) -> Any:
        """
        Apply tool-specific scrubbing rules.
        Each tool may expose different sensitive fields.
        """
        if tool_name == "query_database" and isinstance(result, list):
            return [self.scrub_dict(row) for row in result]
        elif tool_name == "get_user_profile":
            return self.scrub_dict(result)
        else:
            return result
```

---

## Data Minimization in RAG Systems

RAG pipelines accumulate data — apply minimization at each stage:

```
RAG DATA MINIMIZATION:
─────────────────────────────────────────────────────────────────

INGESTION:
  ✅ Strip PII before indexing documents
  ✅ Index only documents relevant to agent's domain
  ✅ Set document TTL (auto-expire stale content)
  ✅ Track who uploaded each document (for deletion requests)

EMBEDDING STORAGE:
  ✅ Store only the chunk text, not the full document
  ✅ Apply field-level encryption to sensitive chunks
  ✅ Tag chunks with access level and expiry date

RETRIEVAL:
  ✅ Return only the top-K most relevant chunks (not all matches)
  ✅ Apply user access filters at query time
  ✅ Log what was retrieved per user (for audit)

GENERATION:
  ✅ Include only retrieved chunks in context, not full documents
  ✅ Do not include chunk content in the final answer unless necessary
  ✅ Summarize rather than quote verbatim when possible
```

---

## Data Minimization Checklist

```
CONTEXT MINIMIZATION:
[ ] Only relevant document sections loaded (semantic search, not full doc)
[ ] Conversation history limited to recent N turns
[ ] User profile: only needed fields loaded per request
[ ] Tool results scrubbed before entering context

PII HANDLING:
[ ] PII detection runs on all inputs before storage
[ ] PII detected in logs is redacted automatically
[ ] High-sensitivity PII (SSN, CC) blocks logging entirely
[ ] PII in RAG knowledge base removed before ingestion

DATA RETENTION:
[ ] Session data deleted within 1 day of session end
[ ] Conversation logs retained max 30 days (or less)
[ ] User data deletion (GDPR) can be triggered per request
[ ] Deletion requests fulfilled within 30 days
[ ] Deletion completion is itself audited

RAG/MEMORY:
[ ] Documents in RAG have expiry dates (TTL)
[ ] Stale documents purged on schedule
[ ] Memory entries have expiry (not infinite retention)
[ ] User can view and delete their own memory entries

OUTPUT:
[ ] Tool results scrubbed before delivery to users
[ ] Sensitive fields (passwords, keys, SSNs) never in output
[ ] Log output scrubbed of PII before writing to log store
[ ] Analytics data anonymized (no raw user data in metrics)
```

---

## Further Reading

- [GDPR Article 5: Principles of Data Processing (including Data Minimization)](https://gdpr-info.eu/art-5-gdpr/)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [OWASP: LLM02 — Insecure Output Handling](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Microsoft Presidio: PII Detection and Anonymization](https://github.com/microsoft/presidio)
- [EU AI Act: Data Governance Requirements](https://artificialintelligenceact.eu/)

---

*← [Prev: Secure Multi-Agent Orchestration](./11-secure-multi-agent.md) | [Next: MAESTRO Framework →](../06-frameworks-and-standards/01-maestro-framework.md)*
