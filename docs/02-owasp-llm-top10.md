# OWASP LLM Top 10 Mitigations in Azure

> **Audience:** Security engineers, cloud architects, application developers  
> **References:** [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/), [Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/), [Azure OpenAI Service](https://learn.microsoft.com/en-us/azure/ai-services/openai/)

---

## Introduction

The [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) identifies the most critical security risks for systems built on LLMs. This document maps each risk to specific Azure security controls, architecture patterns, and implementation guidance.

---

## LLM01: Prompt Injection

### Risk

Attackers craft inputs that override the model's instructions, causing it to ignore safety guardrails, reveal confidential system prompts, or take unauthorized actions.

**Direct prompt injection** targets user-controlled inputs. **Indirect prompt injection** embeds malicious instructions in external content (web pages, documents, emails) that the LLM processes.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Azure AI Content Safety** | Enable prompt shield to detect and block injection attempts before they reach the model |
| **Input validation** | Apply allowlists and regex filters in Azure API Management policies or application middleware |
| **System prompt isolation** | Store system prompts in Azure Key Vault; pass them server-side, never from client |
| **Output grounding** | Use Azure AI Foundry's groundedness detection to verify responses are grounded in retrieved context |
| **Structured inputs** | Where possible, use structured API inputs (JSON schema) rather than free-form natural language |
| **Least-privilege tool access** | Restrict LLM-connected tools via Azure RBAC; agents should only call what is necessary |

### Architecture Pattern

Place an AI gateway (Azure API Management + Azure AI Content Safety) between clients and the LLM endpoint. All prompts are screened before reaching Azure OpenAI.

---

## LLM02: Insecure Output Handling

### Risk

Model outputs are passed directly to downstream systems (browsers, SQL engines, shell interpreters) without sanitization, enabling XSS, SQL injection, or remote code execution.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Output encoding** | Encode all LLM outputs before rendering in web UI (HTML entity encoding) |
| **Azure AI Content Safety** | Use the output moderation API to screen model responses before returning to clients |
| **Parameterized queries** | Never construct SQL or API calls using raw LLM output; use parameterized queries |
| **Sandboxed code execution** | If the LLM generates code to be executed, run it in an isolated Azure Container Instance |
| **Schema validation** | Validate LLM-generated structured outputs against a JSON schema before processing |

---

## LLM03: Training Data Poisoning

### Risk

Malicious actors contaminate training or fine-tuning data to introduce backdoors, biases, or vulnerabilities into the model.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Azure Machine Learning data lineage** | Track dataset provenance with AML dataset versioning and MLflow; enforce data integrity checks |
| **Microsoft Purview data governance** | Classify and catalog training datasets; apply sensitivity labels to restrict access |
| **Secure data pipelines** | Use Azure Data Factory with managed identity and private endpoints for data ingestion |
| **Human review checkpoints** | Implement review gates in AML pipelines before data is used for training |
| **Anomaly detection on data** | Use Azure Databricks or AML monitoring to detect statistical anomalies in training datasets |

---

## LLM04: Model Denial of Service

### Risk

Attackers send computationally expensive requests (very long inputs, complex reasoning tasks, recursive loops) to exhaust inference capacity and degrade availability.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Azure API Management rate limiting** | Apply token-per-minute (TPM) and requests-per-minute (RPM) quotas per consumer |
| **Azure OpenAI quota management** | Set deployment-level quota limits in the Azure OpenAI resource |
| **Input length enforcement** | Reject inputs exceeding a maximum token count in API Management or application layer |
| **Azure DDoS Protection** | Enable Azure DDoS Protection Standard on the virtual network hosting AI endpoints |
| **Azure Front Door / WAF** | Use Azure WAF rules to block malformed or anomalously large requests |
| **Autoscaling** | Configure scale-out policies for application layers to absorb burst traffic |

---

## LLM05: Supply Chain Vulnerabilities

### Risk

Compromised third-party model weights, datasets, libraries, or plugins introduce vulnerabilities into the AI system before it even processes a request.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Azure Container Registry** | Store and scan model container images; enable Microsoft Defender for Container Registries |
| **Azure Artifacts / private feeds** | Host Python packages in Azure Artifacts private feeds; avoid pulling directly from public PyPI in production |
| **Dependency scanning** | Integrate GitHub Advanced Security or Dependabot for supply chain vulnerability alerts |
| **Signed artifacts** | Verify digital signatures of model artifacts using Azure Key Vault signing keys |
| **Approved model catalog** | Use Azure AI Foundry model catalog as the approved source for foundation models |
| **Immutable storage** | Store production model checkpoints in Azure Blob Storage with immutability policies |

---

## LLM06: Sensitive Information Disclosure

### Risk

The LLM inadvertently reveals sensitive data including PII, trade secrets, API keys, or memorized training data in its responses.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Azure AI Content Safety** | Enable PII detection and redaction in output moderation policies |
| **Microsoft Purview** | Apply sensitivity labels and DLP policies to data flowing through AI pipelines |
| **System prompt protection** | Never expose system prompts to end users; store them server-side |
| **Output filtering** | Apply regex or NLP-based filters in the application layer to detect and redact PII before returning responses |
| **Data minimization** | Include only necessary context in the prompt; avoid passing raw PII to the model |
| **Audit logging** | Log all prompt/response pairs to Azure Monitor / Log Analytics for post-incident investigation |

---

## LLM07: Insecure Plugin Design

### Risk

LLM plugins and tools (function calling) are granted excessive permissions or lack input validation, allowing the model to perform unintended or destructive operations.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Least-privilege service principals** | Assign minimal Azure RBAC roles to the identity used by AI agents/tools |
| **API authorization** | Protect tool endpoints with Azure Entra ID OAuth 2.0; never use API keys for tool access |
| **Input validation on tools** | Validate all parameters passed by the LLM to a tool before executing the operation |
| **Human-in-the-loop for destructive actions** | Require explicit human approval before tools that write, delete, or send data are executed |
| **Tool allowlist** | Maintain an explicit allowlist of permitted tool operations; reject anything not in the list |
| **Azure API Management for tool APIs** | Route all tool API calls through APIM to enforce rate limits, logging, and policy checks |

---

## LLM08: Excessive Agency

### Risk

AI agents are given too much autonomy, too many capabilities, or insufficient human oversight, leading to unintended high-impact actions.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Scope limitation** | Define the agent's purpose narrowly and restrict tool access to only what is required |
| **Action confirmation** | Implement confirmation steps for any action with real-world consequences (emails, API writes, file deletions) |
| **Azure Logic Apps guardrails** | Orchestrate agent workflows in Logic Apps to enforce approval gates |
| **Azure Monitor alerts** | Alert on unusual agent activity patterns (e.g., high volume of API calls, access to sensitive resources) |
| **Immutable audit trail** | Log all agent actions to an append-only storage (Azure Storage immutable blobs or Azure Monitor) |
| **Kill switch** | Implement an emergency stop mechanism to disable agent action capabilities without code deployment |

---

## LLM09: Overreliance

### Risk

Users and downstream systems treat LLM outputs as authoritative without verification, leading to decisions made on incorrect or hallucinated information.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Groundedness detection** | Use Azure AI Foundry's groundedness evaluation to flag responses not supported by source documents |
| **Confidence scoring** | Surface model confidence or retrieval citation scores alongside responses |
| **Human review workflows** | Route high-stakes decisions through human review using Azure Logic Apps or Power Automate |
| **Retrieval citations** | Configure RAG systems to return source citations with every response so users can verify |
| **Responsible AI disclosures** | Display AI-generated content labels and disclaimers in the UI |
| **Evaluation pipelines** | Implement continuous evaluation of response quality using Azure AI Foundry evaluation flows |

---

## LLM10: Model Theft

### Risk

Attackers reverse-engineer or extract a proprietary model through repeated API queries (model extraction), stealing intellectual property and enabling offline adversarial testing.

### Azure Mitigations

| Control | Implementation |
|---|---|
| **Azure API Management throttling** | Apply strict per-user/per-application rate limits to limit systematic querying |
| **Anomaly detection** | Use Microsoft Sentinel analytics rules to detect high-volume, systematic query patterns |
| **Azure Private Endpoints** | Deploy Azure OpenAI behind a private endpoint to prevent public internet access |
| **Watermarking** | Apply model output watermarking where feasible to trace stolen model artifacts |
| **Authentication enforcement** | Require Entra ID-backed OAuth tokens for all API access; reject API key-only authentication |
| **IP-based restrictions** | Use Azure API Management IP filtering to restrict access to known client ranges |

---

## Summary Control Matrix

| OWASP Risk | Primary Azure Controls |
|---|---|
| LLM01 Prompt Injection | AI Content Safety Prompt Shield, APIM policies, Key Vault |
| LLM02 Insecure Output Handling | AI Content Safety output moderation, output encoding |
| LLM03 Training Data Poisoning | AML dataset versioning, Purview, private data pipelines |
| LLM04 Model DoS | APIM quotas, Azure DDoS Protection, WAF |
| LLM05 Supply Chain | Defender for Containers, Artifacts, code signing |
| LLM06 Sensitive Info Disclosure | Purview DLP, AI Content Safety PII, output filtering |
| LLM07 Insecure Plugin Design | RBAC, Entra ID OAuth, APIM, input validation |
| LLM08 Excessive Agency | Scope limits, Logic Apps approval gates, Monitor alerts |
| LLM09 Overreliance | Groundedness detection, citations, evaluation pipelines |
| LLM10 Model Theft | APIM throttling, Sentinel, Private Endpoints |

---

## Next Steps

- [Secure Azure OpenAI Architecture](03-secure-architectures/secure-azure-openai.md)
- [Identity and Access for AI Systems](04-identity-access.md)
- [Monitoring and Detection](07-monitoring-detection.md)
