# Secure RAG Architecture

> **Audience:** Cloud architects, platform engineers, AI/ML engineers  
> **Azure Services:** Azure OpenAI, Azure AI Search, Azure Blob Storage, Azure API Management, Microsoft Purview, Azure Key Vault, Microsoft Entra ID

---

## Introduction

Retrieval-Augmented Generation (RAG) enriches LLM responses by grounding them in an organization's proprietary knowledge. It introduces a retrieval pipeline—document ingestion, chunking, embedding, indexing, and search—that significantly expands the attack surface compared to a simple LLM API call.

This document provides a prescriptive secure architecture for enterprise RAG systems built on Azure.

---

## RAG Architecture Overview

A typical enterprise RAG pipeline has two distinct flows:

1. **Ingestion flow:** Documents → chunking → embedding → vector index
2. **Query flow:** User query → embedding → vector search → context assembly → LLM generation → response

Both flows must be secured independently.

---

## Architecture Diagram

```
INGESTION FLOW
══════════════════════════════════════════════════════════════
[Approved       [Azure Data      [Embedding      [Azure AI
 Document        Factory /        Model           Search
 Sources]   ──▶  Event Grid]  ──▶ (Azure      ──▶ Index]
 (Blob Store,    (access ctrl,    OpenAI)]        (RBAC,
  SharePoint)    DLP scan)        (Private        Private
                 [Purview scan]    Endpoint)       Endpoint)

QUERY FLOW
══════════════════════════════════════════════════════════════
[User]  ──▶  [APIM       ──▶  [Orchestrator  ──▶  [Azure AI
             (Auth,            (App Service /       Search]
              Rate limit,       AKS)]           ──▶ [Azure
              Content                               OpenAI]
              Safety)]                         ──▶  [Response
                                                    + Citations]
```

---

## Ingestion Pipeline Security

### Document Source Controls

- **Restrict who can add documents** to the knowledge base. Only approved service principals should have write access to source storage.
- Apply **Azure RBAC** (`Storage Blob Data Contributor`) only to the ingestion pipeline identity; all other users get `Storage Blob Data Reader` at most.
- Enable **Blob Storage versioning and soft delete** to recover from accidental or malicious deletions.

### Data Loss Prevention Before Ingestion

Before documents enter the vector index, scan them for sensitive content:

- Integrate **Microsoft Purview** to classify documents and apply sensitivity labels
- Use **Azure AI Content Safety** or custom classifiers to detect PII, credentials, or confidential markers
- **Block ingestion of documents** classified above a defined sensitivity threshold (e.g., do not index "Confidential" documents into a general-purpose index)
- Log all ingestion operations to Azure Monitor for audit

### Chunking and Embedding Pipeline

The chunking and embedding service (commonly Azure Data Factory, Azure Functions, or a container in AKS) should:

- Run with a **managed identity**; no connection strings in code
- Access Azure OpenAI embedding endpoint via **private endpoint**
- Store intermediate chunks in **encrypted temporary storage** (not in plaintext logs)
- Apply **idempotency controls** — duplicate documents should not create duplicate chunks that pollute retrieval results

### Vector Index Security (Azure AI Search)

| Control | Implementation |
|---|---|
| **Network isolation** | Deploy Azure AI Search with a private endpoint; disable public network access |
| **Authentication** | Disable API key authentication; require Azure Entra ID RBAC for all operations |
| **Index-level RBAC** | Use Azure AI Search semantic ranker and security trimming to enforce per-document access control |
| **Encryption** | Enable customer-managed key (CMK) encryption for index data |
| **Audit logging** | Enable diagnostic logs to Log Analytics for all search queries |

### Security Trimming (Multi-Tenant / Multi-User)

For systems where different users should access different subsets of documents:

1. At ingestion time, attach security metadata to each document chunk (e.g., `allowed_groups: ["finance-team"]`)
2. At query time, include the user's group memberships in the search filter
3. Use **Azure AI Search security filters** to enforce this at the index query level
4. **Never rely solely on the LLM** to enforce access control — enforce it at the retrieval layer

---

## Query Pipeline Security

### Authentication and Authorization

- All client requests must authenticate via **Entra ID OAuth 2.0** tokens
- The orchestration layer validates tokens before processing any query
- The orchestrator's identity (managed identity) is authorized to query the vector index with `Search Index Data Reader` role

### Input Validation and Content Safety

Before assembling the prompt:

1. **Validate query length** — reject queries exceeding a configured token limit
2. **Screen for prompt injection** via Azure AI Content Safety Prompt Shield
3. **Sanitize query text** — strip HTML, control characters, and injection patterns

### Context Assembly Security

The context assembly step combines retrieved chunks with the user's query to form the final prompt. Key controls:

- **Citation tracking** — retain references to source documents for each chunk included in context
- **Context window limits** — enforce maximum context size to prevent context stuffing attacks
- **Chunk attribution validation** — verify chunks came from the authorized index, not from an external source
- **Indirect prompt injection detection** — screen retrieved chunks for embedded instructions before including in context

```python
# Pseudocode: screen retrieved chunks for prompt injection before assembly
for chunk in retrieved_chunks:
    safety_result = content_safety_client.analyze_text(chunk.content)
    if safety_result.has_prompt_injection:
        log.warning(f"Prompt injection detected in chunk from {chunk.source}")
        retrieved_chunks.remove(chunk)
```

### LLM Call Security

- Route all Azure OpenAI calls through a **private endpoint**
- Authenticate using the orchestrator's **managed identity** (`Cognitive Services OpenAI User` role)
- Apply **Azure API Management** as a gateway for rate limiting and logging

### Output Processing

Before returning the response to the user:

1. Run the response through **Azure AI Content Safety** output moderation
2. Check **groundedness** — flag responses that cannot be attributed to retrieved context
3. Return **source citations** alongside the response to support user verification
4. **Strip PII** from the response if detected

---

## Data Isolation for Multi-Tenant RAG

| Scenario | Isolation Approach |
|---|---|
| **Hard tenant isolation** | Separate Azure AI Search resources per tenant; separate storage accounts |
| **Soft tenant isolation** | Single search resource with per-tenant indexes and RBAC; security filters on queries |
| **User-level document access** | Security trimming with Entra ID group membership pushed as search filter |

**Recommendation:** Use hard isolation for tenants with different regulatory requirements (e.g., a tenant with HIPAA data vs. general commercial). Use soft isolation for large multi-tenant SaaS platforms where operational overhead of per-tenant resources is prohibitive.

---

## Compliance and Data Governance

| Requirement | Control |
|---|---|
| **Data residency** | Deploy all resources in the same Azure region; disable geo-redundancy for regulated data |
| **Data retention** | Apply lifecycle policies to blob storage; implement index purge APIs for right-to-erasure requests |
| **Audit trail** | Log all ingestion and query operations to Log Analytics; retain for minimum 90 days |
| **Sensitivity enforcement** | Use Purview sensitivity labels to prevent over-sharing of classified documents |

---

## RAG Security Checklist

### Ingestion Pipeline
- [ ] Only approved identities have write access to document source storage
- [ ] All documents are scanned by Purview / Content Safety before ingestion
- [ ] Documents above the sensitivity threshold are excluded from the index
- [ ] Ingestion pipeline runs with managed identity (no connection strings)
- [ ] Embedding endpoint accessed via private endpoint
- [ ] All ingestion operations logged to Log Analytics

### Vector Index
- [ ] Public network access disabled on Azure AI Search
- [ ] Entra ID RBAC enforced; API key authentication disabled
- [ ] Security trimming metadata applied to all document chunks
- [ ] CMK encryption enabled
- [ ] Diagnostic logs enabled

### Query Pipeline
- [ ] Client authentication via Entra ID OAuth 2.0
- [ ] Input length limits enforced
- [ ] Azure AI Content Safety Prompt Shield applied to user queries
- [ ] Retrieved chunks screened for indirect prompt injection
- [ ] Context window size limits enforced
- [ ] Output moderation applied before returning responses
- [ ] Groundedness check applied
- [ ] Source citations returned with every response

---

## Next Steps

- [Secure AI Agent Architecture](secure-ai-agent-architecture.md)
- [AI Data Governance and Protection](../05-data-governance.md)
- [Monitoring and Detection](../07-monitoring-detection.md)
