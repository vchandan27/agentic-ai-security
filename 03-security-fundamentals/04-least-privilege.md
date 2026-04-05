# 🔏 Principle of Least Privilege for AI Agents

> **Phase 3 · Article 4 of 7** | ⏱️ 16 min read | 🏷️ `#security` `#least-privilege` `#access-control`

---

## TL;DR

- **Principle of Least Privilege (PoLP):** Every component gets the minimum access needed to do its job — nothing more.
- For AI agents, this means: minimum tools, minimum permissions per tool, minimum data access, minimum autonomy, and minimum time with those privileges.
- The biggest mistake in agent design is over-permissioning "for convenience" — this turns every bug and attack into a catastrophe.

---

## Why PoLP Matters More for Agents

In traditional systems, PoLP is a best practice. For AI agents, it's **existential**.

```
TRADITIONAL SOFTWARE:
  Bug in code → predictable bad outcome (crash, wrong output)
  You can reason about the worst case at design time

AGENTIC AI:
  Bug in reasoning → agent takes unexpected action with ALL its powers
  You CANNOT fully predict what an over-permissioned agent will do

Example:
  Agent with read/write/delete file permissions + non-deterministic LLM
  User: "Clean up temporary files"
  Agent (misinterpreted): rm -rf /home/user/Documents/

  With least privilege (read-only + temp_dir_only):
  Agent fails safely — can't touch Documents even if it tries.
```

---

## The Five Dimensions of Agent Privilege

Privilege for AI agents must be considered across five dimensions:

```
┌─────────────────────────────────────────────────────────────────┐
│              5 DIMENSIONS OF AGENT PRIVILEGE                    │
│                                                                 │
│  1. TOOL SCOPE                                                  │
│     Which tools can the agent call?                             │
│     [Minimize: only tools required for the task]               │
│                                                                 │
│  2. TOOL DEPTH                                                  │
│     What can each tool do?                                      │
│     [Minimize: read-only where possible, scope file paths]      │
│                                                                 │
│  3. DATA ACCESS                                                 │
│     What data can the agent retrieve?                           │
│     [Minimize: agent sees only what the user can see]          │
│                                                                 │
│  4. AUTONOMY LEVEL                                              │
│     How much can the agent decide without asking?              │
│     [Minimize: HITL for high-impact actions]                   │
│                                                                 │
│  5. TIME (Temporal Privilege)                                   │
│     How long does the agent hold its privileges?               │
│     [Minimize: short-lived tokens, revoke after task]          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Dimension 1: Tool Scope

**Don't give agents tools they don't need.**

```
BAD: One super-agent with all tools
─────────────────────────────────────────────────────────────────
Agent tools: web_search, read_file, write_file, execute_code,
             send_email, query_db, delete_db_record, call_api,
             read_calendar, create_calendar_event, open_pr,
             deploy_to_production, access_secret_vault

If this agent is compromised → attacker has all of the above.

GOOD: Purpose-built agents with minimal toolsets
─────────────────────────────────────────────────────────────────
Customer Support Agent:  lookup_order, create_ticket
Research Agent:          web_search, read_file, create_summary
Code Review Agent:       read_file, post_comment
Analytics Agent:         sql_readonly, create_chart
Deploy Agent (HITL):     run_tests, deploy_to_staging
                         [deploy_to_production requires human]
```

**Implementing tool scope in code:**

```python
# ❌ WRONG: All-access agent
general_agent = Agent(
    tools=[web_search, read_file, write_file, execute_code,
           send_email, query_db, delete_records, deploy]  # 🚩
)

# ✅ RIGHT: Purpose-limited agents
customer_support_agent = Agent(
    tools=[lookup_order, create_ticket],  # ONLY what's needed
    name="CustomerSupportAgent"
)

research_agent = Agent(
    tools=[web_search, read_file, create_summary],
    name="ResearchAgent"
)

# If you need broader capability, use an orchestrator that
# delegates to the right specialist agent:
orchestrator = Agent(
    tools=[delegate_to_support, delegate_to_research],
    sub_agents=[customer_support_agent, research_agent]
)
```

---

## Dimension 2: Tool Depth

Even within a single tool, you can limit what it can do.

### File System Tools

```python
# ❌ WRONG: Unrestricted file access
def read_file(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()

# Allows: read_file("/etc/passwd"), read_file("~/.ssh/id_rsa")

# ✅ RIGHT: Path-restricted file access
ALLOWED_DIRS = ["/data/uploads/", "/data/reports/"]

def read_file(path: str) -> str:
    # Normalize to prevent path traversal
    real_path = os.path.realpath(path)

    if not any(real_path.startswith(d) for d in ALLOWED_DIRS):
        raise SecurityError(f"Path {path} outside allowed directories")

    with open(real_path, 'r') as f:
        return f.read()
```

### Database Tools

```python
# ❌ WRONG: Full DB access
def query_db(sql: str) -> list:
    return db.execute(sql)  # Agent can run any SQL

# ✅ RIGHT: Read-only + table allowlist
ALLOWED_TABLES = {"orders", "products", "categories"}

def query_db_readonly(sql: str) -> list:
    # Parse the SQL to check it's SELECT only
    parsed = sqlparse.parse(sql)[0]
    if parsed.get_type() != 'SELECT':
        raise SecurityError("Only SELECT queries are allowed")

    # Check no disallowed tables are referenced
    tables = extract_tables(parsed)
    if not tables.issubset(ALLOWED_TABLES):
        raise SecurityError(f"Query references unauthorized tables: {tables - ALLOWED_TABLES}")

    # Use read-only database connection
    return readonly_db.execute(sql)
```

### Email Tools

```python
# ❌ WRONG: Send to any address
def send_email(to: str, subject: str, body: str):
    smtp.send(to, subject, body)

# ✅ RIGHT: Allowlist recipients or restrict to internal domain
INTERNAL_DOMAIN = "@company.com"
ALLOWED_EXTERNAL = {"vendor@trusted-partner.com"}

def send_email(to: str, subject: str, body: str):
    if not (to.endswith(INTERNAL_DOMAIN) or to in ALLOWED_EXTERNAL):
        raise SecurityError(f"Cannot send to external address: {to}")
    smtp.send(to, subject, body)
```

---

## Dimension 3: Data Access

Agents should only retrieve data that the requesting user is authorized to see.

```
THE PRINCIPLE:
  Agent's data access ≤ User's data access

If a user cannot access HR records, the agent working on their
behalf cannot access HR records — even if the agent technically
has database access.

This is different from tool scope: the agent might have the
sql_query tool, but the user's permissions restrict which
tables/rows that query can touch.
```

**Implementing user-scoped data access:**

```python
class AgentContext:
    def __init__(self, user: User):
        self.user = user
        self.permissions = user.get_permissions()

    def query_knowledge_base(self, query: str) -> list[Document]:
        return vectordb.search(
            query=query,
            filter={
                "access_level": {"$in": self.permissions.accessible_levels},
                "department": {"$in": self.permissions.departments}
            }
        )

    def query_database(self, sql: str) -> list:
        # Use a DB connection that has the user's row-level security
        # context applied at the database level
        conn = db.get_connection_for_user(self.user.id)
        return conn.execute(sql)  # DB enforces RLS automatically
```

---

## Dimension 4: Autonomy Level

**Not every action needs agent-autonomous execution.** Match the autonomy level to the risk.

```
AUTONOMY DECISION FRAMEWORK:
─────────────────────────────────────────────────────────────────

                    REVERSIBLE?
                   YES          NO
                ┌──────────────┬───────────────────────────────┐
  LOW     RISK  │ Autonomous   │ Confirm with user             │
                │ (read-only,  │ ("I'm about to send an email  │
                │  searches)   │  to customer. Proceed?")      │
                ├──────────────┼───────────────────────────────┤
  HIGH    RISK  │ Confirm with │ ALWAYS human approval         │
                │ user before  │ ("Transfer $10,000. CFO       │
                │ proceeding   │  must approve this action.")  │
                └──────────────┴───────────────────────────────┘

Risk assessment:
  HIGH RISK = financial, legal, communications, deployments,
              data deletion, account changes, external access
  LOW RISK  = reading, searching, reporting, summarizing
```

**Pattern: Graduated autonomy by task type:**

```python
class TaskClassifier:
    AUTONOMOUS = ["search", "read", "summarize", "analyze"]
    CONFIRM    = ["write", "update", "create", "schedule"]
    HITL       = ["delete", "send", "deploy", "pay", "transfer"]

    def get_autonomy_level(self, tool_name: str) -> str:
        for level, tools in [
            ("autonomous", self.AUTONOMOUS),
            ("confirm", self.CONFIRM),
            ("hitl", self.HITL)
        ]:
            if any(t in tool_name for t in tools):
                return level
        return "hitl"  # Default to human oversight for unknown tools

# In agent execution:
autonomy = TaskClassifier().get_autonomy_level(tool_call.name)

if autonomy == "autonomous":
    result = execute_tool(tool_call)
elif autonomy == "confirm":
    user_confirmed = await ask_user(f"Confirm: {tool_call.description}?")
    result = execute_tool(tool_call) if user_confirmed else None
elif autonomy == "hitl":
    approval = await request_human_approval(tool_call)
    result = execute_tool(tool_call) if approval.granted else None
```

---

## Dimension 5: Temporal Privilege

Privilege should exist only for as long as it's needed.

```
TEMPORAL PRIVILEGE PATTERNS:

1. JUST-IN-TIME ACCESS:
   Agent requests access at task start → access granted for task duration
   → access automatically revoked at task end

   Instead of: agent always has write access to S3
   Use: agent requests S3 write token → gets 15-min token → writes → token expires

2. TASK-SCOPED CREDENTIALS:
   Each agent run gets a new, unique API key
   Key is revoked when the task completes or times out
   No shared long-lived credentials

3. STEP-SCOPED PRIVILEGES:
   In multi-step workflows, escalate privileges only for the step that needs them
   Step 1 (research): read-only tools
   Step 2 (draft):    read + write to draft store
   Step 3 (publish):  [requires explicit human step to grant publish access]
```

**Implementation with short-lived tokens:**

```python
import boto3
from datetime import timedelta

def create_task_credentials(task_id: str, required_permissions: list) -> dict:
    """Create short-lived credentials scoped to this specific task."""
    sts = boto3.client('sts')

    # Generate temporary credentials valid for only 15 minutes
    creds = sts.assume_role(
        RoleArn=f"arn:aws:iam::123456789:role/agent-task-{task_id}",
        RoleSessionName=f"task-{task_id}",
        DurationSeconds=900,  # 15 minutes
        Policy=json.dumps({
            "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Action": required_permissions,
                "Resource": f"arn:aws:s3:::my-bucket/tasks/{task_id}/*"
            }]
        })
    )
    return creds["Credentials"]
```

---

## Blast Radius: PoLP in Practice

The blast radius of an attack = what the attacker can do with compromised agent privileges.

```
BLAST RADIUS COMPARISON:
─────────────────────────────────────────────────────────────────

OVER-PERMISSIONED AGENT:
  Agent has: all_tools, all_db_tables, send_email_any_address
             execute_code, deploy_production, manage_users

  If agent is compromised:
  ☠️  Attacker can: exfiltrate entire DB, deploy backdoors,
      create admin accounts, send phishing to all users

LEAST-PRIVILEGE AGENT (same task):
  Agent has: query_db(readonly, orders_table_only),
             create_support_ticket, send_email(@company.com_only)

  If agent is compromised:
  😐 Attacker can: read order data, create junk tickets,
     send emails internally (visible, auditable)

Blast radius reduced from "complete system compromise"
to "limited, auditable, recoverable damage."
```

---

## PoLP Anti-Patterns

```
❌ "Give it everything, we'll restrict later"
   → You will not restrict later. The agent will be in production
     with full permissions, and restricting later breaks things.

❌ "The LLM is smart enough not to misuse tools"
   → The LLM doesn't reason about permissions — it executes tool calls.
     Even a well-aligned model can be injected into misusing tools.

❌ "We need admin access for some edge cases"
   → Use just-in-time escalation for those cases.
     Don't grant permanent admin for occasional admin needs.

❌ "Our agents are internal-only so PoLP doesn't matter"
   → Insider threats are real. Compromised internal sessions are real.
     PoLP protects you from them too.

❌ "Least privilege will slow down the agent"
   → The agent doesn't need speed at the cost of security.
     Well-scoped agents are faster anyway — fewer irrelevant tools =
     better routing = faster task completion.
```

---

## PoLP Checklist for Agent Design

```
TOOL SCOPE:
[ ] List every tool the agent will ever need
[ ] Remove any tool not essential for the primary use case
[ ] Separate high-risk tools into separate agents with HITL
[ ] No "just in case" tools in production agents

TOOL DEPTH:
[ ] File tools restricted to specific allowed directories
[ ] Database tools use read-only connections where possible
[ ] Email tools restricted to allowlisted recipients
[ ] Web fetch tools blocked from private IP ranges
[ ] Code execution tools sandboxed (no network, no filesystem)

DATA ACCESS:
[ ] Agent data retrieval scoped to the requesting user's permissions
[ ] Database connections use per-user RLS contexts
[ ] RAG retrieval filtered by user's access level
[ ] No global admin DB connections for agent code

AUTONOMY:
[ ] Risk classification for every tool (autonomous/confirm/HITL)
[ ] Irreversible actions always require human approval
[ ] Financial and communication tools always HITL
[ ] Autonomy level reviewed when new tools are added

TEMPORAL:
[ ] Agent credentials are task-scoped, not permanent
[ ] Short-lived tokens used (15-60 minutes maximum)
[ ] Credentials revoked automatically on task completion/timeout
[ ] No shared credentials between agent instances
```

---

## Further Reading

- [NIST SP 800-53: Security Controls — Least Privilege (AC-6)](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [OWASP: LLM06 - Excessive Agency](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Anthropic: Responsible Development and Maintenance of Advanced AI](https://www.anthropic.com/research/responsible-scaling-policy)
- [Google: Securing AI Agents with Least Privilege](https://security.googleblog.com/)

---

*← [Prev: Trust Boundaries](./03-trust-boundaries.md) | [Next: Supply Chain Security →](./05-supply-chain-security.md)*
