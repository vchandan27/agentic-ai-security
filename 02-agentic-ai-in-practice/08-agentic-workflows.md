# 🔄 Agentic Workflows

> **Phase 2 · Article 8 of 9** | ⏱️ 15 min read | 🏷️ `#framework` `#workflows` `#orchestration`

---

## TL;DR

- Agentic workflows are long-running, stateful orchestrations — unlike simple API calls, they persist state across steps, handle failures, and can run for minutes, hours, or days.
- The two main approaches are **LangGraph** (graph-based, LLM-native) and **workflow engines like Temporal** (code-based, enterprise-grade).
- Stateful workflows are harder to attack mid-execution — but they also create persistent attack surfaces that outlast a single session.

---

## The Spectrum: Stateless to Stateful

```
STATELESS                                          STATEFUL
(no memory between calls)                     (persisted state)
        │                                               │
        ▼                                               ▼
  Single LLM call       Chain of calls        Long-running workflow
  "Answer this Q"       "Research topic"      "Manage project for 2 weeks"

  Risk: low             Risk: medium           Risk: high
  Blast radius: one     Blast radius: session  Blast radius: persistent
  response              scope                  real-world effects
```

As workflows become more stateful and long-running, the security implications compound: an attack that happens on day 3 of a 14-day workflow can corrupt everything that follows.

---

## LangGraph Workflows: The Graph Model

LangGraph represents workflows as directed graphs where:
- **Nodes** are actions (LLM calls, tool executions, human approvals)
- **Edges** are transitions (conditional or fixed)
- **State** is a typed dict that persists across nodes

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

# Define your workflow state
class ResearchWorkflowState(TypedDict):
    research_topic: str
    search_results: list[str]
    draft_report: str
    human_approved: bool
    final_report: str

# Define nodes
def search_node(state: ResearchWorkflowState) -> dict:
    """Search for information on the topic."""
    results = web_search(state["research_topic"])
    return {"search_results": results}

def draft_node(state: ResearchWorkflowState) -> dict:
    """Draft a report from search results."""
    draft = llm.generate_draft(state["search_results"])
    return {"draft_report": draft}

def human_review_node(state: ResearchWorkflowState) -> dict:
    """Pause here — wait for human approval."""
    # LangGraph interrupts execution here
    # Human can inspect state and approve/reject
    return {}  # Human input injected via resume()

def finalize_node(state: ResearchWorkflowState) -> dict:
    return {"final_report": state["draft_report"]}

# Build graph
workflow = StateGraph(ResearchWorkflowState)
workflow.add_node("search", search_node)
workflow.add_node("draft", draft_node)
workflow.add_node("human_review", human_review_node)
workflow.add_node("finalize", finalize_node)

# Define flow
workflow.set_entry_point("search")
workflow.add_edge("search", "draft")
workflow.add_edge("draft", "human_review")
workflow.add_conditional_edges(
    "human_review",
    lambda state: "finalize" if state["human_approved"] else "draft"
)
workflow.add_edge("finalize", END)

# Compile with persistence
from langgraph.checkpoint.postgres import PostgresSaver
checkpointer = PostgresSaver.from_conn_string("postgresql://...")
app = workflow.compile(
    checkpointer=checkpointer,
    interrupt_before=["human_review"]  # HITL gate
)
```

**Security benefit of LangGraph checkpointing:** State is persisted to a database at each step. If a workflow is attacked mid-run, you have a complete audit trail of every state transition.

---

## State Security: What Gets Persisted

With LangGraph's PostgreSQL checkpointer, every state transition is stored:

```sql
-- LangGraph checkpoint schema (simplified)
CREATE TABLE checkpoints (
    thread_id TEXT,        -- Unique workflow instance
    checkpoint_id TEXT,    -- Step number
    checkpoint JSONB,      -- Full state at this step
    metadata JSONB,        -- LLM calls, tool calls, etc.
    created_at TIMESTAMPTZ
);
```

**Security implications:**
1. Sensitive data in workflow state is persisted to the DB — encrypt it
2. Historical state enables forensic analysis after an attack
3. Checkpoints can be replayed — a poisoned checkpoint can re-execute a compromised state
4. The checkpoint DB is a high-value target — protect access carefully

---

## Temporal: Enterprise-Grade Workflow Orchestration

For production agentic systems that need reliability and scalability, **Temporal** is the industry-leading workflow engine. It was originally built for distributed systems but works excellently for AI agent workflows.

```python
from temporalio import activity, workflow
from temporalio.client import Client
from temporalio.worker import Worker

# Define activities (atomic steps)
@activity.defn
async def search_web(topic: str) -> list[str]:
    """Durable web search — retries automatically on failure."""
    return await web_search_client.search(topic)

@activity.defn
async def generate_draft(results: list[str]) -> str:
    """LLM draft generation — retried on API failures."""
    return await llm.generate(results)

# Define the workflow
@workflow.defn
class ResearchWorkflow:
    @workflow.run
    async def run(self, topic: str) -> str:
        # Step 1: Search (retried automatically up to 3 times)
        results = await workflow.execute_activity(
            search_web,
            topic,
            retry_policy=RetryPolicy(maximum_attempts=3),
            start_to_close_timeout=timedelta(minutes=2)
        )

        # Step 2: Draft
        draft = await workflow.execute_activity(
            generate_draft,
            results,
            start_to_close_timeout=timedelta(minutes=5)
        )

        # Step 3: Human signal (HITL built into Temporal's model)
        approval = await workflow.wait_for_signal("human_approval")
        if not approval.approved:
            return "Workflow cancelled by human reviewer"

        return draft
```

**Temporal's security advantages for agents:**
- **Automatic retry** — transient failures don't corrupt state
- **Durable timers** — HITL gates that wait days for approval
- **Full history** — every event logged permanently
- **Namespace isolation** — different teams/tenants isolated
- **Worker isolation** — activity code runs in separate workers

---

## Workflow Security Patterns

### Pattern 1: The Checkpoint Gate
```
WORKFLOW STATE:
  PENDING → SEARCHING → DRAFTED → [HUMAN GATE] → APPROVED → PUBLISHED

The human gate is a cryptographically signed state transition:
  agent_checkpoint = {
      "state": "AWAITING_APPROVAL",
      "draft_hash": sha256(draft_content),
      "approver_required": "senior-analyst"
  }
```

### Pattern 2: Immutable Audit Log
```python
# Every workflow transition writes to append-only audit log
@activity.defn
async def log_transition(event: WorkflowEvent):
    await audit_db.append({
        "workflow_id": event.workflow_id,
        "step": event.step,
        "actor": event.actor,
        "timestamp": event.timestamp,
        "state_hash": hash(event.state),
        "tool_calls": event.tool_calls
    })
    # Note: append-only — no updates, no deletes
```

### Pattern 3: Dead Man's Switch
```python
# Long-running workflows should self-terminate if no human
# activity detected for N hours — prevents abandoned workflows
# continuing to execute
@workflow.defn
class SafeAgentWorkflow:
    HUMAN_ACTIVITY_TIMEOUT = timedelta(hours=24)

    @workflow.run
    async def run(self):
        try:
            human_signal = await workflow.wait_for_signal(
                "heartbeat",
                timeout=self.HUMAN_ACTIVITY_TIMEOUT
            )
        except TimeoutError:
            await self.safe_shutdown("No human activity detected")
```

---

## Stateless vs. Stateful: Which to Use?

```
USE STATELESS (simple chain) when:
  ✅ Task completes in one LLM session
  ✅ No external state changes needed
  ✅ Failure means just retry from scratch
  ✅ Low blast radius if compromised

USE STATEFUL (LangGraph/Temporal) when:
  ✅ Task spans multiple sessions or days
  ✅ Requires human approval checkpoints
  ✅ Multiple agents need coordination
  ✅ Failure recovery is important
  ✅ Audit trail is required

ADD TEMPORAL when you need:
  ✅ Production-grade reliability (not just demos)
  ✅ SLA guarantees on workflow completion
  ✅ Thousands of concurrent workflows
  ✅ Teams/tenants need workflow isolation
```

---

## Further Reading

- [LangGraph Persistence Documentation](https://langchain-ai.github.io/langgraph/concepts/persistence/)
- [Temporal Documentation](https://docs.temporal.io/)
- [Temporal Security Best Practices](https://docs.temporal.io/security)
- [Building Production-Ready AI Agents with LangGraph](https://blog.langchain.dev/)

---

*← [Prev: Vector Databases](./07-vector-databases.md) | [Next: Real-World Deployments →](./09-real-world-deployments.md)*
