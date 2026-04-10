# Secure AI Agent Architecture

> **Audience:** Cloud architects, platform engineers, AI/ML engineers, security engineers  
> **Azure Services:** Azure OpenAI, Azure API Management, Azure Logic Apps, Azure Key Vault, Microsoft Entra ID, Azure Monitor, Azure Container Apps

---

## Introduction

AI agents differ from standard LLM applications in that they can take actions in the real world: calling APIs, reading and writing data, spawning sub-agents, and executing multi-step plans autonomously. This autonomy dramatically expands the blast radius of a security failure.

This document provides a secure architecture for AI agent systems deployed on Azure, covering tool access control, prompt injection defenses, human-in-the-loop patterns, and observability.

---

## What Makes Agent Security Different

| LLM Application | AI Agent |
|---|---|
| Responds to a single query | Executes a multi-step plan |
| Output is text returned to a user | Output can trigger real-world actions |
| Limited blast radius | Potentially unlimited blast radius |
| Single trust boundary | Multiple tool trust boundaries |
| Stateless per request | May maintain memory across sessions |

The key security challenge for agents is **constraining what the agent can do**, not just what it can say.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Azure AI Agent System                          │
│                                                                         │
│  [User / Triggering System]                                             │
│         │                                                               │
│         ▼                                                               │
│  [APIM Gateway]  ──── Auth (Entra ID) ────▶  [Agent Orchestrator]      │
│                                               (Container App / AKS)    │
│                                                      │                  │
│                         ┌────────────────────────────┤                  │
│                         │            │               │                  │
│                         ▼            ▼               ▼                  │
│                   [Tool Router] [Memory Store]  [Azure OpenAI]          │
│                        │         (Azure Cache    (Private Endpoint)     │
│                        │          for Redis)                            │
│          ┌─────────────┼─────────────┐                                  │
│          ▼             ▼             ▼                                  │
│   [Read-only     [Approved API  [Human Review                           │
│    Data Tools]    Calls via      Gate (Logic                            │
│    (Search, DB    APIM]          Apps)]                                 │
│    Reader)                       (for write/                            │
│                                   send/delete)                          │
│                                                                         │
│  ─────────────────── Audit Log (all actions) ─────────────────────────  │
│                         Azure Monitor / Sentinel                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Core Security Principles for AI Agents

### 1. Minimal Footprint

The agent should have access only to the tools, APIs, and data it needs for its defined purpose—nothing more.

- Define the agent's purpose in writing before assigning permissions
- Assign permissions to a **dedicated managed identity** for the agent, not a shared identity
- Review and revoke unused tool registrations regularly

### 2. Explicit Tool Allowlist

Maintain an explicit allowlist of tools the agent is permitted to call. Reject any tool call not on the list, even if the LLM generates a valid-looking function call.

```python
ALLOWED_TOOLS = {"search_knowledge_base", "get_customer_record", "send_notification"}

def execute_tool(tool_name: str, params: dict):
    if tool_name not in ALLOWED_TOOLS:
        raise SecurityError(f"Tool '{tool_name}' is not in the allowed list")
    return tool_registry[tool_name](params)
```

### 3. Human-in-the-Loop for Consequential Actions

Require human approval before any action that:
- Sends communications (emails, messages, alerts)
- Writes, modifies, or deletes data
- Makes financial transactions or API calls with cost implications
- Affects other users or systems

Implement approval workflows using **Azure Logic Apps** or **Azure Durable Functions**.

### 4. Immutable Audit Trail

Every agent action—tool call, reasoning step, decision point—must be logged to an immutable, append-only store. This supports incident investigation and accountability.

### 5. Containment

The agent runtime should be isolated in a dedicated compute environment (Azure Container Apps, AKS namespace) with no direct access to production infrastructure except through controlled API calls.

---

## Tool Security

### Tool Authentication

| Tool Type | Authentication Method |
|---|---|
| Azure services (Storage, SQL, etc.) | Managed identity + Azure RBAC |
| Internal APIs | Entra ID client credentials flow |
| External APIs | Secrets stored in Azure Key Vault; never in code |
| LLM function calls | Server-side validation before execution |

### Tool Input Validation

The orchestrator must validate all tool parameters **after the LLM generates them and before execution**:

- Validate parameter types and ranges (e.g., date ranges, ID formats)
- Sanitize string parameters (no SQL injection, no path traversal)
- Reject parameters that reference out-of-scope resources (e.g., a customer ID the current user does not have access to)

```python
def validate_get_customer_record(params: dict) -> bool:
    customer_id = params.get("customer_id")
    # Validate format
    if not re.match(r'^CUST-\d{6}$', customer_id):
        return False
    # Validate that the current user is authorized for this customer
    if not user_has_access_to_customer(current_user, customer_id):
        return False
    return True
```

### Least-Privilege Tool Permissions

| Tool | Minimum Permission |
|---|---|
| Azure AI Search | `Search Index Data Reader` |
| Azure Blob Storage (read) | `Storage Blob Data Reader` |
| Azure Blob Storage (write) | `Storage Blob Data Contributor` on specific container only |
| Azure SQL Database | `db_datareader` on specific tables only |
| Azure Service Bus (send) | `Azure Service Bus Data Sender` on specific queue only |

---

## Memory Security

Many agent frameworks use a memory system (short-term context window + long-term persistent memory) to maintain state across sessions.

### Short-Term Memory (Context Window)

- Screen all content added to the context window for **indirect prompt injection**
- Enforce maximum context size to prevent context stuffing
- Do not persist the raw context window without sanitization

### Long-Term Memory (Persistent Store)

If the agent maintains persistent memory (e.g., Redis, Cosmos DB, Azure AI Search memory):

- Encrypt the memory store at rest with CMK
- Apply strict access controls (only the agent's managed identity can read/write)
- **Sanitize and validate memory contents before injecting into context** — memory stores are a high-value target for persistent prompt injection
- Implement TTL (time-to-live) on memory entries to limit stale or poisoned data

---

## Prompt Injection Defenses for Agents

Agents are especially vulnerable to prompt injection because they interact with external content (web pages, emails, documents, API responses) that may contain adversarial instructions.

### Defense Layers

| Layer | Defense |
|---|---|
| **Input screening** | Azure AI Content Safety Prompt Shield on all user inputs |
| **External content screening** | Screen all tool outputs (web results, documents, API responses) for embedded instructions before injecting into context |
| **Structural separation** | Use clear delimiters (e.g., `<tool_output>` tags) and instruct the model not to follow instructions in tagged sections |
| **Output action validation** | Before executing any action from an agent step, validate the action is within scope |
| **Anomaly detection** | Alert on unexpected tool call patterns (e.g., a document summarization agent calling a mail send API) |

---

## Human-in-the-Loop Implementation

### Azure Logic Apps Approval Pattern

```
Agent requests action → Logic App triggered → Approval email sent to human reviewer
                          ↓                      ↓
                     Action queued         Human approves or rejects
                          ↓                      ↓
                     If approved: execute   If rejected: agent receives rejection + reason
                     If timeout: auto-reject
```

Configure timeouts: if no human response within a defined SLA (e.g., 1 hour), automatically reject the action and alert on-call staff.

### Risk-Based Approval Tiers

| Risk Level | Examples | Approval Required |
|---|---|---|
| **Low** | Read-only searches, information retrieval | None |
| **Medium** | Draft creation, non-public state changes | Async human review (best effort) |
| **High** | Send email, write to database, make API calls | Synchronous human approval required |
| **Critical** | Delete records, financial transactions, external communications | Two-person approval required |

---

## Observability and Anomaly Detection

### Required Logging

Every agent execution must log:

- Agent ID and session ID
- User identity that triggered the agent
- Each reasoning step (LLM input/output)
- Each tool call: tool name, parameters, result, latency
- Approval requests and outcomes
- Final action taken and result

### Anomaly Detection Rules (Microsoft Sentinel)

| Rule | Trigger | Action |
|---|---|---|
| Tool call outside allowlist | Agent attempts to call unauthorized tool | Alert + block |
| High tool call volume | Agent makes >N tool calls in a session | Alert on-call |
| Cross-tenant data access | Agent accesses resources outside authorized scope | Alert + terminate session |
| Approval bypass attempt | Tool execution without required approval | Alert + block + incident |
| Memory write after external input | Memory updated with content from external source | Alert for review |

---

## Agent Security Checklist

- [ ] Agent's purpose is documented and scope is formally defined
- [ ] Agent has a dedicated managed identity (not shared)
- [ ] Explicit tool allowlist is enforced in the orchestrator
- [ ] Tool parameters are validated before execution
- [ ] Least-privilege RBAC applied to all tool identities
- [ ] Secrets stored in Azure Key Vault (not in code or config)
- [ ] Azure AI Content Safety Prompt Shield applied to user inputs
- [ ] Tool outputs screened for indirect prompt injection before context injection
- [ ] Human-in-the-loop required for all high/critical risk actions
- [ ] Logic Apps approval workflows configured with timeouts
- [ ] All agent actions logged to append-only store
- [ ] Microsoft Sentinel rules configured for anomaly detection
- [ ] Long-term memory encrypted and access-controlled
- [ ] Regular review of agent tool permissions and scope

---

## Next Steps

- [Identity and Access for AI Systems](../04-identity-access.md)
- [Monitoring and Detection](../07-monitoring-detection.md)
- [OWASP LLM Top 10 — Excessive Agency (LLM08)](../02-owasp-llm-top10.md#llm08-excessive-agency)
