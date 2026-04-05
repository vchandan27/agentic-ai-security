# 🗄️ Vector Databases

> **Phase 2 · Article 7 of 9** | ⏱️ 15 min read | 🏷️ `#framework` `#vector-db` `#infrastructure`

---

## TL;DR

- Vector databases store and search high-dimensional embeddings — they're the storage layer for RAG and agent long-term memory.
- The top options (Pinecone, Weaviate, Qdrant, pgvector) have different security architectures, deployment models, and access control capabilities.
- A misconfigured vector DB is a **data breach waiting to happen** — all your agent's knowledge is queryable by anyone with access.

---

## What Makes a Vector Database Different

A traditional database stores structured data and searches with exact matches or range queries. A vector database stores unstructured data as embeddings and searches by *semantic similarity*.

```
Traditional DB:
  SELECT * FROM docs WHERE title = 'password policy'
  → Finds exact matches only

Vector DB:
  similarity_search("how do I reset my password")
  → Finds semantically similar documents:
    - "password reset procedure"
    - "account recovery steps"
    - "forgot password guide"
    - "credential management policy"
```

The similarity search is what makes RAG possible — and what makes vector DBs a unique attack surface.

---

## The Major Vector Databases

### Pinecone (Managed SaaS)

```python
from pinecone import Pinecone

pc = Pinecone(api_key="your-key")
index = pc.Index("knowledge-base")

# Upsert vectors
index.upsert(vectors=[
    {"id": "doc1", "values": [0.1, 0.2, ...], "metadata": {"text": "...", "access": "public"}},
])

# Query with metadata filter
results = index.query(
    vector=[0.1, 0.2, ...],
    top_k=5,
    filter={"access": {"$eq": "public"}}  # Access control via metadata filter
)
```

**Security profile:**
- ✅ Managed service — no infra to secure
- ✅ Metadata filtering for access control
- ✅ API key auth
- ❌ No row-level security (metadata filters are client-enforced, not DB-enforced)
- ❌ All namespaces accessible with a single API key

**Recommendation:** Use separate API keys per application, namespace per tenant. Never expose Pinecone API keys client-side.

---

### Weaviate (Self-hosted or Cloud)

```python
import weaviate
from weaviate.classes.config import Configure

client = weaviate.connect_to_local(
    auth_credentials=weaviate.auth.AuthApiKey("your-key")
)

# Create a collection with RBAC
client.collections.create(
    "Documents",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    properties=[
        weaviate.classes.config.Property(name="content", data_type=weaviate.classes.config.DataType.TEXT),
        weaviate.classes.config.Property(name="access_level", data_type=weaviate.classes.config.DataType.TEXT),
    ]
)

# Query with filter
collection = client.collections.get("Documents")
results = collection.query.near_text(
    query="password reset",
    filters=weaviate.classes.query.Filter.by_property("access_level").equal("public"),
    limit=5
)
```

**Security profile:**
- ✅ Self-hostable (full control)
- ✅ RBAC available (in enterprise version)
- ✅ Multi-tenancy with data isolation
- 🟡 Security features require configuration
- ❌ Default local deployment has no auth

---

### Qdrant (Self-hosted or Cloud)

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue

client = QdrantClient(url="http://localhost:6333", api_key="your-key")

# Search with access control payload filter
results = client.search(
    collection_name="knowledge_base",
    query_vector=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[FieldCondition(
            key="access_level",
            match=MatchValue(value="public")
        )]
    ),
    limit=5
)
```

**Security profile:**
- ✅ Open source (auditable)
- ✅ API key + JWT authentication
- ✅ Payload filtering for access control
- 🟡 No built-in RBAC — implement via payload filters
- ❌ Collections not isolated by default

---

### pgvector (PostgreSQL Extension)

```sql
-- Enable vector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create table with vector column and access control
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536),
    access_level TEXT,
    owner_user_id INTEGER,
    department TEXT
);

-- Index for fast similarity search
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops);

-- Query with access control (real SQL-level security)
SELECT content, 1 - (embedding <=> $1) as similarity
FROM documents
WHERE access_level = 'public'
  AND department = $2          -- User's department filter
ORDER BY embedding <=> $1
LIMIT 5;
```

**Security profile:**
- ✅ Full PostgreSQL security model (row-level security, roles, SSL)
- ✅ Row-level security (RLS) is database-enforced, not application-enforced
- ✅ Existing PostgreSQL expertise applies
- ✅ No new infrastructure to manage if you already use Postgres
- 🟡 Slower than dedicated vector DBs at massive scale

**pgvector with row-level security is the most secure default for sensitive workloads.**

---

## The Access Control Gap

The most common vector DB security mistake is **application-layer access control** instead of **database-layer access control**:

```
APPLICATION-LAYER (WEAK):
  Agent code: results = vectordb.search(query, filter={"user": user_id})
  ↑ If this filter is accidentally omitted → all data accessible
  ↑ If the filter is bypassed → all data accessible
  ↑ No guarantee of enforcement

DATABASE-LAYER (STRONG):
  PostgreSQL RLS: the DB itself enforces the rule
  Even if the application forgets the filter → DB denies access
  Even if application is compromised → DB denies access
```

For sensitive RAG workloads (medical, financial, legal), database-layer enforcement is required.

---

## Multi-Tenancy Patterns

In multi-tenant SaaS platforms, you need strict isolation between tenants:

```
Option 1: Namespace per tenant (Pinecone)
  pinecone_index.describe_index_stats(namespace="tenant_A")
  ✅ Simple  |  🟡 Still shares underlying index

Option 2: Collection per tenant (Qdrant, Weaviate)
  qdrant.get_collection("tenant_A_knowledge")
  ✅ Better isolation  |  🟡 Management overhead at scale

Option 3: Database per tenant (pgvector)
  CREATE DATABASE tenant_a;
  ✅ Strongest isolation  |  ❌ Expensive at large scale

Option 4: Schema per tenant (pgvector)
  SET search_path TO tenant_a;
  ✅ Good isolation with RLS  |  ✅ Scalable
```

---

## Vector DB Security Checklist

```
DEPLOYMENT:
[ ] Enable authentication (API key, JWT, or mTLS)
[ ] Enable TLS for all connections (in transit encryption)
[ ] Enable encryption at rest
[ ] Deploy in private network (no public internet exposure)
[ ] Use separate credentials per application/environment

ACCESS CONTROL:
[ ] Implement access control at DB level (not just application)
[ ] Use metadata filters for document-level permissions
[ ] Implement strict tenant isolation for multi-user systems
[ ] Audit who has API keys and with what permissions

MONITORING:
[ ] Log all queries (query text, user, retrieved documents)
[ ] Alert on: bulk retrieval, unusual query patterns
[ ] Alert on: queries by unauthorized users/services
[ ] Track: which documents are retrieved most often (anomaly detection)

DATA LIFECYCLE:
[ ] Track document provenance (source, upload date, uploader)
[ ] Implement document expiry / TTL for sensitive data
[ ] Audit what's in the DB periodically (stale/sensitive data)
[ ] Delete embeddings when source documents are deleted
```

---

## Comparison Summary

| Feature | Pinecone | Weaviate | Qdrant | pgvector |
|---------|---------|----------|--------|----------|
| Hosting | SaaS only | Both | Both | Self-hosted |
| Auth | API Key | API Key + OIDC | API Key + JWT | PostgreSQL auth |
| Row-level security | ❌ Filters only | 🟡 Enterprise | 🟡 Filters only | ✅ Built-in RLS |
| Multi-tenancy | Namespaces | Multi-tenant class | Collections | Schema/DB per tenant |
| Open source | ❌ | ✅ | ✅ | ✅ |
| Best for | Quick setup, managed | Enterprise, ML workflows | Performance, filtering | Existing PG + max security |

---

## Further Reading

- [Weaviate Security Documentation](https://weaviate.io/developers/weaviate/configuration/authentication)
- [Qdrant Access Control](https://qdrant.tech/documentation/guides/security/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Pinecone Security Overview](https://www.pinecone.io/security/)

---

*← [Prev: RAG Systems](./06-rag-systems.md) | [Next: Agentic Workflows →](./08-agentic-workflows.md)*
